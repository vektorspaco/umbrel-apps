# Braulio's Umbrel Apps

Community app store con apps caseras para el stack del PH.

## Como agregar este store a tu Umbrel

1. Abri Umbrel → `App Store`
2. Click en los tres puntitos arriba a la derecha → `Community App Stores`
3. Click `Add` y pega la URL de este repo:
   ```
   https://github.com/braulio/umbrel-apps
   ```
4. Las apps aparecen como cualquier otra en el store, con un tag de "community".

---

## Multicam Player

Reproductor HLS multicam (dos streams de video sincronizados + multiples
pistas de audio) servido desde el Umbrel y expuesto a internet via
Cloudflare Tunnel, protegido por Cloudflare Access.

### Setup paso a paso

#### 1. Cloudflare: crear el tunnel

1. Ir a [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com/) →
   `Networks` → `Tunnels` → `Create a tunnel`.
2. Elegir `Cloudflared` como conector.
3. Nombre del tunnel: `umbrel-home` (o lo que prefieras).
4. **Copiar el token** que aparece (string largo que empieza con `eyJ...`).
   Lo vamos a necesitar en el paso 3.
5. Saltarse la pantalla de "install connector" (lo vamos a correr nosotros
   en Docker, no en bare metal).
6. En `Public Hostname`:
   - Subdomain: `multicam`
   - Domain: `tudominio.com` (tiene que estar en Cloudflare)
   - Service type: `HTTP`
   - URL: `multicam_nginx_1:80`
   - Guardar.

#### 2. Cloudflare Access: proteger con auth por email

1. Ir a `Zero Trust` → `Access` → `Applications` → `Add an application`.
2. Tipo: `Self-hosted`.
3. Configuracion:
   - Name: `Multicam`
   - Subdomain: `multicam`
   - Domain: `tudominio.com`
   - Session duration: `24 hours` (los amigos no se reautentican cada vez)
4. Policy:
   - Name: `Allowed viewers`
   - Action: `Allow`
   - Include: `Emails` → lista de mails de tus amigos + el tuyo.
5. Identity provider: `One-time PIN` (les llega un codigo al mail; no
   necesitan cuenta de Google ni nada).

#### 3. Umbrel: instalar la app

1. App Store → `Multicam Player` → `Install`.
2. Esperar a que termine la instalacion (va a fallar el cloudflared porque
   no tiene token todavia; es esperado).
3. SSH al Umbrel:
   ```bash
   ssh umbrel@umbrel.local
   cd ~/umbrel/app-data/multicam
   echo "TUNNEL_TOKEN=eyJhIjo...el_token_largo" > .env
   ```
4. Reiniciar la app desde la UI de Umbrel (Settings → Restart).
5. Verificar que el tunnel se conecto:
   - En Cloudflare Zero Trust → `Networks` → `Tunnels`, el tunnel debe
     aparecer en estado `HEALTHY`.

#### 4. Subir el contenido

Por SFTP/SCP, subir los archivos a:

```
~/umbrel/app-data/multicam/data/content/
├── index.html
├── 1T/
│   ├── bbc.m3u8
│   ├── tactica.m3u8
│   ├── audio-es.m3u8
│   ├── audio-en.m3u8
│   └── segments/
│       ├── bbc-001.ts
│       └── ...
└── 2T/
    └── ...
```

Ejemplo con rsync desde tu maquina:

```bash
rsync -avh --progress ./multicam-assets/ \
  umbrel@umbrel.local:~/umbrel/app-data/multicam/data/content/
```

#### 5. Cloudflare: cachear los .ts agresivamente

Esto es lo que hace que tu uplink de 100 Mb/s no se sature: Cloudflare
cachea cada segmento la primera vez, los demas viewers lo bajan del edge
de CF, no de tu casa.

1. Ir al dashboard de Cloudflare → tu dominio → `Caching` → `Cache Rules`.
2. Crear regla:
   - Name: `Cache HLS segments`
   - If: `URI Path` `ends with` `.ts`
   - Then:
     - `Cache eligibility`: `Eligible for cache`
     - `Edge TTL`: `Override origin` → `1 year`
     - `Browser TTL`: `Override origin` → `1 year`
3. Guardar.

#### 6. Probar

1. Abrir `https://multicam.tudominio.com` desde otra red (datos del celu).
2. Te debe redirigir a la pagina de Cloudflare Access.
3. Ingresar email → recibir codigo → ingresar codigo → ver el reproductor.

---

### Troubleshooting

**El tunnel no conecta:**
- Verificar que el `.env` tiene el formato correcto: `TUNNEL_TOKEN=eyJ...`
  (sin comillas, sin espacios).
- `docker logs multicam_cloudflared_1` desde SSH te muestra el error.

**Cloudflare Access muestra error 1033 o similar:**
- El tunnel esta caido. Reiniciar la app.

**Los .ts se descargan pero el video buffera:**
- Verificar que la regla de Cache Rules esta activa: ir a `Caching` →
  `Cache Analytics` y ver que aparecen hits en los .ts.
- Si todos son MISS, el origin (tu Umbrel) esta sirviendo todo y tu uplink
  se satura.

**Antivirus de un amigo bloquea el dominio:**
- Verificar que usas tu propio dominio, no `*.trycloudflare.com`. Los
  trycloudflare estan moderadamente quemados.

---

### Consideraciones de TOS

El uso esta cubierto por:
- Cloudflare Tunnel: gratis sin limite, sin restriccion de tipo de contenido.
- Cloudflare Access: gratis hasta 50 usuarios.
- Cloudflare CDN cacheando archivos servidos por un origin propio: zona
  gris pero historicamente tolerada para uso bajo. El TOS 2.8 apunta a
  hostear video pesado *en Cloudflare* (Pages, R2 expuesto), no a usar
  CF como CDN delante de tu propio servidor.

Para 2-5 viewers ocasionales, sin promocion, sin uso comercial, el riesgo
es bajo. Si esto creciera, la opcion limpia es mover a Cloudflare Stream
($1/1000 minutos vistos).
