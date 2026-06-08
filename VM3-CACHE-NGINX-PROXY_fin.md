# 🖥️ VM3 — Proxy Inverso NGINX + Servidor Caché
### Proyecto 17: Infraestructura de Noticias con CDN Simulado
> **SIS313 — USFX Chuquisaca | Semestre 1/2026**

---

## 📋 Ficha Técnica

| Campo | Detalle |
|-------|---------|
| **Nombre de VM** | `vm3-cache` |
| **Rol** | NGINX Proxy Inverso + Caché estático + Fail2ban |
| **Sistema Operativo** | Ubuntu Server 22.04 LTS |
| **RAM** | 1024 MB |
| **CPU** | 2 vCPU |
| **Disco** | 25 GB |
| **Adaptador de red** | Adaptador puente (Bridge) |
| **IP** | `10.73.15.102/24` |
| **Gateway** | `10.73.15.1` (hotspot celular) |
| **DNS** | `10.73.15.100` (VM1-BIND9) |
| **Estado** | 🟢 Operativo |

> ⚠️ **Importante:** Todas las VMs usan **Adaptador Puente (Bridge)** en VirtualBox. Cada VM aparece como una computadora más dentro de la red del hotspot. No se usan VLANs ni redes internas.

---

## 🗺️ Rol en la Arquitectura

```
Hotspot celular — 10.73.15.0/24
         │
   [VM1 · 10.73.15.100]  DNS BIND9
         │
   [VM3-Cache · 10.73.15.102]   ◄── NGINX Proxy Inverso
         │
         ├── /             → Sirve estáticos locales (HTML, CSS, imágenes)
         ├── /static/*     → Sirve desde caché local de NGINX
         ├── /api/*        → Proxy → VM2-Editorial (10.73.15.101:3000)
         └── /nginx-health → Status del proxy
```

Esta es la VM **más importante del proyecto**. NGINX recibe TODO el tráfico de los usuarios, decide si lo responde directamente (estáticos en caché) o lo reenvía al servidor editorial (VM2). También incluye Fail2ban para bloquear ataques.

---

## ⚙️ PASO 1 — Configuración Inicial del Sistema

### 1.1 Actualizar el sistema
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget nano git net-tools
```
- `net-tools` → para ver IPs con `ifconfig`
- `curl` → para probar URLs
- `nano` → editor de texto

### 1.2 Configurar hostname
```bash
sudo hostnamectl set-hostname vm3-cache
```
Verificamos:
```bash
hostnamectl
```

### 1.3 Configurar IP estática con Netplan

Abrimos el archivo de configuración de red:
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

> 💡 **Cómo usar nano:** Las flechas del teclado mueven el cursor. Cuando termines de editar presiona `Ctrl+O` para guardar, luego `Enter` para confirmar el nombre, y `Ctrl+X` para salir.

Borramos todo lo que haya y escribimos exactamente esto (respetando la indentación con espacios, **NO con tabulaciones**):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 10.73.15.102/24
      routes:
        - to: default
          via: 10.73.15.1
      nameservers:
        addresses:
          - 10.73.15.100
        search:
          - noticias.local
```

> ⚠️ `enp0s3` es el nombre de la interfaz en VirtualBox. Si tu interfaz tiene otro nombre, cámbialo. Puedes verlo con: `ip link show`

Guardamos (`Ctrl+O` → `Enter` → `Ctrl+X`) y aplicamos:
```bash
sudo netplan apply
```

### 1.4 Verificar conectividad
```bash
ip addr show
# Buscar: inet 10.73.15.102/24

ping -c 3 10.73.15.1    # Gateway (hotspot)
ping -c 3 10.73.15.100  # VM1-Router DNS
ping -c 3 10.73.15.101  # VM2-Editorial
ping -c 3 10.73.15.103  # VM4-Admin
```

### 1.5 Verificar resolución DNS
```bash
dig www.noticias.local @10.73.15.100
dig static.noticias.local @10.73.15.100
dig api.noticias.local @10.73.15.100
```

---

## 🌐 PASO 2 — Instalación y Configuración de NGINX

### 2.1 Instalar NGINX
```bash
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```
La salida debe mostrar `active (running)` en verde.

### 2.2 Verificar versión
```bash
nginx -v
```

---

## 📁 PASO 3 — Estructura de Contenido Estático

### 3.1 Crear directorios de contenido
```bash
sudo mkdir -p /var/www/noticias/{static,assets/{css,js,img},cache}
sudo chown -R www-data:www-data /var/www/noticias
```

### 3.2 Crear página principal del portal
```bash
sudo nano /var/www/noticias/index.html
```

Escribimos el siguiente contenido:

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Portal de Noticias — USFX</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: 'Segoe UI', sans-serif; background: #0f172a; color: #e2e8f0; }
    header { background: #1e3a5f; padding: 1.5rem 2rem; border-bottom: 3px solid #3b82f6; }
    header h1 { font-size: 2rem; color: #93c5fd; }
    header p  { color: #94a3b8; font-size: 0.9rem; }
    .badge { display: inline-block; background: #10b981; color: white;
             padding: 2px 10px; border-radius: 12px; font-size: 0.75rem; margin-left: 10px; }
    main { max-width: 900px; margin: 2rem auto; padding: 0 1rem; }
    .card { background: #1e293b; border: 1px solid #334155; border-radius: 10px;
            padding: 1.5rem; margin-bottom: 1.5rem; transition: transform 0.2s; }
    .card:hover { transform: translateY(-3px); border-color: #3b82f6; }
    .card h2 { color: #60a5fa; margin-bottom: 0.5rem; }
    .card p { color: #94a3b8; line-height: 1.6; }
    .meta { font-size: 0.8rem; color: #475569; margin-top: 0.8rem; }
    .tag { background: #1d4ed8; color: #bfdbfe; padding: 2px 8px;
           border-radius: 6px; font-size: 0.75rem; }
    footer { text-align: center; padding: 2rem; color: #475569; border-top: 1px solid #1e293b; }
    #api-status { background: #0f2027; border-left: 4px solid #10b981;
                  padding: 1rem; border-radius: 6px; font-family: monospace;
                  font-size: 0.85rem; margin-bottom: 2rem; }
  </style>
</head>
<body>
  <header>
    <h1>📰 Portal de Noticias <span class="badge">CDN Simulado</span></h1>
    <p>Infraestructura: VM3-Cache (NGINX) → VM2-Editorial (Node.js) | SIS313 USFX 2026</p>
  </header>
  <main>
    <div id="api-status">⏳ Consultando API dinámica en vm2-editorial...</div>
    <div id="noticias-container">
      <div class="card">
        <h2>📡 Contenido estático servido por NGINX</h2>
        <p>Esta página es un recurso estático almacenado en caché en VM3. Las noticias
           de abajo son cargadas dinámicamente desde la API en VM2-Editorial.</p>
        <div class="meta"><span class="tag">static</span> Servido desde 10.73.15.102</div>
      </div>
    </div>
  </main>
  <footer>Proyecto 17 — SIS313 USFX 2026 | VM3-Cache · NGINX Proxy Inverso</footer>
  <script>
    fetch('/api/noticias')
      .then(r => r.json())
      .then(data => {
        document.getElementById('api-status').innerHTML =
          `✅ API OK | ${data.total} noticias cargadas desde vm2-editorial:3000`;
        const container = document.getElementById('noticias-container');
        data.noticias.forEach(n => {
          container.innerHTML += `
            <div class="card">
              <h2>${n.titulo}</h2>
              <p>${n.contenido}</p>
              <div class="meta">
                <span class="tag">${n.categoria}</span>
                👤 ${n.autor} · 👁 ${n.vistas} vistas · 📅 ${new Date(n.fecha).toLocaleDateString('es-BO')}
              </div>
            </div>`;
        });
      })
      .catch(e => {
        document.getElementById('api-status').innerHTML =
          `❌ Error al conectar con API: ${e.message}`;
      });
  </script>
</body>
</html>
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

### 3.3 Crear página de contenido estático
```bash
sudo nano /var/www/noticias/static/index.html
```

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Recursos Estáticos — CDN Simulado</title>
  <style>
    body { font-family: monospace; background: #0d1117; color: #58a6ff; padding: 2rem; }
    h1 { color: #3fb950; } pre { background: #161b22; padding: 1rem; border-radius: 6px; }
  </style>
</head>
<body>
  <h1>📦 Servidor de Contenido Estático</h1>
  <p>Este recurso es servido directamente por NGINX desde el caché local.</p>
  <pre>
Servidor : vm3-cache (10.73.15.102)
Ruta     : /var/www/noticias/static/
Tipo     : Contenido estático (HTML, CSS, imágenes)
CDN sim. : NGINX Cache-Control headers activos
  </pre>
</body>
</html>
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

---

## ⚙️ PASO 4 — Configuración de NGINX como Proxy Inverso + Caché

### 4.1 Eliminar sitio default
```bash
sudo rm /etc/nginx/sites-enabled/default
```

### 4.2 Crear configuración del portal
```bash
sudo nano /etc/nginx/sites-available/noticias
```

Escribimos el siguiente contenido:

```nginx
# ─────────────────────────────────────────────────────────────
#  NGINX — Portal de Noticias · CDN Simulado
#  Proyecto 17 SIS313 USFX | VM3-Cache
# ─────────────────────────────────────────────────────────────

# Zona de caché NGINX
proxy_cache_path /var/cache/nginx/noticias
    levels=1:2
    keys_zone=noticias_cache:10m
    max_size=500m
    inactive=60m
    use_temp_path=off;

# Upstream: servidor editorial dinámico (VM2)
upstream editorial_backend {
    server 10.73.15.101:3000;
    keepalive 32;
}

server {
    listen 80;
    server_name www.noticias.local static.noticias.local api.noticias.local noticias.local;

    root  /var/www/noticias;
    index index.html;

    # ── Logs ────────────────────────────────────────────────
    access_log /var/log/nginx/noticias_access.log;
    error_log  /var/log/nginx/noticias_error.log;

    # ── Cabeceras de seguridad ───────────────────────────────
    add_header X-Served-By      "vm3-cache"          always;
    add_header X-Frame-Options  "SAMEORIGIN"          always;
    add_header X-Content-Type-Options "nosniff"       always;

    # ── Rutas estáticas (servidas directamente por NGINX) ───
    location /static/ {
        alias /var/www/noticias/static/;
        expires 7d;
        add_header Cache-Control "public, immutable";
        add_header X-Cache-Type  "STATIC-LOCAL";
    }

    location ~* \.(css|js|png|jpg|gif|ico|woff2?)$ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000";
        add_header X-Cache-Type  "STATIC-ASSET";
    }

    # ── Rutas dinámicas (proxy → VM2-Editorial) ─────────────
    location /api/ {
        proxy_pass         http://editorial_backend;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;

        # Caché para API
        proxy_cache            noticias_cache;
        proxy_cache_valid      200 5m;
        proxy_cache_use_stale  error timeout updating;
        add_header             X-Cache-Status $upstream_cache_status;
        add_header             X-Cache-Type   "PROXY-DYNAMIC";
    }

    # ── Health check del proxy ───────────────────────────────
    location /nginx-health {
        access_log off;
        return 200 '{"status":"OK","servidor":"vm3-cache","rol":"proxy-inverso"}';
        add_header Content-Type application/json;
    }

    # ── Portal principal (estático) ──────────────────────────
    location / {
        try_files $uri $uri/ /index.html;
        add_header X-Cache-Type "STATIC-ROOT";
    }

    # ── Bloquear rutas administrativas (Fail2ban las monitorea) ─
    location ~ ^/(admin|wp-admin|phpmyadmin|\.env) {
        return 403;
        access_log /var/log/nginx/noticias_blocked.log;
    }
}
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

### 4.3 Crear directorio de caché
```bash
sudo mkdir -p /var/cache/nginx/noticias
sudo chown www-data:www-data /var/cache/nginx/noticias
```

### 4.4 Activar sitio y verificar
```bash
sudo ln -s /etc/nginx/sites-available/noticias /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl status nginx
```
Si `nginx -t` muestra errores, volver a abrir el archivo con `sudo nano` y corregirlos.

---

## 🛡️ PASO 5 — Configuración de Fail2ban

### 5.1 Instalar Fail2ban
```bash
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
```

### 5.2 Configurar jail para NGINX
```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 5
backend  = systemd

[sshd]
enabled  = true
port     = 2222
logpath  = /var/log/auth.log
maxretry = 3

[nginx-http-auth]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/noticias_error.log
maxretry = 5

[nginx-botsearch]
enabled  = true
port     = http,https
filter   = nginx-botsearch
logpath  = /var/log/nginx/noticias_access.log
maxretry = 2
findtime = 60
bantime  = 86400
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

### 5.3 Crear filtro para rutas bloqueadas
```bash
sudo nano /etc/fail2ban/filter.d/nginx-botsearch.conf
```

```ini
[Definition]
failregex = ^<HOST> .* "(GET|POST) /(admin|wp-admin|phpmyadmin|\.env|\.git)
ignoreregex =
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

### 5.4 Iniciar Fail2ban
```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client status nginx-botsearch
```

---

## 🔥 PASO 6 — Firewall Local (UFW)

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing

# HTTP público (portal de noticias)
sudo ufw allow 80/tcp comment 'HTTP portal noticias'

# SSH endurecido
sudo ufw allow 2222/tcp comment 'SSH endurecido'

sudo ufw enable
```
Escribe `y` cuando pregunte si continuar.

```bash
sudo ufw status verbose
```

---

## 🔐 PASO 7 — Hardening de SSH

```bash
sudo nano /etc/ssh/sshd_config
```

Busca estas líneas con `Ctrl+W` y modifícalas (o agrégalas si no existen):

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

```bash
sudo systemctl restart ssh
sudo systemctl status ssh
```

> ⚠️ Antes de cerrar la sesión SSH actual, abre OTRA terminal y prueba conectarte con la clave. Si funciona, recién cierra la sesión anterior.

---

## ✅ PASO 8 — Verificaciones para el Parcial

```bash
# 1. Confirmar IP correcta
ip addr show | grep "inet "
# Debe mostrar: inet 10.73.15.102/24

# 2. Ping a todas las VMs
ping -c 2 10.73.15.100  # VM1-Router
ping -c 2 10.73.15.101  # VM2-Editorial
ping -c 2 10.73.15.103  # VM4-Admin

# 3. NGINX activo y configuración válida
sudo nginx -t
sudo systemctl status nginx --no-pager

# 4. Portal accesible
curl http://localhost/
curl http://localhost/nginx-health

# 5. Proxy a editorial funcionando
curl http://localhost/api/noticias
curl http://localhost/api/info

# 6. Contenido estático
curl http://localhost/static/

# 7. Cabeceras de caché (ver X-Cache-Status y X-Cache-Type)
curl -I http://localhost/api/noticias

# 8. DNS resolviendo desde VM1
dig www.noticias.local @10.73.15.100 +short
# Debe devolver: 10.73.15.102
dig static.noticias.local @10.73.15.100 +short
dig api.noticias.local @10.73.15.100 +short

# 9. Fail2ban activo
sudo fail2ban-client status
sudo fail2ban-client status nginx-botsearch

# 10. Firewall
sudo ufw status verbose

# 11. SSH endurecido
grep -E "^(Port|PermitRootLogin|PasswordAuthentication)" /etc/ssh/sshd_config

# 12. Logs de NGINX
sudo tail -20 /var/log/nginx/noticias_access.log
sudo tail -10 /var/log/nginx/noticias_blocked.log

# 13. Simular escaneo de rutas admin (para demostrar Fail2ban)
curl http://localhost/admin
curl http://localhost/wp-admin
curl http://localhost/.env
# Verificar que Fail2ban detecta los intentos
sudo fail2ban-client status nginx-botsearch
```

---

## 🚨 Comandos de Emergencia

```bash
# NGINX no arranca — ver el error exacto
sudo nginx -t
sudo journalctl -u nginx -n 30

# Fail2ban baneó tu propia IP (desbanear)
sudo fail2ban-client set nginx-botsearch unbanip 10.73.15.103

# Ver todas las IPs baneadas
sudo fail2ban-client status nginx-botsearch

# Limpiar caché NGINX
sudo find /var/cache/nginx/noticias -type f -delete
sudo systemctl reload nginx

# Ver conexiones activas al proxy
ss -tlnp | grep nginx
ss -tnp | grep :80

# Reiniciar la red si perdiste conectividad
sudo netplan apply

# Si te quedaste sin acceso SSH
# Accede directo por consola VirtualBox y ejecuta:
sudo ufw disable
# Luego corrige las reglas y vuelve a habilitar
```

---

*Proyecto 17 — SIS313 USFX | VM3 · 10.73.15.102 · Proxy Inverso NGINX + Caché + Fail2ban*
