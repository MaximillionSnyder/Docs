# Playwright en Termux (Android) — cómo está configurado

Tests end-to-end de la app con **Playwright 1.63** sobre Chromium en Termux,
reutilizando la instalación global y los navegadores ya cacheados.

## Requisitos previos (global)

- `playwright` 1.63.0 global: `npm i -g playwright`
- Navegadores en `~/.cache/ms-playwright/`:
  - `chromium_headless_shell-1243/chrome-headless-shell-linux-arm64/chrome-headless-shell` (headless por defecto)
  - `chromium-1243/chrome-linux-arm64/chrome` (Chromium completo)
- Wrapper `~/.local/bin/playwright` → `~/.playwright-termux/bin/playwright`, que exporta:
  - `NODE_OPTIONS=--require ~/.playwright-termux/android-compat.js`
  - `PLAYWRIGHT_SKIP_VALIDATE_HOST_REQUIREMENTS=1`

## Los tres problemas en Termux

1. **`process.platform === 'android'`**: `playwright-core` lanza
   `Unsupported platform: android` al calcular su directorio de caché. Se
   soluciona reportando `linux` (Android usa kernel Linux) para que elija el
   layout `linux-arm64`.
2. **No existe `ldd`**: la validación de dependencias del host falla. Se
   desactiva con `PLAYWRIGHT_SKIP_VALIDATE_HOST_REQUIREMENTS=1`.
3. **Shebangs de `node_modules/.bin/`**: apuntan a `#!/usr/bin/env`, que no
   existe en Termux (`/usr/bin/env: bad interpreter`). Por eso el CLI se
   invoca directamente como `node node_modules/@playwright/test/cli.js`.

## Setup aplicado en este proyecto

### 1. Dependencia local

Se instala `@playwright/test` sin volver a descargar navegadores
(`PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1`) y fijando la versión que coincide con
los navegadores cacheados (1.63.0 ↔ chromium-1243):

```bash
PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 npm install -D @playwright/test@1.63.0 --save-exact
```

Si más adelante se actualiza Playwright, hay que ejecutar también
`playwright install chromium` para tener los navegadores de esa versión.

### 2. Shim de Termux

`scripts/termux-playwright-shim.cjs` corrige la plataforma, desactiva la
validación del host y se propaga a los procesos hijos (workers del test
runner) vía `NODE_OPTIONS`:

```js
'use strict';

if (process.platform === 'android') {
  Object.defineProperty(process, 'platform', {
    value: 'linux',
    configurable: true,
    writable: true,
  });
}

process.env.PLAYWRIGHT_SKIP_VALIDATE_HOST_REQUIREMENTS = '1';

const nodeOptions = process.env.NODE_OPTIONS ?? '';
if (!nodeOptions.includes(__filename)) {
  process.env.NODE_OPTIONS = `${nodeOptions} --require ${__filename}`.trim();
}
```

### 3. Script npm

En `package.json`:

```json
"e2e": "node --require ./scripts/termux-playwright-shim.cjs node_modules/@playwright/test/cli.js test"
```

No se usa `playwright` a secas porque el binario de `.bin/` no es ejecutable
en Termux (punto 3 de arriba).

### 4. `playwright.config.ts`

- `testDir: './e2e'`, 1 worker, sin paralelismo.
- `webServer`: levanta `npm run dev` en `127.0.0.1:5173` (`--strictPort`) y lo
  apaga al terminar; `reuseExistingServer: true` reutiliza uno ya activo.
- `env: { NODE_OPTIONS: '' }` en el `webServer`: Vite usa rolldown en modo
  WASI y no debe heredar el shim (el override de plataforma puede confundir
  al binding WASI). Solo los procesos de Playwright necesitan el shim.
- En fallo guarda screenshot y trace en `test-results/`.

### 5. Vitest vs Playwright

`vite.config.ts` excluye `e2e/**` en `test.exclude` para que Vitest no intente
cargar los specs de Playwright (importan `@playwright/test`, que falla bajo
Vitest).

### 6. Selectores estables

Se añadieron `data-testid` en la app para no depender de clases CSS:

- `slot-<posicion>` en `CharacterCard` (p. ej. `slot-objetivo`, `slot-padre`).
- `afinidad-rango` y `afinidad-total` en la barra de puntuación de `PedigreeTree`.

## Ejecutar

```bash
npm run e2e                      # suite completa (primer plano)
npm run e2e -- --grep "matriz"   # filtrar tests
nohup npm run e2e > e2e.log 2>&1 &   # en background
```

- Los specs viven en `e2e/` (`e2e/pedigree.spec.ts`).
- En caso de fallo: `test-results/` con screenshot y trace
  (`npm run e2e -- --trace on` para forzar traces siempre).

## Notas

- `npm run lint` (oxlint) crashea con `Illegal instruction` en este Termux,
  incluso con `--version`; es un problema del binario nativo, ajeno a
  Playwright.
- Los artefactos de test (`test-results/`, `playwright-report/`,
  `blob-report/`, `playwright/.cache`) están en `.gitignore`.
