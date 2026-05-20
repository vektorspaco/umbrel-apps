# vektorspaco's Umbrel Apps

Community app store con apps caseras para el stack de Espacio Vectorial.

## Como agregar este store a tu Umbrel

1. Abri Umbrel → `App Store`
2. Click en los tres puntitos arriba a la derecha → `Community App Stores`
3. Click `Add` y pega la URL de este repo:
   ```
   https://github.com/vektorspaco/umbrel-apps/
   ```
4. Las apps aparecen como cualquier otra en el store, con un tag de "community".

## Apps disponibles

| App | Descripcion | Doc |
|---|---|---|
| **Multicam Player** | Reproductor HLS multicam (dos streams de video sincronizados + multiples pistas de audio) servido por nginx, listo para exposicion via Tailscale Funnel. | [vektorspaco-multicam/README.md](vektorspaco-multicam/README.md) |

## Estructura del repo

```
.
├── README.md                    Este archivo
├── umbrel-app-store.yml         Manifest del store (id, name)
└── vektorspaco-multicam/        App: Multicam Player (ver README propio)
```
