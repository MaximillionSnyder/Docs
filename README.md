# Docs

Notas técnicas y diagnósticos. Un tema por archivo, autocontenido: cada uno incluye el
entorno, los comandos usados y las salidas relevantes, para poder reconstruir lo que
pasó sin depender de la memoria.

| Documento | Tema |
| --- | --- |
| [afinidad.md](afinidad.md) | Afinidad (相性) en Uma Musume — nota técnica |
| [dsh-read-image-android.md](dsh-read-image-android.md) | `read_image` roto en DSH sobre Android/Termux — diagnóstico y arreglo |
| [freebuff-termux.md](freebuff-termux.md) | Freebuff en Termux nativo |
| [playwright-termux.md](playwright-termux.md) | Playwright en Termux (Android) — cómo está configurado |

## Convenciones

- **Nombre:** kebab-case, descriptivo, en español (`tema-entorno.md`).
- **Estructura:** síntoma → diagnóstico → causa raíz → arreglo → verificación →
  mantenimiento.
- **Evidencia:** pegar las salidas reales de los comandos, no parafrasearlas. Si algo
  se descartó, decir **por qué**.
- **Portabilidad:** anotar lo que es específico de Android/Termux, porque suele ser la
  causa de los fallos.

## Entorno habitual

Termux en Android aarch64, sin acceso a los servicios del sistema (dominio SELinux
`untrusted_app`), con `root` sin capacidades (`CapEff = 0`).
