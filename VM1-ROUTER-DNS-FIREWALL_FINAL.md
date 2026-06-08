# 🖥️ VM1 — DNS / Firewall
### Proyecto 17: Infraestructura de Noticias con CDN Simulado
> **SIS313 — USFX Chuquisaca | Semestre 1/2026**
> **Responsable:** Danner

---

## 📋 Ficha Técnica

| Campo | Detalle |
|-------|---------|
| **Nombre de VM** | `vm1-router` |
| **Responsable** | Danner |
| **Rol** | DNS BIND9 + Firewall UFW |
| **Sistema Operativo** | Ubuntu Server 22.04 LTS |
| **RAM** | 1024 MB |
| **CPU** | 1 vCPU |
| **Disco** | 20 GB |
| **Adaptador de red** | Adaptador puente (Bridge) |
| **IP** | `10.73.15.100/24` |
| **Gateway** | `10.73.15.1` (hotspot celular) |
| **Estado** | 🟢 Operativo |

> ⚠️ **Importante:** Todas las VMs usan **Adaptador Puente (Bridge)** en VirtualBox. Esto hace que cada VM aparezca como una computadora más dentro de la red del hotspot. No se usan VLANs ni redes internas.

---

## 🗺️ Rol en la Arquitectura

```
Hotspot celular — 10.73.15.0/24
         │
   [VM1 · Danner · 10.73.15.100]
   DNS BIND9 + Firewall UFW
         │
         ├── Resuelve www.noticias.local     → 10.73.15.102 (VM3)
         ├── Resuelve static.noticias.local  → 10.73.15.102 (VM3)
         ├── Resuelve api.noticias.local     → 10.73.15.102 (VM3)
         └── Resuelve admin.noticias.local   → 10.73.15.103 (VM4)
```

---

## ⚙️ PASO 1 — Configuración Inicial

### 1.1 Actualizar el sistema
Siempre actualizamos primero para tener los paquetes más recientes y evitar errores de dependencias.
```bash
sudo apt update && sudo apt upgrade -y
```
Luego instalamos herramientas básicas que usaremos durante toda la configuración:
```bash
sudo apt install -y net-tools curl wget nano git
```
- `net-tools` → para ver IPs con `ifconfig`
- `curl` → para probar URLs
- `nano` → editor de texto para editar archivos de configuración

### 1.2 Configurar el hostname
El hostname identifica esta máquina en la red. Lo cambiamos a `vm1-router`:
```bash
sudo hostnamectl set-hostname vm1-router
```
Verificamos que quedó bien:
```bash
hostnamectl
```

### 1.3 Configurar IP estática con Netplan

Por defecto Ubuntu usa DHCP (el router asigna IP automáticamente). Necesitamos una IP fija para que el DNS siempre tenga la misma dirección. Abrimos el archivo de configuración de red:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

> 💡 **Cómo usar nano:** Las flechas del teclado mueven el cursor. Cuando termines de editar presiona `Ctrl+O` para guardar, luego `Enter` para confirmar el nombre, y `Ctrl+X` para salir.

Borramos todo lo que haya en el archivo y escribimos exactamente esto (respetando la indentación con espacios, **NO con tabulaciones**):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 10.73.15.100/24
      routes:
        - to: default
          via: 10.73.15.1
      nameservers:
        addresses:
          - 10.73.15.100
        search:
          - noticias.local
```

> ⚠️ `enp0s3` es el nombre de la interfaz de red en VirtualBox. Si tu interfaz tiene otro nombre, cámbialo. Puedes verlo con: `ip link show`

Guardamos (`Ctrl+O` → `Enter` → `Ctrl+X`) y aplicamos la configuración:
```bash
sudo netplan apply
```

### 1.4 Verificar conectividad
Comprobamos que la IP quedó bien asignada:
```bash
ip addr show
```
Buscamos en la salida: `inet 10.73.15.100/24`

Hacemos ping al gateway (hotspot) para confirmar que hay red:
```bash
ping -c 3 10.73.15.1
```
Hacemos ping a las otras VMs (deben estar encendidas):
```bash
ping -c 3 10.73.15.101   # VM2-Editorial
ping -c 3 10.73.15.102   # VM3-Cache
ping -c 3 10.73.15.103   # VM4-Admin
```
Si algún ping falla, verificar que esa VM esté encendida y con IP correcta.

---

## 🌍 PASO 2 — Instalación y Configuración de DNS (BIND9)

BIND9 es el servidor DNS que va a resolver los nombres `*.noticias.local` dentro de la red del proyecto.

### 2.1 Instalar BIND9
```bash
sudo apt install -y bind9 bind9utils bind9-doc dnsutils
```
- `bind9` → el servidor DNS principal
- `bind9utils` → herramientas como `named-checkconf` para verificar configuración
- `dnsutils` → herramientas como `dig` y `nslookup` para probar el DNS

Habilitamos BIND9 para que arranque automáticamente con el sistema:
```bash
sudo systemctl enable bind9
sudo systemctl start bind9
sudo systemctl status bind9
```
La salida debe mostrar `active (running)` en verde.

### 2.2 Configurar opciones globales de BIND9

Abrimos el archivo principal de opciones:
```bash
sudo nano /etc/bind/named.conf.options
```

Reemplazamos todo el contenido con esto:

```
options {
    directory "/var/cache/bind";

    // Escuchar solo en la IP de esta VM y en localhost
    listen-on { 127.0.0.1; 10.73.15.100; };

    // Permitir consultas desde cualquier IP de la red
    allow-query { any; };

    // Si no conoce el dominio, reenviar al DNS de Google
    forwarders { 8.8.8.8; 8.8.4.4; };
    forward only;

    recursion yes;
    dnssec-validation no;
};
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

### 2.3 Configurar las zonas DNS

Las zonas definen qué dominios maneja este servidor. Abrimos el archivo:
```bash
sudo nano /etc/bind/named.conf.local
```

Agregamos las dos zonas (zona directa e inversa):

```
// Zona directa: resuelve nombres → IPs
zone "noticias.local" {
    type master;
    file "/etc/bind/zones/db.noticias.local";
};

// Zona inversa: resuelve IPs → nombres
zone "15.73.10.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.10.73.15";
};
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

### 2.4 Crear el directorio para los archivos de zona
```bash
sudo mkdir -p /etc/bind/zones
```

### 2.5 Crear la zona directa (nombres → IPs)

Este archivo dice a qué IP corresponde cada nombre del dominio `noticias.local`:
```bash
sudo nano /etc/bind/zones/db.noticias.local
```

Escribimos exactamente esto:

```
$TTL    604800
@   IN  SOA vm1-router.noticias.local. admin.noticias.local. (
            2026060801  ; Serial — cambiar este número si modificas el archivo
            604800      ; Refresh — cada cuánto los esclavos actualizan (7 días)
            86400       ; Retry   — si falla, reintentar cada (1 día)
            2419200     ; Expire  — cuánto tiempo sigue siendo válido (28 días)
            604800 )    ; Negative TTL — cuánto cachear respuestas "no existe"

; Servidor de nombres de esta zona
@           IN  NS  vm1-router.noticias.local.

; Registros A — máquina (hostname) → IP
vm1-router    IN  A   10.73.15.100
vm2-editorial IN  A   10.73.15.101
vm3-cache     IN  A   10.73.15.102
vm4-admin     IN  A   10.73.15.103

; Subdominios del portal — todos apuntan a VM3 (NGINX)
; excepto admin que apunta a VM4
www           IN  A   10.73.15.102
static        IN  A   10.73.15.102
api           IN  A   10.73.15.102
admin         IN  A   10.73.15.103
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

### 2.6 Crear la zona inversa (IPs → nombres)

Este archivo hace lo contrario: dado una IP, devuelve el nombre:
```bash
sudo nano /etc/bind/zones/db.10.73.15
```

```
$TTL    604800
@   IN  SOA vm1-router.noticias.local. admin.noticias.local. (
            2026060801
            604800
            86400
            2419200
            604800 )

@   IN  NS  vm1-router.noticias.local.

; Resolución inversa: último octeto de la IP → nombre
100  IN  PTR  vm1-router.noticias.local.
101  IN  PTR  vm2-editorial.noticias.local.
102  IN  PTR  vm3-cache.noticias.local.
103  IN  PTR  vm4-admin.noticias.local.
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

### 2.7 Verificar y reiniciar BIND9

Verificamos que no hay errores de sintaxis en los archivos:
```bash
# Verificar configuración general
sudo named-checkconf

# Verificar zona directa
sudo named-checkzone noticias.local /etc/bind/zones/db.noticias.local

# Verificar zona inversa
sudo named-checkzone 15.73.10.in-addr.arpa /etc/bind/zones/db.10.73.15
```
Si alguno muestra errores, volver a abrir ese archivo con `sudo nano` y corregirlos.

Si todo está OK, reiniciamos BIND9:
```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

### 2.8 Probar que el DNS funciona

Probamos resolución directa:
```bash
dig @10.73.15.100 www.noticias.local
dig @10.73.15.100 api.noticias.local
dig @10.73.15.100 static.noticias.local
dig @10.73.15.100 admin.noticias.local
```
En la sección `ANSWER SECTION` debe aparecer la IP correspondiente.

Probamos resolución inversa:
```bash
dig @10.73.15.100 -x 10.73.15.101
```
Debe aparecer `vm2-editorial.noticias.local.`

---

## 🔥 PASO 3 — Configuración del Firewall (UFW)

UFW (Uncomplicated Firewall) controla qué tráfico entra y sale de esta VM. Lo configuramos para bloquear todo excepto lo necesario.

### 3.1 Instalar UFW
```bash
sudo apt install -y ufw
```

### 3.2 Definir políticas por defecto
Bloqueamos todo el tráfico entrante y permitimos todo el saliente:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### 3.3 Abrir los puertos necesarios

Puerto SSH (usamos 2222 en lugar del 22 estándar por seguridad):
```bash
sudo ufw allow 2222/tcp comment 'SSH endurecido'
```

Puerto DNS (necesario para que las otras VMs puedan hacer consultas DNS):
```bash
sudo ufw allow 53/tcp comment 'DNS TCP'
sudo ufw allow 53/udp comment 'DNS UDP'
```
> 💡 El DNS usa principalmente UDP, pero también TCP para respuestas grandes.

### 3.4 Activar el firewall
```bash
sudo ufw enable
```
Escribe `y` cuando pregunte si continuar.

Verificamos las reglas activas:
```bash
sudo ufw status verbose
```

---

## 🔐 PASO 4 — Hardening de SSH

El puerto por defecto del SSH es el 22, que es el primero que atacan los bots en internet. Lo cambiamos al 2222 y deshabilitamos el login con contraseña.

### 4.1 Editar configuración de SSH
```bash
sudo nano /etc/ssh/sshd_config
```

Buscamos estas líneas en el archivo (usa `Ctrl+W` para buscar en nano) y las modificamos:

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
```

> 💡 Para buscar en nano: `Ctrl+W` → escribe el texto → `Enter`

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

### 4.2 Reiniciar SSH y verificar
```bash
sudo systemctl restart ssh
sudo systemctl status ssh
```

### 4.3 Generar clave SSH (ejecutar en la laptop anfitriona, NO en la VM)
```bash
ssh-keygen -t ed25519 -C "sis313-vm1"
ssh-copy-id -p 2222 TU_USUARIO@10.73.15.100
```
Reemplaza `TU_USUARIO` por el usuario de Ubuntu de esta VM.

> ⚠️ **Importante:** Antes de cerrar la sesión SSH actual, abre OTRA terminal y prueba conectarte con la clave. Si funciona, recién cierra la sesión anterior.

---

## 👤 PASO 5 — Usuarios y Permisos

Creamos un usuario dedicado para administrar DNS, sin usar root directamente:
```bash
sudo useradd -m -s /bin/bash dns-admin
sudo usermod -aG sudo dns-admin
```

Asignamos permisos restrictivos a los archivos de zona DNS:
```bash
# El usuario bind (que corre BIND9) es el dueño
sudo chown -R bind:bind /etc/bind/zones/

# Solo el dueño puede leer/escribir; grupo solo leer; otros nada
sudo chmod -R 640 /etc/bind/zones/

# Verificamos
ls -la /etc/bind/zones/
```

---

## ✅ PASO 6 — Verificaciones para el Parcial

Ejecuta estos comandos uno por uno y muestra los resultados:

```bash
# 1. Confirmar IP correcta
ip addr show | grep "inet "
# Debe mostrar: inet 10.73.15.100/24

# 2. Ping a todas las VMs
ping -c 2 10.73.15.101
ping -c 2 10.73.15.102
ping -c 2 10.73.15.103

# 3. BIND9 activo
sudo systemctl status bind9 --no-pager
# Debe mostrar: active (running)

# 4. Resolución DNS de subdominios
dig @10.73.15.100 www.noticias.local +short
# Debe devolver: 10.73.15.102
dig @10.73.15.100 static.noticias.local +short
dig @10.73.15.100 api.noticias.local +short
dig @10.73.15.100 admin.noticias.local +short
# Debe devolver: 10.73.15.103

# 5. Resolución inversa
dig @10.73.15.100 -x 10.73.15.101 +short
# Debe devolver: vm2-editorial.noticias.local.

# 6. Verificar que SSH cambió al puerto 2222
grep -E "^Port" /etc/ssh/sshd_config

# 7. Verificar hardening SSH completo
grep -E "^(Port|PermitRootLogin|PasswordAuthentication)" /etc/ssh/sshd_config

# 8. Estado del firewall
sudo ufw status verbose

# 9. Permisos de zonas DNS
ls -la /etc/bind/zones/
```

---

## 🚨 Comandos de Emergencia

```bash
# BIND9 no arranca — ver el error exacto
sudo journalctl -u bind9 -n 30
# Verificar sintaxis de los archivos
sudo named-checkconf
sudo named-checkzone noticias.local /etc/bind/zones/db.noticias.local

# Si el DNS no responde pero BIND9 está corriendo
# Verificar que el firewall tiene abierto el puerto 53
sudo ufw status verbose | grep 53

# Reiniciar la red si perdiste conectividad
sudo netplan apply

# Ver logs del firewall
sudo tail -f /var/log/ufw.log

# Si te quedaste sin acceso SSH (bloqueaste tu propia IP)
# Accede directo por consola VirtualBox y ejecuta:
sudo ufw disable
# Luego corrige las reglas y vuelve a habilitar
```

---

*Proyecto 17 — SIS313 USFX 2026 | VM1 · Danner · 10.73.15.100*
