# Kali Linux ARM64 en VMware Fusion - Workflow de Pentesting

## Configuración del Sistema

| Componente | Especificación |
|------------|----------------|
| **Host** | MacBook Pro M4 Pro |
| **RAM Host** | 48GB unificada |
| **CPU Host** | 14 núcleos |
| **Hypervisor** | VMware Fusion Pro 13.6.3 |
| **Guest OS** | Kali Linux 2025.3 ARM64 |
| **RAM VM** | 16GB recomendado |
| **vCPU VM** | 8 cores recomendado |

---

## Máquinas Virtuales del Lab

### VM Principal: kali-linux-2025.3

| Propiedad | Valor |
|-----------|-------|
| Nombre | kali-linux-2025.3 |
| Propósito | VM principal de trabajo |
| Script | `~/bin/kali-start.sh` |
| Alias | `kali-*` |

### VM Attacker: kali-attacker-3

| Propiedad | Valor |
|-----------|-------|
| Nombre | kali-attacker-3 |
| Propósito | Laboratorio de ataques, pruebas aisladas |
| Script | `~/bin/kali3-start.sh` |
| Alias | `kali3-*` |

### Configuración de Red

| Parámetro | Valor |
|-----------|-------|
| Tipo | NAT (vmnet8) |
| Subnet | 192.168.25.0/24 |
| Gateway | 192.168.25.2 |
| DHCP | Habilitado |

---

## Comandos Rápidos

### VM Principal (kali)

```bash
# Estado de la VM
kali-status          # Ver si está corriendo
kali-ip              # Obtener IP de la VM

# Control de VM
kali-start           # Iniciar headless (background)
kali-gui             # Iniciar con interfaz gráfica
kali-stop            # Apagar gracefully
kali-suspend         # Suspender

# Conexión
kali-ssh             # SSH automático a la VM

# Snapshots
kali-snap            # Crear snapshot con fecha
kali-snap "pre-test" # Snapshot con nombre
kali list            # Listar snapshots
```

### VM Attacker-3 (kali3)

```bash
# Estado de la VM
kali3-status         # Ver si está corriendo
kali3-ip             # Obtener IP de la VM

# Control de VM
kali3-start          # Iniciar headless (background)
kali3-gui            # Iniciar con interfaz gráfica
kali3-stop           # Apagar gracefully

# Conexión
kali3-ssh            # SSH automático a la VM

# Snapshots
kali3-snap           # Crear snapshot con fecha
```

### Lab Completo (Ambas VMs)

```bash
# Iniciar todo el laboratorio
lab-start            # Inicia kali + kali3

# Detener todo el laboratorio
lab-stop             # Detiene kali + kali3

# Ver estado del laboratorio
lab-status           # Muestra estado de ambas VMs
```

**Ejemplo de salida de `lab-status`:**
```
=== Kali Principal ===
Kali VM is RUNNING
IP Address: 192.168.25.128

=== Kali Attacker-3 ===
Kali Attacker-3 is RUNNING
IP Address: 192.168.25.129
```

---

## Flujo de Trabajo de Pentesting

### 1. Preparación del Laboratorio

```bash
# 1. Iniciar todo el lab (ambas VMs)
lab-start

# 2. Verificar estado y conectividad
lab-status

# 3. Conectar a cada VM
kali-ssh              # Terminal 1: VM principal
kali3-ssh             # Terminal 2: VM attacker

# 4. Crear snapshots base
kali-snap "clean-state"
kali3-snap "clean-state"
```

**Alternativa: Una sola VM**
```bash
kali-start            # Solo VM principal
kali-ip
kali-snap "clean-state"
```

### 2. Reconocimiento

```bash
# Dentro de Kali VM:

# Escaneo de red
sudo nmap -sn 192.168.x.0/24

# Escaneo de puertos
sudo nmap -sV -sC -p- <target>

# Enumeración web
whatweb <url>
nikto -h <url>
```

### 3. Herramientas por Categoría

#### Información Gathering
| Tool | Uso |
|------|-----|
| `nmap` | Escaneo de puertos y servicios |
| `masscan` | Escaneo rápido de puertos |
| `theHarvester` | OSINT - emails, subdominios |
| `recon-ng` | Framework de reconocimiento |
| `amass` | Enumeración de subdominios |

#### Web Application Testing
| Tool | Uso |
|------|-----|
| `burpsuite` | Proxy interceptor |
| `sqlmap` | SQL injection automático |
| `nikto` | Escáner de vulnerabilidades web |
| `dirb/gobuster` | Directory bruteforcing |
| `wfuzz` | Fuzzing web |

#### Password Attacks
| Tool | Uso |
|------|-----|
| `john` | Cracking de hashes |
| `hashcat` | GPU hash cracking |
| `hydra` | Brute force online |
| `medusa` | Parallel login brute forcer |

#### Exploitation
| Tool | Uso |
|------|-----|
| `metasploit` | Framework de explotación |
| `searchsploit` | Búsqueda de exploits |
| `msfvenom` | Generador de payloads |

#### Post-Exploitation
| Tool | Uso |
|------|-----|
| `mimikatz` | Extracción de credenciales Windows |
| `linpeas/winpeas` | Escalación de privilegios |
| `crackmapexec` | Movimiento lateral |

---

## Integración con macOS Host

### Compartir Archivos

```bash
# Opción 1: Carpeta compartida VMware
# Configurar en: VM Settings > Sharing

# Opción 2: SCP desde host
scp archivo.txt kali@$(kali-ip):/home/kali/

# Opción 3: SCP desde Kali
scp kali@$(kali-ip):/home/kali/resultados.txt ~/Desktop/
```

### Port Forwarding para Herramientas Web

```bash
# En macOS, acceder a Burp Suite de Kali
ssh -L 8080:localhost:8080 kali@$(kali-ip)
# Ahora accede a http://localhost:8080 en macOS
```

### Clipboard Compartido

VMware Tools permite copiar/pegar entre macOS y Kali si está instalado:

```bash
# Dentro de Kali
sudo apt install open-vm-tools-desktop
```

---

## Configuración de Red

### Modos de Red Disponibles

| Modo | Uso | IP |
|------|-----|-----|
| **NAT** | Acceso a internet, oculto en red local | 192.168.x.x |
| **Bridged** | Mismo segmento que host, visible en red | DHCP de red |
| **Host-Only** | Solo comunicación con host | 172.16.x.x |

### Cambiar Modo de Red

1. VM > Settings > Network Adapter
2. Seleccionar modo deseado
3. Reiniciar servicios de red en Kali:
   ```bash
   sudo systemctl restart NetworkManager
   ```

---

## Buenas Prácticas

### Antes de Cada Engagement

1. **Snapshot limpio**: `kali-snap "pre-engagement-$(date +%Y%m%d)"`
2. **Actualizar**: `sudo apt update && sudo apt full-upgrade -y`
3. **Verificar conectividad**: `ping -c 3 8.8.8.8`
4. **Documentar**: Crear directorio para el proyecto

### Durante el Engagement

```bash
# Crear workspace
mkdir -p ~/engagements/$(date +%Y%m%d)-cliente
cd ~/engagements/$(date +%Y%m%d)-cliente

# Logging automático
script session-$(date +%H%M%S).log
```

### Después del Engagement

1. **Exportar evidencia**
2. **Restaurar snapshot**: Para limpiar cualquier dato sensible
3. **Documentar lecciones aprendidas**

---

## Troubleshooting

### VM no obtiene IP

```bash
# Dentro de Kali
sudo dhclient -v eth0
# o
sudo systemctl restart NetworkManager
```

### VMware Tools no funciona

```bash
# Reinstalar
sudo apt purge open-vm-tools open-vm-tools-desktop
sudo apt install open-vm-tools open-vm-tools-desktop
sudo reboot
```

### Rendimiento lento

1. Verificar asignación de RAM/CPU en VM Settings
2. Deshabilitar efectos visuales en XFCE
3. Usar Kali headless + SSH para máximo rendimiento

### SSH Connection Refused

```bash
# Dentro de Kali
sudo systemctl enable ssh
sudo systemctl start ssh
```

---

## Scripts Útiles

### Escaneo Rápido de Red

```bash
#!/bin/bash
# quick-scan.sh
TARGET=$1
echo "[*] Quick scan of $TARGET"
nmap -sV -sC --top-ports 1000 -oA quick-$TARGET $TARGET
```

### Generador de Wordlists Personalizado

```bash
#!/bin/bash
# gen-wordlist.sh
COMPANY=$1
cewl -d 2 -m 5 https://$COMPANY.com -w $COMPANY-wordlist.txt
```

---

## Recursos

- [Kali Documentation](https://www.kali.org/docs/)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [HackTricks](https://book.hacktricks.xyz/)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)

---

## Checklist de Setup Inicial

### VM Principal (kali-linux-2025.3)
- [x] VMware Fusion Pro instalado
- [x] Kali Linux ARM64 ISO descargado y verificado
- [x] VM creada con recursos óptimos (8 vCPU, 16GB RAM)
- [x] Script `kali-start.sh` configurado
- [x] Aliases `kali-*` en `.zshrc` agregados
- [x] VMware Tools instalado en guest
- [x] SSH habilitado en Kali
- [x] Snapshot "clean-install" creado
- [ ] Carpeta compartida configurada

### VM Attacker (kali-attacker-3)
- [x] VM clonada desde kali-linux-2025.3
- [x] Script `kali3-start.sh` configurado
- [x] Aliases `kali3-*` en `.zshrc` agregados
- [x] Hostname cambiado a `kali-attacker-3`
- [x] Claves SSH regeneradas
- [ ] Snapshot "clean-state" creado

### Lab Completo
- [x] Aliases `lab-start`, `lab-stop`, `lab-status` configurados
- [x] Red NAT compartida (192.168.25.0/24)
- [ ] Comunicación entre VMs verificada

---

## Documentación Relacionada

- [SCRIPTS_WORKFLOW.md](./SCRIPTS_WORKFLOW.md) - Scripts y automatizaciones
- [SHELL_WORKFLOW.md](./SHELL_WORKFLOW.md) - Configuración de shell
- [README.md](./README.md) - Setup general de Neovim
