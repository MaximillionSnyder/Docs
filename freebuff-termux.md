# Freebuff en Termux nativo

Guía de lo que se hizo para instalar `freebuff` en Termux (Android/aarch64): paquete apt del repo de Ivam3 + truco `LD_PRELOAD` para usar `apt-get` bajo el fake-root de proot.

Fecha: 21 sep 2026 — freebuff versión instalada: **0.0.180** (paquete apt `0.0.174-1`)

---

## 1. Qué es freebuff

- `freebuff` es el agente de código gratis para la terminal (tier gratis de Codebuff): `npm install -g freebuff`, luego `freebuff` dentro del proyecto.
- En Termux **no sirve el npm directo**: igual que opencode, el instalador no contempla `process.platform === 'android'` (`EBADPLATFORM`, quiere `darwin,linux,win32`).
- Existe paquete apt en el repo de Ivam3 (`https://ivam3.github.io/termux-packages`, `stable/extras`):
  - `apt-cache show freebuff` → `Depends: glibc, clang, python, jq, curl, tar`, `Suggests: i-haklab`
  - El `postinst` descarga el binario `linux-arm64` (~129 MB, compilado con Bun/glibc), lo deja en `~/.local/share/freebuff/freebuff.real` y compila un bootstrapper C nativo (bionic) en `$PREFIX/bin/freebuff`.

## 2. El problema: apt se niega a correr como root y vivimos en fake-root

- Desde que opencode corre con **proot como shim** (ver `arreglandoopencode.md`), todo lo que ejecuta ve `uid=0` (fake-root por `proot -0`). El uid real es `10428` (`u0_a428`), visible en `/proc/self/status`; el `TracerPid` es el proceso proot.
- Los binarios `apt`/`apt-get` de Termux chequean `getuid() == 0` y abortan:
  ```
  Ability to run this command as root has been disabled permanently for safety purposes.
  ```
- Afecta a `apt` y `apt-get` (ni siquiera dejan `--version`). En cambio `apt-cache` y `dpkg` **sí** funcionan bajo fake-root, por eso `apt-cache policy freebuff` mostraba el candidato sin problema.

## 3. La solución: `LD_PRELOAD` que devuelve el uid real

La idea: sobrescribir `getuid`/`geteuid`/`getgid`/`getegid`/`getresuid`/`getresgid` para que devuelvan el uid/gid real (10428). Funciona porque proot falsifica el uid a nivel de syscall (ptrace); al devolver la constante sin hacer syscall, no hay nada que interceptar y apt cree que no somos root.

Detalle importante: el linker de Android rechaza `LD_PRELOAD` desde `$TMP` (`/usr/tmp`):

```
CANNOT LINK EXECUTABLE "apt-get": library ".../fakeuid.so" ... is not accessible for the namespace "(default)"
```

El `.so` **tiene que vivir en `$PREFIX/lib`**.

### 3.1 Código (`fakeuid.c`)

```c
#include <unistd.h>
#include <sys/types.h>
uid_t getuid(void) { return 10428; }
uid_t geteuid(void) { return 10428; }
gid_t getgid(void) { return 10428; }
gid_t getegid(void) { return 10428; }
int getresuid(uid_t *r, uid_t *e, uid_t *s) {
  if (r) *r = 10428;
  if (e) *e = 10428;
  if (s) *s = 10428;
  return 0;
}
int getresgid(gid_t *r, gid_t *e, gid_t *s) {
  if (r) *r = 10428;
  if (e) *e = 10428;
  if (s) *s = 10428;
  return 0;
}
```

> Si el uid real cambia en otro dispositivo, sacarlo de `cat /proc/self/status | grep Uid` (la primera columna es el uid real aunque `id -u` diga 0).

### 3.2 Compilar y usar

```bash
clang -shared -fPIC -o "$PREFIX/lib/fakeuid.so" fakeuid.c
chmod 755 "$PREFIX/lib/fakeuid.so"

export LD_PRELOAD="$PREFIX/lib/fakeuid.so"
apt-get update
apt-get install -y freebuff
```

Comprobación del bypass:

```bash
LD_PRELOAD=$PREFIX/lib/fakeuid.so id          # → uid=10428(u0_a428)
LD_PRELOAD=$PREFIX/lib/fakeuid.so apt-get --version  # → apt 2.8.1 (sin error de root)
```

### 3.3 Lo que hizo el postinst

```
Preparing to unpack .../freebuff_0.0.174-1_aarch64.deb ...
Downloading freebuff 0.0.180 (glibc)...
Extracting binary...
 [✔] Binary installed to /data/data/com.termux/files/home/.local/share/freebuff/freebuff.real
Compiling native bootstrapper...
 [✔] Bootstrapper installed: /data/data/com.termux/files/usr/bin/freebuff
 [✔] freebuff installation finished.
 [📢] Run 'freebuff' to execute it.
```

Verificación:

```bash
freebuff --version   # → 0.0.180
```

## 4. Qué hacer para nuestros empaquetados

Reglas a seguir cuando empaquetemos nuestras propias herramientas como `.deb` para Termux (mismo patrón que este paquete `freebuff`):

1. **Declarar `Depends` reales** en el control del paquete (`glibc, clang, python, jq, curl, tar`, lo que corresponda) para que apt resuelva la base. Usar `Suggests` para lo opcional (ej. `i-haklab`).
2. **Binario glibc (Bun/Node empaquetado) → bootstrapper C bionic, NO patchelf al binario.** El `patchelf --set-interpreter/--set-rpath` sobre binarios de Bun los corrompe (segfault). El lanzador debe hacer `unsetenv("LD_PRELOAD")`, `unsetenv("LD_LIBRARY_PATH")` y `execv` al loader glibc (`$PREFIX/glibc/lib/ld-linux-aarch64.so.1 --library-path $PREFIX/glibc/lib <binario-real>`). Ver `opencode_helper_direct*.c` como referencia.
3. **`postinst` no interactivo**: descarga por arquitectura, extrae a `~/.local/share/<paquete>/`, compila el bootstrapper a `$PREFIX/bin/<paquete>`, imprime versión instalada y ayuda/contacto, limpia temporales.
4. **Probar bajo fake-root (`proot -0`)**: `apt`/`apt-get` se niegan como "root" aunque el uid real no sea 0. Para instalar/probar ahí, usar el truco `LD_PRELOAD` del §3 o `dpkg -i` manual. `apt-cache` sirve para consultar sin truco.
5. **Dejar constancia con fecha y versión** (`paquete apt X + binario Y`, salida de `--version`) en un doc como este.

## 5. Comandos útiles

```bash
# actualizar
export LD_PRELOAD="$PREFIX/lib/fakeuid.so"
apt-get update && apt-get install -y freebuff   # o: apt-get install --only-upgrade freebuff

# verificar
freebuff --version
which freebuff   # → $PREFIX/bin/freebuff (bootstrapper, ~7 KB)

# desinstalar
export LD_PRELOAD="$PREFIX/lib/fakeuid.so"
apt-get remove freebuff
```

- `fakeuid.so` queda en `$PREFIX/lib/fakeuid.so` reutilizable para cualquier instalación apt bajo fake-root (solo afecta a los comandos lanzados con ese `LD_PRELOAD`, nada global).
