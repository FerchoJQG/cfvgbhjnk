# Guias-para-el-Proyectio-finale


### Proyecto 17: Infraestructura de Noticias con CDN Simulado
> **Universidad San Francisco Xavier de Chuquisaca | Semestre 1/2026**
> **Docente:** Ing. Marcelo Quispe Ortega | **Fecha:** 29 de mayo de 2026

---

## 👥 Integrantes y Asignaciones

| Integrante | VM | IP Real | Rol |
|------------|----|---------|-----|
| **Danner** | VM1 | `10.73.15.100` | DNS BIND9 + Firewall UFW |
| **Limbert** | VM2 | `10.73.15.101` | Node.js + PM2 (API dinámica) |
| **Fer** | VM3 | `10.73.15.102` | NGINX Proxy Inverso + Caché + Fail2ban |
| **Melany** | VM4 | `10.73.15.103` | Administración + Monitoreo + Backups |

> **Red:** Hotspot celular `10.73.15.0/24` | Gateway: `10.73.15.1` | Adaptador puente (Bridge)

---

## 📋 Tabla de Infraestructura

> Entregar impresa o en PDF al docente al inicio de la presentación.

| VM | Nombre | Rol | SO | IP | Gateway | Estado |
|----|--------|-----|----|----|---------|:------:|
| VM1 | `vm1-router` | DNS BIND9 + Firewall UFW | Ubuntu 22.04 | `10.73.15.100` | `10.73.15.1` | 🟢 Operativo |
| VM2 | `vm2-editorial` | Node.js + PM2 (API noticias) | Ubuntu 22.04 | `10.73.15.101` | `10.73.15.1` | 🟢 Operativo |
| VM3 | `vm3-cache` | NGINX Proxy Inverso + Fail2ban | Ubuntu 22.04 | `10.73.15.102` | `10.73.15.1` | 🟢 Operativo |
| VM4 | `vm4-admin` | Administración + Monitoreo | Ubuntu 22.04 | `10.73.15.103` | `10.73.15.1` | 🟢 Operativo |

---

## 📓 Bitácora de Avance

> Completar con fechas y nombres reales del grupo.

| # | Fecha | Actividad | Responsable | Dificultad superada |
|---|-------|-----------|-------------|---------------------|
| 1 | __/__/2026 | Creación de las 4 VMs en VirtualBox con adaptador puente. Configuración de IPs estáticas con Netplan en red `10.73.15.0/24` | Todos | Las IPs con adaptador puente requieren que la red del hotspot esté activa antes de arrancar las VMs |
| 2 | __/__/2026 | Instalación y configuración de BIND9 en VM1. Zonas directa e inversa para `noticias.local` con subdominios `www`, `static`, `api`, `admin` | Danner | Los archivos de zona requieren formato exacto; se usó `named-checkzone` para depurar errores de sintaxis |
| 3 | __/__/2026 | Despliegue de API Node.js + PM2 en VM2. Configuración de NGINX como proxy inverso en VM3 con caché para `/api/*` | Limbert / Fer | PM2 no persistía al reiniciar la VM; solución: `pm2 startup systemd` + `pm2 save` |
| 4 | __/__/2026 | Configuración de Fail2ban en VM3 con filtro personalizado para bloquear escaneos de rutas administrativas | Fer | La regex del filtro `nginx-botsearch` requirió ajustarse al formato real de los logs de NGINX |
| 5 | __/__/2026 | Scripts de monitoreo, menú interactivo y backup remoto en VM4. Acceso SSH sin contraseña entre VMs | Melany | `ssh-copy-id` requiere que el servidor de destino ya tenga el puerto 2222 abierto en UFW antes de copiar la clave |

---

## 🗺️ Diagrama de Arquitectura

```
┌──────────────────────────────────────────────────────────────────┐
│            PROYECTO 17 — CDN Simulado de Noticias                │
│     Red: 10.73.15.0/24 (Hotspot) | Adaptador Puente              │
│                  SIS313 USFX | 2026                              │
└──────────────────────────────────────────────────────────────────┘

  Hotspot celular (Gateway: 10.73.15.1)
         │
         ├─────────────────────────────────────────────┐
         │                                             │
         │                                      [Usuarios / Docente]
         │                                             │ HTTP :80
         ▼                                             ▼
┌─────────────────────┐                  ┌─────────────────────────┐
│  VM1 — vm1-router   │                  │   VM3 — vm3-cache       │
│  Danner             │                  │   Fer                   │
│  10.73.15.100       │◄── DNS queries ──│   10.73.15.102          │
│  BIND9 DNS          │                  │   NGINX Proxy Inverso   │
│  UFW Firewall       │                  │   Caché estático        │
│  🟢 Operativo       │                  │   Fail2ban              │
└─────────────────────┘                  │   🟢 Operativo          │
                                         └────────────┬────────────┘
                                                      │ proxy /api/*
                                                      ▼
                                         ┌─────────────────────────┐
                                         │   VM2 — vm2-editorial   │
                                         │   Limbert               │
                                         │   10.73.15.101          │
                                         │   Node.js + PM2         │
                                         │   API REST :3000        │
                                         │   🟢 Operativo          │
                                         └─────────────────────────┘

┌─────────────────────┐
│  VM4 — vm4-admin    │──── SSH :2222 ──► VM1, VM2, VM3
│  Melany             │
│  10.73.15.103       │  Monitor + Backups + Menú admin
│  🟢 Operativo       │
└─────────────────────┘

FLUJO DE DATOS:
  Usuario → VM3:80 → [/static/*]  → NGINX responde directo (caché)
                   → [/api/*]     → proxy → VM2:3000 → responde
  DNS     → VM1:53 → resuelve www / static / api / admin .noticias.local
  Admin   → VM4    → SSH → gestiona y monitorea las demás VMs

LEYENDA:
  🟢 Operativo      = Servicio funcionando y verificado
  🟡 En config.     = Instalado, pendiente de ajustes
  ⚫ Pendiente      = No iniciado aún
```

---

## ⏱️ Cronograma de los 10 Minutos — Guión Exacto

> **Ensayar con cronómetro. El docente para a los 10 minutos exactos.**

---

### 🔵 APERTURA — 30 segundos
**Habla:** Danner (o quien coordine el grupo)

> *"Buenos días Ingeniero. Somos el Grupo [N], presentamos el Proyecto 17: Infraestructura de Noticias con CDN Simulado. Contamos con 4 VMs en red de adaptador puente sobre hotspot, con IPs en el rango 10.73.15.X. VM1 es nuestro servidor DNS, VM2 la API dinámica en Node.js, VM3 el proxy inverso NGINX con caché, y VM4 la administración y monitoreo. Procedemos."*

**Entregar al docente:** Tabla + Bitácora + Diagrama.

---

### 🟢 HITO 1 — Infraestructura Base · 1 minuto
**Habla:** Danner

```bash
# En VM1 — mostrar IP
ip addr show | grep "inet "

# Ping cruzado desde VM1 a las demás
ping -c 2 10.73.15.101   # → VM2 Limbert
ping -c 2 10.73.15.102   # → VM3 Fer
ping -c 2 10.73.15.103   # → VM4 Melany

# SSH activo
sudo systemctl status ssh --no-pager
```

> *"Tenemos 4 VMs operativas con IPs estáticas en la red del hotspot. El ping cruzado confirma conectividad completa entre todas las máquinas."*

---

### 🟡 HITO 2 — Servicios Core · 4 minutos

#### Parte 1: DNS con BIND9 — 2 minutos
**Habla:** Danner

```bash
# Estado de BIND9
sudo systemctl status bind9 --no-pager

# Resolución de todos los subdominios
dig @10.73.15.100 www.noticias.local
dig @10.73.15.100 static.noticias.local
dig @10.73.15.100 api.noticias.local

# Resolución inversa
dig @10.73.15.100 -x 10.73.15.101
```

> *"BIND9 resuelve los tres subdominios del portal hacia VM3 y la zona inversa hacia los nombres de host correspondientes."*

#### Parte 2: NGINX Proxy Inverso + Caché — 2 minutos
**Habla:** Fer (VM3) y Limbert (VM2)

```bash
# VM3: NGINX respondiendo
sudo nginx -t
curl http://10.73.15.102/nginx-health

# VM3: Portal estático
curl -I http://10.73.15.102/

# VM3: Proxy a VM2 — ver cabecera X-Cache-Status
curl -I http://10.73.15.102/api/noticias
curl http://10.73.15.102/api/noticias

# VM3: Contenido estático puro
curl http://10.73.15.102/static/

# VM2: API directa (origen del proxy)
pm2 status
curl http://10.73.15.101:3000/health
```

> *"NGINX en VM3 sirve los estáticos directamente con caché y reenvía las peticiones /api/* a Node.js en VM2. El encabezado X-Cache-Status confirma el comportamiento del caché. Las noticias se cargan dinámicamente desde la API."*

---

### 🔴 HITO 3 — Seguridad y Hardening · 1 minuto 30 segundos
**Habla:** Fer + cualquier integrante

```bash
# SSH endurecido (mostrar en cualquier VM)
grep -E "^(Port|PermitRootLogin|PasswordAuthentication)" /etc/ssh/sshd_config

# Firewall en VM3 (la más expuesta)
sudo ufw status verbose

# Fail2ban activo
sudo fail2ban-client status
sudo fail2ban-client status nginx-botsearch

# Simular escaneo (demostración en vivo)
curl http://10.73.15.102/admin       # → 403
curl http://10.73.15.102/wp-admin    # → 403
sudo fail2ban-client status nginx-botsearch  # → intento registrado

# Permisos de directorios críticos
ls -la /var/www/noticias/    # VM3
ls -la /opt/noticias/        # VM2
```

> *"SSH en puerto 2222 sin acceso root en todas las VMs. UFW con política de denegación por defecto. Fail2ban bloquea escaneos de rutas administrativas. Los directorios tienen permisos 750 con usuarios dedicados."*

---

### 🟣 HITO 4 — Planificación y Defensa · 2 minutos

#### Mostrar diagrama y bitácora (30 seg)
> *"El diagrama muestra el estado actual: 4 VMs operativas con el flujo de datos desde el usuario hasta la API a través del proxy."*

#### Defensa individual — 15 segundos por persona

| Integrante | Guión |
|------------|-------|
| **Danner** | *"Yo configuré VM1: instalé BIND9 con zonas directa e inversa para noticias.local con los subdominios www, static y api. También configuré UFW como firewall."* |
| **Limbert** | *"Yo desplegué VM2: creé la API REST en Node.js con Express, la configuré con PM2 para inicio automático y definí las rutas de noticias."* |
| **Fer** | *"Yo configuré VM3: NGINX como proxy inverso con caché para estáticos y dinámicos, y Fail2ban con filtro personalizado para bloquear escaneos."* |
| **Melany** | *"Yo administré VM4: los scripts de monitoreo a color, el menú interactivo de administración con 9 opciones y el sistema de backups remotos con cron."* |

---

## 📊 Rúbrica y Puntaje Esperado

| Hito | Criterio | Pts | Esperado |
|------|----------|:---:|:--------:|
| **Hito 1** | Infraestructura base | 20 | **20** |
| | VMs levantadas y accesibles (≥60%) | 8 | 8 |
| | Red configurada y ping funcional | 7 | 7 |
| | Tabla de infraestructura entregada | 5 | 5 |
| **Hito 2** | Servicios core | 35 | **34** |
| | DNS BIND9 (T7) operativo | 17 | 17 |
| | NGINX Proxy + Caché (T4) operativo | 17 | 17 |
| | Integración coherente entre ambos | 1 | — |
| **Hito 3** | Seguridad y hardening | 25 | **25** |
| | SSH endurecido | 10 | 10 |
| | Firewall activo y restrictivo | 10 | 10 |
| | Usuarios y permisos diferenciados | 5 | 5 |
| **Hito 4** | Planificación y defensa | 20 | **19** |
| | Diagrama con leyenda de estado | 5 | 5 |
| | Bitácora (3+ entradas) | 5 | 5 |
| | Defensa grupal clara | 10 | 9 |
| **TOTAL** | | **100** | **~98** |

---

## ✅ Checklist del Día del Parcial

### Antes de salir (verificar en cada laptop):

- [ ] VM encendida y visible en VirtualBox
- [ ] IP correcta: `ip addr show | grep inet`
- [ ] Ping a todas las demás VMs responde

**Danner (VM1):**
- [ ] `sudo systemctl status bind9` → active
- [ ] `dig @10.73.15.100 www.noticias.local` → devuelve `10.73.15.102`

**Limbert (VM2):**
- [ ] `pm2 status` → `noticias-api` online
- [ ] `curl http://10.73.15.101:3000/health` → `{"status":"OK"...}`

**Fer (VM3):**
- [ ] `sudo nginx -t` → syntax is ok
- [ ] `curl http://10.73.15.102/api/noticias` → devuelve noticias
- [ ] `sudo fail2ban-client status` → activo

**Melany (VM4):**
- [ ] `/opt/admin/scripts/monitor.sh` → todos en verde
- [ ] Backup remoto ejecutado con éxito

**Todo el grupo:**
- [ ] 3 documentos subidos a eCampus antes de las **09:00**
- [ ] Terminales abiertas en cada VM listas para mostrar
- [ ] Presentación ensayada con cronómetro (10 min exactos)
- [ ] Cada integrante sabe exactamente sus 15 segundos de defensa

---

## 🚨 Plan B — Si Algo Falla en Vivo

| Problema | Solución inmediata |
|----------|--------------------|
| NGINX caído | `sudo systemctl restart nginx` |
| PM2/Node.js caído | `pm2 restart noticias-api` |
| BIND9 caído | `sudo systemctl restart bind9` |
| DNS no resuelve | Usar IPs directas: `curl http://10.73.15.101:3000/health` |
| Fail2ban te baneó | `sudo fail2ban-client set nginx-botsearch unbanip TU_IP` |
| VM apagada | Encender desde VirtualBox — los servicios arrancan solos |
| Hotspot cambió IP | Verificar nueva IP con `ip addr show`, ajustar Netplan |

---

## 📌 Hoja de Referencia Rápida — Comandos del Parcial

```bash
# ── HITO 1: Infraestructura ─────────────────────────────────
ip addr show | grep "inet "
ping -c 2 10.73.15.100 && ping -c 2 10.73.15.101 && ping -c 2 10.73.15.102
sudo systemctl status ssh --no-pager

# ── HITO 2a: DNS ────────────────────────────────────────────
sudo systemctl status bind9 --no-pager
dig @10.73.15.100 www.noticias.local
dig @10.73.15.100 static.noticias.local
dig @10.73.15.100 api.noticias.local
dig @10.73.15.100 -x 10.73.15.101

# ── HITO 2b: NGINX + Node.js ────────────────────────────────
sudo nginx -t
sudo systemctl status nginx --no-pager
curl http://10.73.15.102/nginx-health
curl http://10.73.15.102/api/noticias
curl -I http://10.73.15.102/api/noticias     # ver X-Cache-Status
curl http://10.73.15.102/static/
pm2 status
curl http://10.73.15.101:3000/health
curl http://10.73.15.101:3000/api/noticias

# ── HITO 3: Seguridad ────────────────────────────────────────
grep -E "^(Port|PermitRootLogin|PasswordAuthentication)" /etc/ssh/sshd_config
sudo ufw status verbose
sudo fail2ban-client status
sudo fail2ban-client status nginx-botsearch
curl http://10.73.15.102/admin               # debe dar 403
ls -la /var/www/noticias/
ls -la /opt/noticias/

# ── HITO 4: Documentación ───────────────────────────────────
/opt/admin/scripts/monitor.sh
ls -lh /opt/admin/backups/
crontab -l
```

---

*Proyecto 17 — SIS313 USFX 2026 | Danner · Limbert · Fer · Melany*

