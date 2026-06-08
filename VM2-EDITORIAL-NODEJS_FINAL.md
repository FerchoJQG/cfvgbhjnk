# 🖥️ VM2 — Servidor Editorial (Node.js + PM2)
### Proyecto 17: Infraestructura de Noticias con CDN Simulado
> **SIS313 — USFX Chuquisaca | Semestre 1/2026**
> **Responsable:** Limbert

---

## 📋 Ficha Técnica

| Campo | Detalle |
|-------|---------|
| **Nombre de VM** | `vm2-editorial` |
| **Responsable** | Limbert |
| **Rol** | Servidor Node.js + PM2 (API dinámica de noticias) |
| **Sistema Operativo** | Ubuntu Server 22.04 LTS |
| **RAM** | 1536 MB |
| **CPU** | 2 vCPU |
| **Disco** | 25 GB |
| **Adaptador de red** | Adaptador puente (Bridge) |
| **IP** | `10.73.15.101/24` |
| **Gateway** | `10.73.15.1` (hotspot celular) |
| **DNS** | `10.73.15.100` (VM1-BIND9) |
| **Estado** | 🟢 Operativo |

> ⚠️ **Importante:** Todas las VMs usan **Adaptador Puente (Bridge)** en VirtualBox. No se usan VLANs ni redes internas. VM2 solo acepta conexiones del puerto 3000 desde VM3 y VM4.

---

## 🗺️ Rol en la Arquitectura

```
[VM3-Cache · Fer · 10.73.15.102]
         │  NGINX hace proxy de /api/* hacia aquí
         ▼
[VM2-Editorial · Limbert · 10.73.15.101]
  Node.js + PM2
  Puerto 3000
  API REST de noticias
```

Esta VM entrega el **contenido dinámico** del portal. NGINX en VM3 reenvía todas las peticiones `/api/*` hacia el puerto 3000 de esta máquina. VM2 **no** es accesible directamente desde el navegador, solo a través del proxy.

---

## ⚙️ PASO 1 — Configuración Inicial

### 1.1 Actualizar el sistema
```bash
sudo apt update && sudo apt upgrade -y
```
Instalamos herramientas básicas:
```bash
sudo apt install -y curl wget nano git build-essential net-tools
```
- `build-essential` → compiladores necesarios para algunos paquetes de Node.js
- `net-tools` → para verificar puertos con `netstat`

### 1.2 Configurar el hostname
```bash
sudo hostnamectl set-hostname vm2-editorial
```
Verificamos:
```bash
hostnamectl
```

### 1.3 Configurar IP estática con Netplan

Necesitamos IP fija para que VM3 siempre sepa dónde enviar las peticiones. Abrimos el archivo:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

> 💡 **Cómo usar nano:** Flechas para mover el cursor. `Ctrl+O` para guardar, `Enter` para confirmar, `Ctrl+X` para salir. `Ctrl+W` para buscar texto.

Borramos todo y escribimos (usa espacios, **NO tabulaciones**):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 10.73.15.101/24
      routes:
        - to: default
          via: 10.73.15.1
      nameservers:
        addresses:
          - 10.73.15.100
        search:
          - noticias.local
```

> ⚠️ Verifica el nombre de tu interfaz con `ip link show`. Si no es `enp0s3`, cámbialo.

Guardamos y aplicamos:
```bash
sudo netplan apply
```

### 1.4 Verificar conectividad y DNS

Verificamos que la IP quedó bien:
```bash
ip addr show
```
Debe mostrar: `inet 10.73.15.101/24`

Hacemos ping a VM1 (DNS) y VM3 (NGINX):
```bash
ping -c 3 10.73.15.100   # VM1-DNS
ping -c 3 10.73.15.102   # VM3-Cache
```

Probamos que el DNS de VM1 funciona correctamente:
```bash
dig @10.73.15.100 www.noticias.local
```
Debe resolver a `10.73.15.102`.

---

## 🟢 PASO 2 — Instalación de Node.js y PM2

### 2.1 Instalar Node.js 20 LTS

Primero descargamos el script de configuración del repositorio oficial de NodeSource:
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
```
Luego instalamos Node.js:
```bash
sudo apt install -y nodejs
```
Verificamos las versiones instaladas:
```bash
node --version   # Debe mostrar v20.x.x
npm --version    # Debe mostrar 10.x.x
```

### 2.2 Instalar PM2 globalmente

PM2 es el gestor de procesos que mantiene la API corriendo aunque se cierre la terminal o se reinicie el servidor:
```bash
sudo npm install -g pm2
pm2 --version
```

---

## 📁 PASO 3 — Crear Estructura del Proyecto

### 3.1 Crear directorios
```bash
sudo mkdir -p /opt/noticias/api
sudo mkdir -p /opt/noticias/logs
sudo mkdir -p /opt/noticias/backups
```

Asignamos propiedad al usuario actual para poder escribir sin sudo:
```bash
sudo chown -R $USER:$USER /opt/noticias
```

### 3.2 Inicializar el proyecto Node.js
```bash
cd /opt/noticias/api
npm init -y
```
El `-y` acepta todos los valores por defecto automáticamente.

Instalamos las dependencias necesarias:
```bash
npm install express cors morgan
```
- `express` → framework web para crear la API REST
- `cors` → permite que el navegador haga peticiones desde otros dominios (necesario para el proxy NGINX)
- `morgan` → registra en logs todas las peticiones HTTP

---

## 🚀 PASO 4 — Crear la API de Noticias

Creamos el archivo principal de la aplicación:
```bash
sudo nano /opt/noticias/api/app.js
```

Escribe el siguiente código completo:

```javascript
const express = require('express');
const cors    = require('cors');
const morgan  = require('morgan');

const app  = express();
const PORT = process.env.PORT || 3000;

// ── Middlewares (se ejecutan en cada petición) ────────────────
app.use(cors());                  // Permite peticiones cross-origin
app.use(morgan('combined'));      // Registra cada petición en consola
app.use(express.json());          // Permite recibir JSON en el body

// ── Base de datos simulada ────────────────────────────────────
// En un proyecto real, esto vendría de una base de datos como PostgreSQL
const noticias = [
  {
    id: 1,
    titulo: "Infraestructura tecnológica en USFX",
    categoria: "tecnologia",
    contenido: "La carrera de Ingeniería de Sistemas implementa nuevo centro de datos con tecnología de punta para los proyectos de red del semestre.",
    autor: "Redacción USFX",
    fecha: new Date().toISOString(),
    vistas: 1240
  },
  {
    id: 2,
    titulo: "Avances en redes de fibra óptica en Sucre",
    categoria: "tecnologia",
    contenido: "La ciudad de Sucre moderniza su infraestructura de telecomunicaciones con fibra óptica de alta velocidad en los principales distritos.",
    autor: "Redacción TIC",
    fecha: new Date().toISOString(),
    vistas: 892
  },
  {
    id: 3,
    titulo: "Seguridad informática: mejores prácticas 2026",
    categoria: "seguridad",
    contenido: "Expertos recomiendan hardening en todos los servidores, uso de claves SSH, firewalls activos y monitoreo constante de logs.",
    autor: "Equipo de Seguridad",
    fecha: new Date().toISOString(),
    vistas: 3105
  },
  {
    id: 4,
    titulo: "Bolivia avanza en transformación digital",
    categoria: "nacional",
    contenido: "El gobierno impulsa la digitalización de servicios públicos en todo el país, con énfasis en ciudades intermedias.",
    autor: "Redacción Nacional",
    fecha: new Date().toISOString(),
    vistas: 2450
  }
];

// ── RUTA: Health check ────────────────────────────────────────
// GET /health — Para verificar que la API está viva
app.get('/health', (req, res) => {
  res.json({
    status:    'OK',
    servidor:  'vm2-editorial',
    ip:        '10.73.15.101',
    timestamp: new Date().toISOString(),
    uptime:    Math.floor(process.uptime()) + 's'
  });
});

// ── RUTA: Todas las noticias ──────────────────────────────────
// GET /api/noticias          → devuelve todas
// GET /api/noticias?categoria=tecnologia → filtra por categoría
app.get('/api/noticias', (req, res) => {
  const { categoria } = req.query;
  const resultado = categoria
    ? noticias.filter(n => n.categoria === categoria)
    : noticias;
  res.json({ total: resultado.length, noticias: resultado });
});

// ── RUTA: Noticia por ID ──────────────────────────────────────
// GET /api/noticias/1 → devuelve la noticia con id=1
app.get('/api/noticias/:id', (req, res) => {
  const noticia = noticias.find(n => n.id === parseInt(req.params.id));
  if (!noticia) {
    return res.status(404).json({ error: 'Noticia no encontrada' });
  }
  res.json(noticia);
});

// ── RUTA: Información del servidor ───────────────────────────
// GET /api/info — Para identificar qué servidor responde
app.get('/api/info', (req, res) => {
  res.json({
    servidor:  'vm2-editorial',
    ip:        '10.73.15.101',
    rol:       'API dinámica de noticias',
    version:   '1.0.0',
    node:      process.version,
    timestamp: new Date().toISOString()
  });
});

// ── Iniciar servidor ──────────────────────────────────────────
// Escucha en 0.0.0.0 para aceptar conexiones de cualquier IP
app.listen(PORT, '0.0.0.0', () => {
  console.log(`[VM2-Editorial] API corriendo en http://10.73.15.101:${PORT}`);
  console.log(`[VM2-Editorial] Endpoints disponibles:`);
  console.log(`  GET /health`);
  console.log(`  GET /api/noticias`);
  console.log(`  GET /api/noticias/:id`);
  console.log(`  GET /api/info`);
});
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

Probamos que el archivo funciona antes de usar PM2:
```bash
cd /opt/noticias/api
node app.js
```
Debes ver los mensajes de `[VM2-Editorial] API corriendo en...`. Presiona `Ctrl+C` para detenerlo.

---

## ⚙️ PASO 5 — Gestión con PM2

### 5.1 Iniciar la aplicación con PM2
```bash
cd /opt/noticias/api
pm2 start app.js --name "noticias-api" --log /opt/noticias/logs/api.log
```
- `--name` → nombre para identificar el proceso
- `--log` → archivo donde guardar los logs

Verificamos que está corriendo:
```bash
pm2 status
```
Debe mostrar `noticias-api` en estado `online`.

### 5.2 Configurar inicio automático con el sistema

Cuando se reinicia la VM, queremos que la API arranque sola:
```bash
pm2 startup systemd
```
Este comando imprime en pantalla otro comando que debes copiar y ejecutar. Se ve algo así:
```bash
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u TU_USUARIO --hp /home/TU_USUARIO
```
Copia **ese comando exacto** que aparece en tu terminal y ejecútalo.

Luego guardamos el estado actual de PM2:
```bash
pm2 save
```

### 5.3 Comandos útiles de PM2

```bash
pm2 status                        # Ver estado de todos los procesos
pm2 logs noticias-api             # Ver logs en tiempo real (Ctrl+C para salir)
pm2 logs noticias-api --lines 30  # Ver las últimas 30 líneas de log
pm2 restart noticias-api          # Reiniciar la API
pm2 stop noticias-api             # Detener la API
pm2 reload noticias-api           # Reiniciar sin cortar conexiones activas
pm2 monit                         # Dashboard visual en terminal (Ctrl+C para salir)
```

---

## 💾 PASO 6 — Script de Backup

Este script crea una copia comprimida de toda la API en `/opt/noticias/backups/`:

```bash
sudo nano /opt/noticias/backups/backup_editorial.sh
```

```bash
#!/bin/bash
# ─────────────────────────────────────────────────────────────
#  Script de Backup — VM2-Editorial | Proyecto 17 SIS313 USFX
# ─────────────────────────────────────────────────────────────

FECHA=$(date +%Y%m%d_%H%M%S)
DESTINO="/opt/noticias/backups"
ARCHIVO="backup_editorial_${FECHA}.tar.gz"
LOG="/opt/noticias/logs/backup.log"

echo "[$(date)] Iniciando backup..." | tee -a "$LOG"

# Crear el archivo comprimido (excluye la carpeta de backups para no hacer backup del backup)
tar -czf "${DESTINO}/${ARCHIVO}" \
    --exclude="${DESTINO}" \
    /opt/noticias/api

# Verificar que el archivo no está corrupto
if tar -tzf "${DESTINO}/${ARCHIVO}" > /dev/null 2>&1; then
    TAMANIO=$(du -sh "${DESTINO}/${ARCHIVO}" | cut -f1)
    echo "[$(date)] ✅ Backup exitoso: ${ARCHIVO} (${TAMANIO})" | tee -a "$LOG"
else
    echo "[$(date)] ❌ ERROR: El backup está corrupto" | tee -a "$LOG"
    exit 1
fi

# Conservar solo los últimos 7 backups para no llenar el disco
ls -t "${DESTINO}"/backup_editorial_*.tar.gz | tail -n +8 | xargs rm -f 2>/dev/null
echo "[$(date)] Total backups guardados: $(ls ${DESTINO}/backup_editorial_*.tar.gz | wc -l)" | tee -a "$LOG"
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

Damos permiso de ejecución:
```bash
chmod +x /opt/noticias/backups/backup_editorial.sh
```

Probamos que funciona:
```bash
/opt/noticias/backups/backup_editorial.sh
ls -lh /opt/noticias/backups/
```

### 6.1 Programar backup automático diario con cron
```bash
crontab -e
```
Si pregunta qué editor usar, elige `1` (nano). Agrega esta línea al final:
```
# Backup diario a las 02:00 AM
0 2 * * * /opt/noticias/backups/backup_editorial.sh
```
Guarda y sal: `Ctrl+O` → `Enter` → `Ctrl+X`

---

## 🔥 PASO 7 — Firewall (UFW)

Protegemos VM2 para que el puerto 3000 solo sea accesible desde VM3 (NGINX) y VM4 (Admin). Nadie más puede conectarse a la API directamente.

```bash
sudo apt install -y ufw
```

Políticas por defecto:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

SSH solo desde VM4-Admin (para que Melany pueda administrar esta VM):
```bash
sudo ufw allow from 10.73.15.103 to any port 2222 comment 'SSH desde VM4-Admin'
```

Puerto 3000 (Node.js API) solo desde VM3-Cache y VM4-Admin:
```bash
sudo ufw allow from 10.73.15.102 to any port 3000 comment 'API desde VM3-Cache'
sudo ufw allow from 10.73.15.103 to any port 3000 comment 'API desde VM4-Admin'
```

> 💡 **¿Por qué solo VM3 y VM4?** VM2 no tiene IP pública ni interfaz web. Solo VM3 (NGINX) hace de intermediario con el usuario, y VM4 (Admin) necesita acceder para monitoreo y backup.

Activamos el firewall:
```bash
sudo ufw enable
```

Verificamos las reglas:
```bash
sudo ufw status verbose
```

---

## 🔐 PASO 8 — Hardening de SSH

```bash
sudo nano /etc/ssh/sshd_config
```

Busca (`Ctrl+W`) y modifica estas líneas:
```
Port 2222
PermitRootLogin no
PasswordAuthentication no
MaxAuthTries 3
```

Reiniciamos SSH:
```bash
sudo systemctl restart ssh
```

Verificamos que los cambios quedaron:
```bash
grep -E "^(Port|PermitRootLogin|PasswordAuthentication)" /etc/ssh/sshd_config
```

---

## 👤 PASO 9 — Usuarios y Permisos

Creamos un usuario dedicado para correr la aplicación (sin privilegios de root):
```bash
sudo useradd -m -s /bin/bash noticias-app
sudo usermod -aG sudo noticias-app
```

Asignamos los permisos correctos:
```bash
# El usuario noticias-app es dueño de toda la carpeta
sudo chown -R noticias-app:noticias-app /opt/noticias

# Permisos: dueño puede leer/escribir/ejecutar; grupo leer/ejecutar; otros nada
sudo chmod -R 750 /opt/noticias

# Los logs solo los puede leer el dueño y grupo
sudo chmod -R 640 /opt/noticias/logs

# Verificar
ls -la /opt/noticias/
```

---

## ✅ PASO 10 — Verificaciones para el Parcial

```bash
# 1. IP correcta
ip addr show | grep "inet "
# Debe mostrar: inet 10.73.15.101/24

# 2. Node.js y PM2 instalados correctamente
node --version && npm --version && pm2 --version

# 3. API corriendo con PM2
pm2 status
# Debe mostrar noticias-api en estado "online"

# 4. Probar todos los endpoints de la API localmente
curl http://localhost:3000/health
# Debe devolver: {"status":"OK",...}

curl http://localhost:3000/api/noticias
# Debe devolver lista de noticias en JSON

curl http://localhost:3000/api/noticias/1
# Debe devolver la noticia con id=1

curl http://localhost:3000/api/info
# Debe devolver info del servidor

# 5. API accesible desde la red (ejecutar desde VM3 o VM4)
curl http://10.73.15.101:3000/health

# 6. Firewall activo con reglas correctas
sudo ufw status verbose

# 7. SSH endurecido
grep -E "^(Port|PermitRootLogin|PasswordAuthentication)" /etc/ssh/sshd_config

# 8. Permisos del directorio
ls -la /opt/noticias/

# 9. Backup funciona
/opt/noticias/backups/backup_editorial.sh
ls -lh /opt/noticias/backups/

# 10. PM2 configurado para autostart
pm2 list
systemctl status pm2-$USER --no-pager
```

---

## 🚨 Comandos de Emergencia

```bash
# La API no responde — ver logs detallados
pm2 logs noticias-api --lines 50

# Reiniciar la API
pm2 restart noticias-api

# Ver si el puerto 3000 está en uso por otro proceso
ss -tlnp | grep 3000

# Eliminar y volver a iniciar desde cero
pm2 delete noticias-api
cd /opt/noticias/api
pm2 start app.js --name "noticias-api" --log /opt/noticias/logs/api.log
pm2 save

# Error "Cannot find module 'express'" — reinstalar dependencias
cd /opt/noticias/api
npm install

# Ver logs generales del sistema
sudo journalctl -f
```

---

*Proyecto 17 — SIS313 USFX 2026 | VM2 · Limbert · 10.73.15.101*
