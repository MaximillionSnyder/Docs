# `read_image` roto en DSH sobre Android/Termux — diagnóstico y arreglo

**Fecha:** 2026-09-23
**Entorno:** Termux en Android aarch64 · DSH `0.1.5-rc.2` · Node del sistema
**Paquete afectado:** `@deepseek-ai/dsh-attachment-local`
**Síntoma en una línea:** `read_image` fallaba con `EACCES` para *cualquier* ruta.

---

## 1. El síntoma

La herramienta `read_image` devolvía siempre el mismo error, sin importar qué archivo
se le pasara:

```
EACCES: permission denied, open '/data/data'
```

Tres detalles que orientaron el diagnóstico desde el principio:

- La ruta del error (`/data/data`) **no era la ruta pedida**. Se probó con un archivo
  dentro del workspace y con otro en `/storage/emulated/0/Download`, y el error fue
  idéntico. Eso descartó que el problema estuviera en el archivo de destino.
- `read` (texto) **sí funcionaba** sobre el mismo archivo y el mismo directorio.
- La subida de imágenes desde la GUI también estaba rota, por la misma causa.

---

## 2. Diagnóstico

### 2.1 Descartar que fuera un problema de usuario

El servidor DSH corre como `u0_a428`, mientras que el shell del agente corre como
`root`:

```
u0_a428  17065  1  node --expose-internals .../dsh/lib/bin.js web --no-open
uid=0(root) ... context=u:r:untrusted_app_27:s0
```

Se probó el acceso a los mismos archivos desde **ambos** usuarios con un script de
Node que baja privilegios (`process.setgid` / `process.setuid`). Resultado: los dos se
comportan igual. **No era un problema de usuario.**

### 2.2 Descartar permisos Unix (DAC)

```
CapPrm: 0000000000000000
CapEff: 0000000000000000     <- root SIN capacidades
CapBnd: 0000000000000000
```

Con `CapEff = 0`, `root` no puede saltarse las comprobaciones DAC. Pero el dato
decisivo es que **`/` también falla**, y `/` es `drwxr-xr-x`: cualquiera debería poder
abrirlo. Si DAC lo permite y aun así da `EACCES`, la negativa viene de **SELinux**
(dominio `untrusted_app_27`).

### 2.3 Identificar la llamada exacta por la firma del error

Node usa un verbo distinto en el mensaje según la API. Se comprobó cada una:

| API | Mensaje |
| --- | --- |
| `fs.readdir` | `EACCES: permission denied, scandir '<path>'` |
| `fs.opendir` | `EACCES: permission denied, opendir '<path>'` |
| `fs.open` / `fs.readFile` / `fs.createReadStream` | `EACCES: permission denied, open '<path>'` |

El error observado decía `open`. Por tanto la llamada que falla es `open()`,
`readFile()` o `createReadStream()` **sobre el path literal `/data/data`**.

### 2.4 Rastrear la cadena en el código

`read_image` vive en `@deepseek-ai/dsh-tool-fs`. Su pipeline es:

```
read_image
  -> assertImageCapableRoute
  -> resolveRegularReadTarget        (ctx.fs.resolve + ctx.fs.stat)
  -> ctx.fs.readBytes
  -> attachments.saveImage           <- aquí está el problema
```

Y dentro de `@deepseek-ai/dsh-attachment-local`, la cadena de guardado es:

```
saveImage
  -> commitPreparedImageFile
  -> publishImmutableObject
  -> stageImmutableObject            (línea 490)
       -> ensureDurableHome(dirname(dirname(resolve(root))))   (línea 492)
            -> ensureDurableDirectory(home, parse(home).root)  (boundary = "/")
                 -> bucle que hace syncDirectory() de CADA ancestro
                      -> syncDirectory("/data/data")  ->  EACCES
```

### 2.5 Reproducir la caminata exacta

Se replicó el bucle en Node para confirmar el punto de ruptura:

```
--- replica de ensureDurableHome(~/.dsh) ---
  OK    open /data/data/com.termux/files/home
  OK    open /data/data/com.termux/files
  OK    open /data/data/com.termux
  FALLA open /data/data  ->  EACCES: permission denied, open '/data/data'
  FALLA open /data
  FALLA open /
```

**Coincidencia exacta con el error de `read_image`.** Diagnóstico cerrado.

---

## 3. Causa raíz

En `@deepseek-ai/dsh-attachment-local/lib/index.js`:

```js
async function syncDirectory(path) {
	if (process.platform === "win32") return;
	const handle = await open(path, constants.O_RDONLY);   // <-- aquí
	try {
		await handle.sync();
	} finally {
		await handle.close();
	}
}

async function ensureDurableHome(path) {
	const home = resolve(path);
	if (!durableHomes.has(home)) {
		await ensureDurableDirectory(home, parse(home).root);   // boundary = "/"
		durableHomes.add(home);
	}
	return home;
}

async function ensureDurableDirectory(path, boundary) {
	const target = resolve(path);
	const stop = resolve(boundary);
	await mkdir(target, { recursive: true, mode: 448 });
	await chmod(target, 448);
	let level = target;
	while (level !== stop) {
		const parent = dirname(level);
		await syncDirectory(parent);       // fsync del ancestro
		if (parent === level) return;
		level = parent;
	}
}
```

`ensureDurableHome` usa como *boundary* la **raíz del sistema de archivos** (`/`), así
que la caminata abre **todos** los ancestros hasta `/`. En Linux de escritorio eso
funciona siempre. En Android, SELinux no deja abrir `/data/data`, `/data` ni `/`, y la
excepción se propaga, abortando el guardado del adjunto.

Como `durableHomes` es un `Set` **en memoria** y nunca llega a poblarse (siempre falla
antes), el error se repite en cada llamada. No es un fallo intermitente: es
determinista.

---

## 4. Por qué NO había arreglo por configuración

Se probó si mover `DSH_HOME` a otro sitio evitaba el problema. No sirve: la caminata
siempre termina en `/`, y `/` no se puede abrir desde este dominio. Se verificó qué
ancestros son abribles:

```
FALLA /data
FALLA /data/data
FALLA /data/local
FALLA /data/local/tmp
FALLA /storage
FALLA /storage/emulated      (ENOENT)
OK    /storage/emulated/0
OK    /data/data/com.termux/files/home
```

**Ninguna ubicación de `DSH_HOME` se salva**, porque el bucle no se detiene antes de `/`.

---

## 5. Por qué actualizar el paquete NO servía

Se descargó la última versión publicada y se comparó el código:

```bash
npm pack @deepseek-ai/dsh-attachment-local@0.1.7-rc.1
grep -n "async function syncDirectory" -A 12 package/lib/index.js
```

`0.1.7-rc.1` tiene **el código idéntico**. No está corregido upstream. Instalada:
`0.1.5-rc.2`.

---

## 6. El arreglo

Un solo punto de cambio. El `fsync` de ancestros es una garantía de **durabilidad ante
crash**, no un requisito para guardar el archivo — de hecho la función ya se salta el
paso entero en Windows. Así que se toleran los errores de permiso y se propaga
cualquier otro:

```js
async function syncDirectory(path) {
	if (process.platform === "win32") return;
	/* Android/Termux: SELinux deniega abrir /data/data, /data y /. El fsync de
	   ancestros es best-effort (durabilidad ante crash); no debe ser fatal. */
	let handle;
	try {
		handle = await open(path, constants.O_RDONLY);
	} catch (error) {
		if (error.code === "EACCES" || error.code === "EPERM" || error.code === "ENOENT") return;
		throw error;
	}
	try {
		await handle.sync();
	} finally {
		await handle.close();
	}
}
```

**Consecuencia asumida:** en Android no se puede garantizar la durabilidad de los
ancestros inabribles ante un crash. Es un intercambio aceptable: la alternativa es no
poder leer ni subir imágenes en absoluto.

---

## 7. Verificación

1. **Sintaxis:** `node --check` sobre el archivo parcheado → OK.
2. **Lógica:** se replicó `ensureDurableDirectory` ya parcheada y la caminata completa
   sin lanzar error:
   ```
   ensureDurableHome(~/.dsh)        ->  COMPLETA sin lanzar error
   ensureDurableDirectory(staging)  ->  COMPLETA
   ```
3. **Extremo a extremo:** tras reiniciar DSH, `read_image` devolvió la imagen
   correctamente (PNG 756x2410, 59872 bytes).

---

## 8. Operación y mantenimiento

El arreglo se aplica con un script idempotente, con backup y reversión:

```bash
~/dsh-fix-syncdir.sh apply     # aplica (crea index.js.orig la primera vez)
~/dsh-fix-syncdir.sh status    # PARCHEADO / ORIGINAL
~/dsh-fix-syncdir.sh revert    # restaura el original
```

Tras aplicar hay que **reiniciar DSH** (el módulo ya está cargado en memoria):

```bash
~/dsh-restart.sh               # desatendido, con retraso
# o manualmente:
pkill -f 'bin\.js web' && ~/dsh-up.sh
```

> **Importante:** el parche vive en `node_modules`, así que **reinstalar o actualizar
> DSH lo borra**. Después de cada actualización, ejecutar
> `~/dsh-fix-syncdir.sh apply` y reiniciar.

---

## 9. Notas de fondo

- En Android, un proceso en el dominio SELinux `untrusted_app` **no puede abrir**
  `/data/data`, `/data` ni `/`, aunque sea `root`. `CapEff = 0` impide además saltarse
  las comprobaciones DAC.
- Sí puede, en cambio, leer archivos concretos dentro de
  `/data/data/com.termux/files/home`. Lo que falla es abrir los **directorios**
  intermedios.
- Código pensado para Linux de escritorio que asume que todos los ancestros hasta `/`
  son abribles **se rompe en Android**. Merece la pena revisar cualquier caminata de
  este tipo al portar herramientas.
- Para identificar qué API de Node lanza un error de filesystem, la **firma del
  mensaje** (`open` / `opendir` / `scandir`) acota muchísimo la búsqueda.

---

## 10. Archivos implicados

| Ruta | Qué es |
| --- | --- |
| `.../dsh-attachment-local/lib/index.js` | El paquete parcheado |
| `.../dsh-attachment-local/lib/index.js.orig` | Backup del original |
| `~/dsh-fix-syncdir.sh` | Aplicar / revertir / consultar estado |
| `~/dsh-restart.sh` | Reinicio desatendido de DSH |
| `~/dsh-restart.log` | Resultado del reinicio |
| `~/dsh-url.txt` | URL con el token vigente |
