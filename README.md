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

---

## Multicam Player

Reproductor HLS multicam (dos streams de video sincronizados + multiples
pistas de audio) servido desde el Umbrel sobre nginx. La app expone el
contenido en LAN; para que viewers externos accedan, se usa **Tailscale
Funnel** (recomendado), Tailscale Serve, o cualquier otro tunneling que
prefieras delante.

### Que incluye la app

- **nginx** sirviendo los assets HLS desde `~/umbrel/app-data/vektorspaco-multicam/data/content/`.
- CORS abierto para `.m3u8` y `.ts`, cache largo sobre los segmentos
  inmutables, cache corto sobre los manifests (ver `data/nginx.conf`).
- Slot para `.htpasswd` ya mapeado en el container (si despues queres
  sumar auth basica, no hace falta tocar el compose).

### Dos puertos: para que sirve cada uno

La app expone nginx por **dos caminos** distintos:

- **`http://umbrel.local:18080`** → pasa por el `app_proxy` de Umbrel.
  Requiere estar logueado al dashboard de Umbrel para entrar. Es el
  puerto declarado en el manifest (`port: 18080` en `umbrel-app.yml`)
  y el que abre la app cuando hacés click en "Open" desde el dashboard.
  **Uso: vos, en tu LAN, con tu cuenta de Umbrel.**

- **`http://umbrel.local:8081`** → bindea nginx directo, **sin pasar
  por el app_proxy**, sin auth de Umbrel. Es el bypass que se usa para
  exposicion externa (Tailscale Funnel) donde los viewers no tienen
  cuenta en tu Umbrel. **Uso: amigos, via Tailscale Funnel.**

### Setup paso a paso

#### 1. Instalar la app

1. App Store → `Multicam Player` → `Install`.
2. La app arranca y queda accesible:
   - En `http://umbrel.local:18080` desde el dashboard de Umbrel.
   - En `http://umbrel.local:8081` directo (bypass para Funnel).

#### 2. Subir el contenido

Por SFTP/SCP, los archivos van a:

```
~/umbrel/app-data/vektorspaco-multicam/data/content/
├── index.html
├── 1T/
│   ├── bbc/
│   │   ├── index.m3u8
│   │   └── seg_*.ts
│   ├── tactic/
│   │   ├── index.m3u8
│   │   └── seg_*.ts
│   └── bbc_audio/
│       ├── index.m3u8
│       └── seg_*.ts
└── 2T/
    └── ...
```

Ejemplo con rsync desde tu maquina:

```bash
rsync -avh --progress ./multicam-assets/ \
  umbrel@umbrel.local:~/umbrel/app-data/vektorspaco-multicam/data/content/
```

No hace falta reiniciar la app: nginx ve los archivos nuevos al toque.

#### 3. (Opcional) Auth basica con .htpasswd

El compose ya monta `${APP_DATA_DIR}/data/.htpasswd:/etc/nginx/.htpasswd:ro`,
asi que solo tenes que generar el archivo y reactivar `auth_basic` en
`nginx.conf`. Pasos:

1. En tu maquina o en el Umbrel:
   ```bash
   htpasswd -c ~/umbrel/app-data/vektorspaco-multicam/data/.htpasswd <usuario>
   ```
   (te pide la contraseña interactivamente).
2. Editar `~/umbrel/app-data/vektorspaco-multicam/data/nginx.conf` y agregar
   dentro del `server { ... }`:
   ```nginx
   auth_basic           "Restricted";
   auth_basic_user_file /etc/nginx/.htpasswd;
   ```
3. Reiniciar la app desde la UI de Umbrel.

Si no necesitas auth (porque ya esta detras de Tailscale o solo lo usas en
LAN), saltea este paso.

#### 4. Exposicion externa con Tailscale Funnel

Si ya tenes Tailscale corriendo en el Umbrel (hay app oficial en el store
de Umbrel):

```bash
# en el Umbrel, via SSH — apunta al BYPASS (8081), no al app_proxy
tailscale funnel --bg 8081
```

Eso publica el puerto LAN `8081` (nginx directo, sin auth de Umbrel) en
una URL publica `https://<nombre-nodo>.<tu-tailnet>.ts.net/` con
certificado SSL automatico. Los viewers entran a esa URL desde cualquier
red sin instalar nada y sin necesitar cuenta en tu Umbrel.

**Importante**: NO apuntes el funnel a `18080`. Ese puerto pasa por el
`app_proxy` de Umbrel que pide login al dashboard — tus amigos no van a
poder entrar.

Variante mas restrictiva: `tailscale serve` en lugar de `funnel` expone
el servicio solo a peers de tu tailnet (necesitan tener Tailscale
instalado y estar autorizados). Tambien apunta al `8081`:

```bash
tailscale serve --bg 8081
```

#### 5. Probar

- LAN propia (con login de Umbrel): `http://umbrel.local:18080`
- LAN directa (sin auth): `http://umbrel.local:8081`
- Internet (Funnel): `https://<nombre-nodo>.<tailnet>.ts.net/`
- Solo tailnet (Serve): misma URL que Funnel pero accesible solo desde
  peers autorizados.

---

### Estructura del repo

```
.
├── README.md                       Este archivo
├── umbrel-app-store.yml            Manifest del store (id, name)
└── vektorspaco-multicam/           La app
    ├── umbrel-app.yml              Manifest de la app
    ├── docker-compose.yml          nginx + app_proxy
    └── data/
        ├── nginx.conf              Config del nginx (CORS, cache)
        └── content/                Donde van los assets HLS (no commiteado)
```

### Troubleshooting

**La app aparece como "stopped" tras instalar:**
- `docker logs vektorspaco-multicam_nginx_1` desde SSH al Umbrel. Si dice
  "no such file" sobre `.htpasswd`, creá uno vacio: `touch
  ~/umbrel/app-data/vektorspaco-multicam/data/.htpasswd`.

**El video carga el manifest pero los `.ts` tiran 404:**
- Verificar que la estructura de carpetas dentro de `data/content/` sea
  exactamente lo que pide tu `index.html` (los paths son relativos).
- `curl -I http://umbrel.local:8081/1T/bbc/seg_00000.ts` debe retornar
  `200 OK` con `Content-Type: video/mp2t`. (Testealo siempre contra el
  bypass `8081` para descartar el app_proxy de Umbrel del medio.)

**Tailscale Funnel devuelve "could not connect":**
- `tailscale status` desde el Umbrel para verificar que el nodo esta
  autenticado y conectado.
- `tailscale funnel status` para ver si el funnel esta activo.

**No se reproduce en algunos browsers / hay glitches en seek:**
- Verificar que los headers de CORS y `Content-Type` estan llegando bien:
  `curl -I http://umbrel.local:8080/1T/bbc/index.m3u8`.
- Algunos browsers (Firefox especialmente) son estrictos con el
  `Access-Control-Allow-Origin` en HLS.

---

### Notas

- La app es 100% nginx + assets estaticos. Cero base de datos, cero
  state, cero backend. Reinicios son inocuos.
- Si queres sumar mas proyectos al reproductor, simplemente copias mas
  carpetas dentro de `data/content/` y actualizas el `index.html` raiz
  para listarlos.
- Para regenerar los assets HLS desde los videos originales, ver el
  README del proyecto "Final con seleccion de camaras" (los `.bat` con
  comandos ffmpeg NVENC).
