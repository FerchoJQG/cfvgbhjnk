# 🖥️ VM4 — Administración / Monitoreo
### Proyecto 17: Infraestructura de Noticias con CDN Simulado
> **SIS313 — USFX Chuquisaca | Semestre 1/2026**
> **Responsable:** (VM4-Admin)

---

## 📋 Ficha Técnica

| Campo | Detalle |
|-------|---------|
| **Nombre de VM** | `vm4-admin` |
| **Rol** | Servidor de administración + Monitoreo + Backups remotos |
| **Sistema Operativo** | Ubuntu Server 22.04 LTS |
| **RAM** | 1024 MB |
| **CPU** | 1 vCPU |
| **Disco** | 25 GB |
| **Adaptador de red** | Adaptador puente (Bridge) |
| **IP** | `10.73.15.103/24` |
| **Gateway** | `10.73.15.1` (hotspot celular) |
| **DNS** | `10.73.15.100` (VM1-BIND9) |
| **Estado** | 🟢 Operativo |

> ⚠️ **Importante:** Todas las VMs usan **Adaptador Puente (Bridge)** en VirtualBox. Cada VM aparece como una computadora más dentro de la red del hotspot. No se usan VLANs ni redes internas.

---

## 🗺️ Rol en la Arquitectura

```
Hotspot celular — 10.73.15.0/24
         │
   [VM4-Admin · 10.73.15.103]
   ├── Script de monitoreo de toda la infraestructura
   ├── Backup remoto de VM2 (editorial)
   ├── Menú interactivo de administración
   └── Health checks automatizados (cron)

   Solo esta VM administra SSH a las demás
```

Esta VM actúa como el **panel de control** de toda la infraestructura. Desde aquí el administrador monitorea el estado de todos los servicios, ejecuta backups remotos y gestiona la red.

---

## ⚙️ PASO 1 — Configuración Inicial del Sistema

### 1.1 Actualizar el sistema
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget nano git net-tools nmap dnsutils bc
```
- `nmap` → para escanear puertos
- `dnsutils` → para usar `dig` y probar el DNS
- `bc` → calculadora para scripts
- `nano` → editor de texto

### 1.2 Configurar hostname
```bash
sudo hostnamectl set-hostname vm4-admin
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
        - 10.73.15.103/24
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

### 1.4 Verificar conectividad completa
```bash
ip addr show | grep "inet "
# Debe mostrar: inet 10.73.15.103/24

ping -c 3 10.73.15.1    # Gateway (hotspot)
ping -c 3 10.73.15.100  # VM1-Router
ping -c 3 10.73.15.101  # VM2-Editorial
ping -c 3 10.73.15.102  # VM3-Cache

dig www.noticias.local @10.73.15.100
dig admin.noticias.local @10.73.15.100
# Debe devolver 10.73.15.103 para admin
```

---

## 🔑 PASO 2 — Configurar Acceso SSH sin Contraseña a las Demás VMs

Esta VM necesita acceso SSH a las otras para ejecutar monitoreo y backups remotos.

### 2.1 Generar par de claves
```bash
ssh-keygen -t ed25519 -C "vm4-admin-sis313" -f ~/.ssh/id_ed25519 -N ""
```

### 2.2 Copiar clave pública a cada VM
Reemplaza `usuario` por el nombre de usuario de Ubuntu en cada VM:
```bash
# A VM1-Router
ssh-copy-id -p 2222 usuario@10.73.15.100

# A VM2-Editorial
ssh-copy-id -p 2222 usuario@10.73.15.101

# A VM3-Cache
ssh-copy-id -p 2222 usuario@10.73.15.102
```
> 💡 Si las otras VMs aún tienen PasswordAuthentication activo durante la configuración inicial, esto funcionará. Después de copiar la clave, las otras VMs la desactivan.

### 2.3 Probar acceso sin contraseña
```bash
ssh -p 2222 usuario@10.73.15.101 "hostname && ip addr | grep inet"
ssh -p 2222 usuario@10.73.15.102 "nginx -v && systemctl is-active nginx"
```

---

## 📊 PASO 3 — Script de Monitoreo de Infraestructura

### 3.1 Crear directorio de scripts
```bash
sudo mkdir -p /opt/admin/{scripts,logs,backups,reportes}
sudo chown -R $USER:$USER /opt/admin
```

### 3.2 Crear el script principal de monitoreo
```bash
nano /opt/admin/scripts/monitor_infraestructura.sh
```

Escribimos el siguiente contenido:

```bash
#!/bin/bash
# ═══════════════════════════════════════════════════════════════
#  MONITOR DE INFRAESTRUCTURA — Proyecto 17 SIS313 USFX
#  VM4-Admin | Verifica todos los servicios de la red
# ═══════════════════════════════════════════════════════════════

# ─── Colores ─────────────────────────────────────────────────
VERDE='\033[0;32m'
ROJO='\033[0;31m'
AMARILLO='\033[1;33m'
AZUL='\033[0;34m'
CYAN='\033[0;36m'
BOLD='\033[1m'
RESET='\033[0m'

LOG="/opt/admin/logs/monitor_$(date +%Y%m%d).log"
ERRORES=0

# ─── Función de estado ────────────────────────────────────────
check() {
    local nombre="$1"
    local cmd="$2"
    printf "  %-35s" "$nombre"
    if eval "$cmd" > /dev/null 2>&1; then
        echo -e "${VERDE}✅ OK${RESET}"
        echo "[$(date +%H:%M:%S)] OK    - $nombre" >> "$LOG"
    else
        echo -e "${ROJO}❌ FALLA${RESET}"
        echo "[$(date +%H:%M:%S)] ERROR - $nombre" >> "$LOG"
        ((ERRORES++))
    fi
}

# ─── Cabecera ────────────────────────────────────────────────
clear
echo -e "${BOLD}${AZUL}"
echo "  ╔══════════════════════════════════════════════════════╗"
echo "  ║    MONITOR DE INFRAESTRUCTURA — SIS313 USFX 2026    ║"
echo "  ║          Proyecto 17: CDN Simulado de Noticias       ║"
echo "  ╚══════════════════════════════════════════════════════╝"
echo -e "${RESET}"
echo -e "  ${CYAN}Fecha/Hora:${RESET} $(date '+%d/%m/%Y %H:%M:%S')"
echo -e "  ${CYAN}Monitor desde:${RESET} vm4-admin (10.73.15.103)"
echo ""

# ─── VM1: Router / DNS / Firewall ────────────────────────────
echo -e "${BOLD}${AMARILLO}━━━ VM1 — Router / DNS / Firewall (10.73.15.100) ━━━${RESET}"
check "Ping a VM1-Router"          "ping -c 1 -W 2 10.73.15.100"
check "SSH activo en VM1"          "nc -z -w 3 10.73.15.100 2222"
check "DNS: www.noticias.local"    "dig @10.73.15.100 www.noticias.local +short | grep -q '.'"
check "DNS: static.noticias.local" "dig @10.73.15.100 static.noticias.local +short | grep -q '.'"
check "DNS: api.noticias.local"    "dig @10.73.15.100 api.noticias.local +short | grep -q '.'"
check "DNS: admin.noticias.local"  "dig @10.73.15.100 admin.noticias.local +short | grep -q '.'"
check "DNS: resolución inversa"    "dig @10.73.15.100 -x 10.73.15.101 +short | grep -q '.'"
echo ""

# ─── VM2: Servidor Editorial ──────────────────────────────────
echo -e "${BOLD}${AMARILLO}━━━ VM2 — Servidor Editorial Node.js (10.73.15.101) ━━━${RESET}"
check "Ping a VM2-Editorial"       "ping -c 1 -W 2 10.73.15.101"
check "SSH activo en VM2"          "nc -z -w 3 10.73.15.101 2222"
check "API Node.js puerto 3000"    "nc -z -w 3 10.73.15.101 3000"
check "Health check API"           "curl -sf http://10.73.15.101:3000/health | grep -q OK"
check "Ruta /api/noticias OK"      "curl -sf http://10.73.15.101:3000/api/noticias | grep -q noticias"
echo ""

# ─── VM3: Cache / NGINX / Fail2ban ──────────────────────────
echo -e "${BOLD}${AMARILLO}━━━ VM3 — Cache NGINX + Fail2ban (10.73.15.102) ━━━${RESET}"
check "Ping a VM3-Cache"           "ping -c 1 -W 2 10.73.15.102"
check "SSH activo en VM3"          "nc -z -w 3 10.73.15.102 2222"
check "NGINX puerto 80"            "nc -z -w 3 10.73.15.102 80"
check "Portal principal"           "curl -sf http://10.73.15.102/ | grep -qi noticias"
check "Proxy /api/noticias"        "curl -sf http://10.73.15.102/api/noticias | grep -q noticias"
check "Contenido /static/"         "curl -sf http://10.73.15.102/static/ | grep -qi estatico"
check "NGINX health endpoint"      "curl -sf http://10.73.15.102/nginx-health | grep -q OK"
echo ""

# ─── Resumen ─────────────────────────────────────────────────
echo -e "${BOLD}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${RESET}"
if [ "$ERRORES" -eq 0 ]; then
    echo -e "  ${VERDE}${BOLD}✅ TODOS LOS SERVICIOS OPERATIVOS${RESET}"
    echo -e "  ${CYAN}Infraestructura 100% funcional${RESET}"
else
    echo -e "  ${ROJO}${BOLD}⚠️  $ERRORES SERVICIO(S) CON PROBLEMAS${RESET}"
    echo -e "  ${AMARILLO}Revisar log: $LOG${RESET}"
fi
echo -e "  Log guardado en: ${CYAN}$LOG${RESET}"
echo ""
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

```bash
chmod +x /opt/admin/scripts/monitor_infraestructura.sh
```

### 3.3 Ejecutar el monitor
```bash
/opt/admin/scripts/monitor_infraestructura.sh
```

---

## 🖥️ PASO 4 — Menú Interactivo de Administración

```bash
nano /opt/admin/scripts/menu_admin.sh
```

```bash
#!/bin/bash
# ═══════════════════════════════════════════════════════════════
#  MENÚ DE ADMINISTRACIÓN — Proyecto 17 SIS313 USFX
# ═══════════════════════════════════════════════════════════════

VERDE='\033[0;32m'; ROJO='\033[0;31m'; CYAN='\033[0;36m'
AZUL='\033[1;34m';  BOLD='\033[1m';    RESET='\033[0m'

mostrar_menu() {
    clear
    echo -e "${BOLD}${AZUL}"
    echo "  ╔══════════════════════════════════════════════════╗"
    echo "  ║   ADMINISTRACIÓN — Noticias CDN · SIS313 USFX   ║"
    echo "  ╚══════════════════════════════════════════════════╝"
    echo -e "${RESET}"
    echo -e "  ${CYAN}[1]${RESET}  Monitorear toda la infraestructura"
    echo -e "  ${CYAN}[2]${RESET}  Ver estado de servicios (resumen rápido)"
    echo -e "  ${CYAN}[3]${RESET}  Ejecutar backup de contenido editorial"
    echo -e "  ${CYAN}[4]${RESET}  Ver logs de NGINX en VM3"
    echo -e "  ${CYAN}[5]${RESET}  Ver IPs bloqueadas por Fail2ban"
    echo -e "  ${CYAN}[6]${RESET}  Consultar API de noticias"
    echo -e "  ${CYAN}[7]${RESET}  Verificar DNS (todos los subdominios)"
    echo -e "  ${CYAN}[8]${RESET}  Ver últimos backups"
    echo -e "  ${CYAN}[9]${RESET}  Conectar SSH a una VM"
    echo -e "  ${ROJO}[0]${RESET}  Salir"
    echo ""
    echo -n "  Seleccione una opción: "
}

opcion_1() {
    echo -e "${CYAN}Ejecutando monitor...${RESET}"
    /opt/admin/scripts/monitor_infraestructura.sh
    read -p "  Presione ENTER para continuar..."
}

opcion_2() {
    echo -e "${CYAN}Resumen rápido de servicios:${RESET}"
    echo ""
    for vm in "VM1:10.73.15.100" "VM2:10.73.15.101" "VM3:10.73.15.102"; do
        nombre="${vm%%:*}"
        ip="${vm##*:}"
        if ping -c 1 -W 2 "$ip" > /dev/null 2>&1; then
            echo -e "  ${VERDE}✅${RESET} $nombre ($ip) — respondiendo"
        else
            echo -e "  ${ROJO}❌${RESET} $nombre ($ip) — sin respuesta"
        fi
    done
    echo ""
    read -p "  Presione ENTER para continuar..."
}

opcion_3() {
    echo -e "${CYAN}Ejecutando backup remoto de VM2...${RESET}"
    /opt/admin/scripts/backup_remoto.sh
    read -p "  Presione ENTER para continuar..."
}

opcion_4() {
    echo -e "${CYAN}Últimas 30 líneas de access.log en VM3:${RESET}"
    ssh -p 2222 usuario@10.73.15.102 "sudo tail -30 /var/log/nginx/noticias_access.log"
    echo ""
    read -p "  Presione ENTER para continuar..."
}

opcion_5() {
    echo -e "${CYAN}IPs baneadas por Fail2ban en VM3:${RESET}"
    ssh -p 2222 usuario@10.73.15.102 "sudo fail2ban-client status nginx-botsearch 2>/dev/null || echo 'Fail2ban no activo'"
    echo ""
    read -p "  Presione ENTER para continuar..."
}

opcion_6() {
    echo -e "${CYAN}Consultando API de noticias:${RESET}"
    echo ""
    curl -s http://10.73.15.101:3000/api/noticias | python3 -m json.tool 2>/dev/null \
        || curl -s http://10.73.15.101:3000/api/noticias
    echo ""
    read -p "  Presione ENTER para continuar..."
}

opcion_7() {
    echo -e "${CYAN}Verificando DNS para todos los subdominios:${RESET}"
    echo ""
    for sub in www static api admin; do
        result=$(dig @10.73.15.100 ${sub}.noticias.local +short 2>/dev/null)
        if [ -n "$result" ]; then
            echo -e "  ${VERDE}✅${RESET} ${sub}.noticias.local → ${result}"
        else
            echo -e "  ${ROJO}❌${RESET} ${sub}.noticias.local → Sin respuesta"
        fi
    done
    echo ""
    read -p "  Presione ENTER para continuar..."
}

opcion_8() {
    echo -e "${CYAN}Backups disponibles:${RESET}"
    ls -lh /opt/admin/backups/ 2>/dev/null || echo "  No hay backups aún"
    echo ""
    read -p "  Presione ENTER para continuar..."
}

opcion_9() {
    echo -e "${CYAN}¿A qué VM conectar?${RESET}"
    echo "  [1] VM1-Router    (10.73.15.100)"
    echo "  [2] VM2-Editorial (10.73.15.101)"
    echo "  [3] VM3-Cache     (10.73.15.102)"
    echo -n "  Opción: "
    read vm_opcion
    case $vm_opcion in
        1) ssh -p 2222 usuario@10.73.15.100 ;;
        2) ssh -p 2222 usuario@10.73.15.101 ;;
        3) ssh -p 2222 usuario@10.73.15.102 ;;
        *) echo "Opción inválida" ;;
    esac
}

# ─── Loop principal ──────────────────────────────────────────
while true; do
    mostrar_menu
    read opcion
    case $opcion in
        1) opcion_1 ;;
        2) opcion_2 ;;
        3) opcion_3 ;;
        4) opcion_4 ;;
        5) opcion_5 ;;
        6) opcion_6 ;;
        7) opcion_7 ;;
        8) opcion_8 ;;
        9) opcion_9 ;;
        0) echo -e "${VERDE}Saliendo...${RESET}"; exit 0 ;;
        *) echo -e "${ROJO}Opción inválida${RESET}"; sleep 1 ;;
    esac
done
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

```bash
chmod +x /opt/admin/scripts/menu_admin.sh
```

---

## 💾 PASO 5 — Script de Backup Remoto

```bash
nano /opt/admin/scripts/backup_remoto.sh
```

```bash
#!/bin/bash
# ─────────────────────────────────────────────────────────────
#  Backup remoto de VM2-Editorial → VM4-Admin
#  Proyecto 17 SIS313 USFX
# ─────────────────────────────────────────────────────────────

FECHA=$(date +%Y%m%d_%H%M%S)
DESTINO="/opt/admin/backups"
LOG="/opt/admin/logs/backup_remoto.log"
VM2_IP="10.73.15.101"
VM2_PUERTO="2222"
VM2_USER="usuario"
VM2_RUTA="/opt/noticias"
ARCHIVO="backup_vm2_${FECHA}.tar.gz"

mkdir -p "$DESTINO"

echo "[$(date)] ─── Iniciando backup remoto de VM2 ───" | tee -a "$LOG"

# Comprimir en VM2 y traer via SSH
ssh -p "$VM2_PUERTO" "${VM2_USER}@${VM2_IP}" \
    "tar -czf /tmp/${ARCHIVO} --exclude=/opt/noticias/backups ${VM2_RUTA}/api ${VM2_RUTA}/articulos" && \
scp -P "$VM2_PUERTO" "${VM2_USER}@${VM2_IP}:/tmp/${ARCHIVO}" "${DESTINO}/" && \
ssh -p "$VM2_PUERTO" "${VM2_USER}@${VM2_IP}" "rm /tmp/${ARCHIVO}"

# Verificar integridad
if tar -tzf "${DESTINO}/${ARCHIVO}" > /dev/null 2>&1; then
    TAMANIO=$(du -sh "${DESTINO}/${ARCHIVO}" | cut -f1)
    echo "[$(date)] ✅ Backup remoto OK: ${ARCHIVO} (${TAMANIO})" | tee -a "$LOG"
else
    echo "[$(date)] ❌ ERROR: Backup corrompido" | tee -a "$LOG"
    exit 1
fi

# Rotación: conservar últimos 5 backups remotos
ls -t "${DESTINO}"/backup_vm2_*.tar.gz | tail -n +6 | xargs rm -f 2>/dev/null
echo "[$(date)] Backups conservados: $(ls ${DESTINO}/backup_vm2_*.tar.gz 2>/dev/null | wc -l)" | tee -a "$LOG"
```

Guardamos: `Ctrl+O` → `Enter` → `Ctrl+X`

```bash
chmod +x /opt/admin/scripts/backup_remoto.sh
```

### 5.1 Programar con cron
```bash
crontab -e
```
Agregamos las siguientes líneas al final:
```
# Backup remoto cada 6 horas
0 */6 * * * /opt/admin/scripts/backup_remoto.sh
# Monitor automático cada 5 minutos (guarda log)
*/5 * * * * /opt/admin/scripts/monitor_infraestructura.sh >> /opt/admin/logs/monitor_auto.log 2>&1
```

---

## 🔥 PASO 6 — Firewall Local (UFW)

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH endurecido
sudo ufw allow 2222/tcp comment 'SSH admin'

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

---

## 👤 PASO 8 — Usuarios y Permisos

```bash
# Usuario administrador dedicado (no root)
sudo useradd -m -s /bin/bash adminsis313
sudo usermod -aG sudo adminsis313

# Permisos de scripts
sudo chown -R adminsis313:adminsis313 /opt/admin
sudo chmod -R 750 /opt/admin/scripts
sudo chmod -R 640 /opt/admin/logs

# Verificar
ls -la /opt/admin/
ls -la /opt/admin/scripts/
```

---

## ✅ PASO 9 — Verificaciones para el Parcial

```bash
# 1. Confirmar IP correcta
ip addr show | grep "inet "
# Debe mostrar: inet 10.73.15.103/24

# 2. Ping a todas las VMs
ping -c 2 10.73.15.100  # VM1-Router
ping -c 2 10.73.15.101  # VM2-Editorial
ping -c 2 10.73.15.102  # VM3-Cache

# 3. Monitoreo completo
/opt/admin/scripts/monitor_infraestructura.sh

# 4. Menú interactivo
/opt/admin/scripts/menu_admin.sh

# 5. Backup remoto funcionando
/opt/admin/scripts/backup_remoto.sh
ls -lh /opt/admin/backups/

# 6. DNS completo desde VM1
for sub in www static api admin; do
    echo -n "$sub.noticias.local → "
    dig @10.73.15.100 ${sub}.noticias.local +short
done

# 7. Firewall
sudo ufw status verbose

# 8. SSH endurecido
grep -E "^(Port|PermitRootLogin|PasswordAuthentication)" /etc/ssh/sshd_config

# 9. Permisos
ls -la /opt/admin/
ls -la /opt/admin/scripts/

# 10. Cron activo
crontab -l

# 11. Acceso SSH a las otras VMs (sin contraseña)
ssh -p 2222 usuario@10.73.15.100 "hostname"
ssh -p 2222 usuario@10.73.15.101 "hostname && pm2 status 2>/dev/null || systemctl is-active node"
ssh -p 2222 usuario@10.73.15.102 "hostname && sudo nginx -t"

# 12. Logs de monitoreo
cat /opt/admin/logs/monitor_$(date +%Y%m%d).log
```

---

## 🚨 Comandos de Emergencia

```bash
# Si no hay conexión SSH a otra VM (modo verbose para depurar)
ssh -p 2222 -v usuario@10.73.15.101

# Si backup falla — copia manual de un archivo
scp -P 2222 usuario@10.73.15.101:/opt/noticias/api/app.js /tmp/

# Restaurar backup en VM2
scp -P 2222 /opt/admin/backups/backup_vm2_*.tar.gz usuario@10.73.15.101:/tmp/
ssh -p 2222 usuario@10.73.15.101 "tar -xzf /tmp/backup_vm2_*.tar.gz -C /opt/"

# Ver todos los logs de monitoreo
tail -f /opt/admin/logs/monitor_$(date +%Y%m%d).log

# Reiniciar la red si perdiste conectividad
sudo netplan apply

# Si te quedaste sin acceso SSH
# Accede directo por consola VirtualBox y ejecuta:
sudo ufw disable
# Luego corrige las reglas y vuelve a habilitar
```

---

*Proyecto 17 — SIS313 USFX | VM4 · 10.73.15.103 · Administración / Monitoreo*
