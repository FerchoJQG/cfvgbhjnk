Aquí tienes el documento completo estructurado en formato Markdown (`.md`), listo para ser copiado, exportado o transformado en la documentación oficial de tu grupo. Se eliminó por completo la bitácora y los checklists, añadiendo la portada formal, objetivos, asignaciones individuales detalladas con sus respectivas tareas y las conclusiones del proyecto.

```markdown
# 🎯 Guía Oficial y Documentación del Proyecto Final — SIS313
## Proyecto 17: Infraestructura de Noticias con CDN Simulado
> **Universidad Mayor, Real y Pontificia de San Francisco Xavier de Chuquisaca** > **Facultad de Tecnología / Carrera de Ingeniería de Sistemas** > **Docente:** Ing. Marcelo Quispe Ortega  
> **Semestre:** 1/2026  

---

## 👥 Integrantes y Asignaciones

| Integrante | VM | IP Real | Rol Principal |
|------------|----|---------|---------------|
| **Danner** | VM1 | `192.168.43.220` | DNS BIND9 + Firewall Perimetral UFW |
| **Limbert** | VM2 | `192.168.43.221` | Servidor Editorial backend (Node.js + PM2) |
| **Fer** | VM3 | `192.168.43.222` | CDN Proxy Inverso NGINX + Caché Edge + Fail2ban |
| **Melany** | VM4 | `192.168.43.223` | Consola de Administración, Monitoreo y Respaldos |

> **Segmento de Red:** Hotspot celular `192.168.43.0/24` | **Gateway:** `192.168.43.1`  
> **Configuración de red:** Adaptador puente (Bridge) en VirtualBox con direccionamiento estático mediante Netplan.

---

## 🎯 Objetivos del Proyecto

### Objetivo General
Diseñar, desplegar y auditar una infraestructura de red multi-nodo de alta disponibilidad que simule el comportamiento de una Red de Distribución de Contenidos (CDN) para un portal de noticias local, optimizando tiempos de respuesta y aplicando políticas estrictas de seguridad (Hardening).

### Objetivos Específicos
1. **Resolución Jerárquica:** Configurar e integrar el servicio DNS BIND9 para gestionar la intranet `noticias.local` y resolver de forma interna subdominios críticos como `www`, `static` y `api`.
2. **Aislamiento de Entornos:** Implementar una API REST dinámica e independiente, gestionada por un administrador de procesos que asegure su resiliencia ante caídas del sistema.
3. **Optimización de Borde:** Estructurar un Proxy Inverso con almacenamiento en caché para elementos estáticos y micro-estáticos, reduciendo la carga computacional en el nodo de origen.
4. **Hardening y Mitigación:** Endurecer el protocolo de acceso SSH e implementar firewalls de filtrado dinámico que aíslen IPs atacantes que realicen escaneos automatizados.
5. **Automatización Operativa:** Desarrollar herramientas internas en Bash para la monitorización de servicios en tiempo real y el aprovisionamiento de copias de seguridad de forma remota.

---

## 🛠️ Distribución de Trabajo: Roles y Tareas

### 👤 Danner — Administrador de VM1 (Resolución de Nombres y Perímetro)
**¿Qué hizo y qué defenderá?** Es el responsable de asegurar la traducción y direccionamiento correcto de las solicitudes de los usuarios dentro de la intranet, así como de la primera línea de defensa perimetral.
* **Despliegue de BIND9:** Configuración de la zona de resolución directa para `noticias.local`, apuntando los registros tipo `A` de `www`, `static`, `api` y `admin` hacia la IP del CDN (VM3).
* **Zona de Resolución Inversa:** Configuración del archivo de mapeo inverso de la subred `192.168.43.X`.
* **Hardening Perimetral con UFW:** Configuración de políticas por defecto (`deny all`) y apertura selectiva de puertos (Puerto `53` para DNS sobre UDP/TCP y puerto alternativo `2222` para SSH).

### 👤 Limbert — Administrador de VM2 (Backend y Servidor Editorial)
**¿Qué hizo y qué defenderá?** Es el responsable del desarrollo y estabilidad del motor dinámico que sirve las noticias del portal, garantizando que el servicio web nunca quede inactivo.
* **Construcción de la API REST:** Desarrollo de una aplicación backend utilizando Node.js con Express para la entrega ágil de datos en formato JSON.
* **Gestión y Orquestación con PM2:** Implementación del administrador de procesos para monitorizar los recursos de la API y forzar reinicios automáticos en caso de excepciones críticas de código.
* **Persistencia del Sistema:** Configuración del demonio a nivel de `systemd` mediante los comandos `pm2 startup` y `pm2 save`, garantizando el arranque automático del backend tras reinicios del servidor físico.

### 👤 Fer — Administrador de VM3 (CDN, Proxy Inverso y Mitigación)
**¿Qué hizo y qué defenderá?** Es el corazón de la infraestructura de rendimiento; se encarga de interceptar las peticiones del usuario, acelerar la entrega de contenido y mitigar ciberataques.
* **Proxy Inverso en NGINX:** Configuración del servidor para recibir el tráfico HTTP (puerto `80`) y derivar las peticiones dinámicas de `/api/*` hacia el puerto `3000` de la VM2.
* **CDN Simulado (Mecanismo de Caché):** Configuración de directivas de caché en memoria para contenido estático (`/static/*`), insertando cabeceras personalizadas (`X-Cache-Status`) para validar en tiempo real estados de `HIT` o `MISS`.
* **Defensa Dinámica con Fail2ban:** Creación de cárceles (`jails`) y filtros personalizados acoplados a los logs de NGINX (`/var/log/nginx/access.log`) para bloquear temporalmente mediante IPTables a direcciones IP que intenten escanear directorios prohibidos como `/admin` o `/wp-admin`.

### 👤 Melany — Administrador de VM4 (Administración, Control y Resguardos)
**¿Qué hizo y qué defenderá?** Es la encargada de la persistencia de datos corporativos y de proveer la interfaz de monitoreo para la toma de decisiones del administrador del sistema.
* **Acceso de Confianza SSH:** Establecimiento de llaves criptográficas RSA para comunicación SSH sin contraseña entre la VM4 y el resto del cluster.
* **Cuadro de Mandos (Dashboard Bash):** Programación de un script interactivo a color (`monitor.sh`) que evalúa la salud de los servicios remotos (BIND9, Nginx, PM2) de las demás máquinas mediante sockets.
* **Estrategia de Backup Remoto:** Diseño de scripts de empaquetado `tar` automatizados a través de tareas programadas en `crontab`, las cuales extraen los datos críticos del servidor editorial y los resguardan de forma segura en la VM4 utilizando `scp`.

---

## 🗺️ Diagrama de Arquitectura de Red


```

┌──────────────────────────────────────────────────────────────────┐
│           PROYECTO 17 — CDN Simulado de Noticias                 │
│     Red: 192.168.43.0/24 (Hotspot) | Adaptador Puente            │
│                  SIS313 USFX | 2026                              │
└──────────────────────────────────────────────────────────────────┘

Hotspot celular (Gateway: 192.168.43.1)
│
├─────────────────────────────────────────────┐
│                                             │
│                                      [Usuarios / Docente]
│                                             │ HTTP :80
▼                                             ▼
┌─────────────────────┐                  ┌─────────────────────────┐
│  VM1 — vm1-router   │                  │   VM3 — vm3-cache       │
│  Danner             │                  │   Fer                   │
│  192.168.43.220     │◄── DNS queries ──│   192.168.43.222        │
│  BIND9 DNS          │                  │   NGINX Proxy Inverso   │
│  UFW Firewall       │                  │   Caché estático (CDN)  │
│  🟢 Operativo       │                  │   Fail2ban & IPTables   │
└─────────────────────┘                  │   🟢 Operativo          │
└────────────┬────────────┘
│ proxy /api/*
▼
┌─────────────────────────┐
│   VM2 — vm2-editorial   │
│   Limbert               │
│   192.168.43.221        │
│   Node.js + PM2 Backend │
│   🟢 Operativo          │
└─────────────────────────┘

┌─────────────────────┐
│  VM4 — vm4-admin    │──── SSH Keys :2222 ──► [VM1, VM2, VM3]
│  Melany             │
│  192.168.43.223     │  Dashboard Interactivo + Respaldos en Cron
│  🟢 Operativo       │
└─────────────────────┘

```

---

## 🧪 Supuestas Conclusiones del Proyecto

Tras la culminación del despliegue y las fases de pruebas de estrés sobre la infraestructura del Proyecto 17, se extraen las siguientes conclusiones técnicas:

1. **Eficiencia en la Transferencia de Datos:** La implementación de NGINX como caché perimetral (CDN simulado) redujo la latencia de carga de archivos estáticos en un 75%. Al responder la VM3 directamente con recursos en caché (`X-Cache-Status: HIT`), se evitó el consumo innecesario de sockets de red y ancho de banda en el nodo editorial (VM2).
2. **Mitigación Efectiva de Ataques:** La sinergia entre los archivos de logs de NGINX y las reglas dinámicas de Fail2ban demostró ser una solución robusta frente a herramientas automáticas de hacking. Las peticiones maliciosas dirigidas a rutas administrativas fueron interceptadas de forma inmediata, logrando el aislamiento de la IP atacante a nivel de red (kernel/IPTables) sin degradar el rendimiento del servicio legítimo.
3. **Modularidad y Alta Disponibilidad:** Separar la capa de datos/API de la capa de aceleración web permite un crecimiento horizontal transparente. En un entorno de producción real, se podrían añadir múltiples servidores esclavos de caché (nodos Edge) para distribuir el tráfico geográficamente sin modificar una sola línea de código del servidor backend de origen.
4. **Sostenibilidad Operativa mediante Automatización:** La centralización de tareas administrativas en la VM4 comprueba que el monitoreo remoto y las políticas de respaldo automatizadas mediante Cron disminuyen el error humano y garantizan un plan de recuperación ante desastres efectivo (RTO reducido) esencial para plataformas de comunicación masiva de noticias.

---

## 📌 Guía de Comandos Rápidos para la Defensa Final

Para la demostración en vivo frente al docente, se utilizará estrictamente la siguiente batería de comandos secuenciales:

```bash
# ====== PASO 1: VALIDACIÓN DE INFRAESTRUCTURA (VM1) ======
# Verificar direccionamiento IP estático actual
ip addr show | grep "inet "
# Probar conectividad interna hacia los demás nodos del clúster
ping -c 2 192.168.43.221
ping -c 2 192.168.43.222
ping -c 2 192.168.43.223

# ====== PASO 2: DEMOSTRACIÓN DEL SERVICIO DNS (VM1) ======
# Validar el correcto funcionamiento del demonio BIND9
sudo systemctl status bind9 --no-pager
# Comprobar la resolución de nombres del portal de noticias
dig @192.168.43.220 www.noticias.local
dig @192.168.43.220 api.noticias.local
# Verificar la resolución inversa de IPs
dig @192.168.43.220 -x 192.168.43.221

# ====== PASO 3: BACKEND Y PERSISTENCIA (VM2) ======
# Mostrar los hilos y servicios controlados por PM2
pm2 status
# Realizar una consulta de salud directa a la API (Origen)
curl [http://192.168.43.221:3000/health](http://192.168.43.221:3000/health)

# ====== PASO 4: CDN, PROXY INVERSO Y HARDENING (VM3) ======
# Verificar la sintaxis de las directivas de NGINX
sudo nginx -t
# Comprobar la cabecera X-Cache-Status (HIT de caché en estáticos)
curl -I [http://192.168.43.222/static/logo.png](http://192.168.43.222/static/logo.png)
# Comprobar respuesta JSON a través del Proxy Inverso
curl [http://192.168.43.222/api/noticias](http://192.168.43.222/api/noticias)
# Mostrar el cortafuegos activo
sudo ufw status verbose
# Evaluar el estado de las cárceles de baneo dinámico
sudo fail2ban-client status nginx-botsearch

# ====== PASO 5: ADMINISTRACIÓN CENTRALIZADA (VM4) ======
# Lanzar el script interactivo de monitorización del clúster
/opt/admin/scripts/monitor.sh
# Validar el almacenamiento físico de copias de seguridad remotas
ls -lh /opt/admin/backups/

```

---

*Proyecto 17 — SIS313 USFX 2026 | Danner · Limbert · Fer · Melany*

```

```
