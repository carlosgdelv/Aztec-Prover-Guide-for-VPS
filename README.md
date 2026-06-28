<div align="center">

# 🖥️ Linux Operating System & Node Infrastructure Handbook

### Guía práctica de administración de sistemas Linux y despliegue de nodos blockchain (Ethereum L1 + Aztec L2)

[![Bash](https://img.shields.io/badge/Bash-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white)](#)
[![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)](#)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)](#)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](#)
[![Ethereum](https://img.shields.io/badge/Ethereum-3C3C3D?style=for-the-badge&logo=ethereum&logoColor=white)](#)

[![License](https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)](LICENSE)
[![Maintained](https://img.shields.io/badge/maintained-yes-success.svg?style=flat-square)](#)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](#)

</div>

---

## 📖 Acerca de

Colección de comandos, procedimientos y guías de despliegue para administración de sistemas **Linux (Ubuntu)** y para operar infraestructura de nodos blockchain: un nodo Ethereum L1 (Geth + Prysm) y un prover de Aztec L2 sobre VPS.

> [!IMPORTANT]
> Las secciones 6 y 7 (Geth/Prysm y Aztec Prover) involucran **claves privadas y fondos reales**. Cada bloque de versión incluye una nota de verificación con fecha y enlace a la fuente oficial. Antes de desplegar en mainnet, comprueba siempre la versión vigente en la documentación oficial enlazada — estos protocolos actualizan versiones con frecuencia.

---

## 📑 Tabla de contenidos

| # | Sección | Descripción |
|---|---------|-------------|
| 1 | [🐧 Instalación Live de Linux](#-1-instalación-live-de-linux) | Creación de USB arrancable, verificación de ISO |
| 2 | [💾 Almacenamiento y LVM](#-2-almacenamiento-y-lvm) | Diagnóstico, extensión y reducción de volúmenes |
| 3 | [🔍 Monitorización del sistema](#-3-monitorización-del-sistema) | CPU, disco, red, logs y contenedores |
| 4 | [🐳 Docker](#-4-docker) | Contenedores, Compose, volúmenes, imágenes y redes |
| 5 | [🛠️ Herramientas generales de sistema](#-5-herramientas-generales-de-sistema) | SSH, tmux, rsync, cron, firewall |
| 6 | [⟠ Nodo Ethereum L1 — Geth + Prysm](#-6-nodo-ethereum-l1--geth--prysm) | Execution + consensus client con Docker Compose |
| 7 | [🌳 Aztec Prover — Broker + Agents](#-7-aztec-prover--broker--agents) | Arquitectura distribuida de prueba en VPS |

---

## 🐧 1. Instalación Live de Linux

> Procedimiento para crear un USB arrancable con una distribución Linux (ejemplo: Ubuntu 24.04).

> [!CAUTION]
> El comando `dd` escribe a nivel de bloque y **borra todo el contenido del disco de destino sin posibilidad de deshacerlo**. Verifica el dispositivo (`/dev/sdX`) con `lsblk -fp` antes de cada paso. Un error de dispositivo puede destruir tu disco principal.

### 1.1 Detectar y preparar el USB

**Paso 1 — Detectar la ruta del USB**

```bash
lsblk -fp
```

**Paso 2 — Desmontar el USB (si está montado)**

```bash
sudo umount /dev/sda1
sudo umount /dev/sda2
```

**Paso 3 — Borrar la tabla de particiones del USB**

> ⚠️ Solo el USB, nunca tu SSD/disco principal.

```bash
sudo wipefs -a /dev/sda
```

---

### 1.2 Descargar y verificar la imagen ISO

**Paso 4 — Descargar una ISO de Linux**

Ejemplo con Ubuntu 24.04:

```bash
wget https://releases.ubuntu.com/24.04/ubuntu-24.04-desktop-amd64.iso -O ubuntu.iso
```

**Paso 5 — Verificar la integridad de la ISO**

Antes de grabar la imagen, comprueba que no está corrupta ni manipulada comparando su checksum con el oficial publicado por Ubuntu.

```bash
sha256sum ubuntu.iso
```

Compara el resultado con el valor publicado en [releases.ubuntu.com](https://releases.ubuntu.com/24.04/SHA256SUMS).

---

### 1.3 Grabar la ISO en el USB

**Paso 6 — Escribir la ISO en el USB**

```bash
sudo dd if=ubuntu.iso of=/dev/sda bs=4M status=progress oflag=sync
```

> 💡 **Alternativa con barra de progreso (`pv`)**: si prefieres una barra de progreso más detallada que `status=progress`, puedes canalizar la escritura a través de `pv`:
>
> ```bash
> sudo apt install pv -y
> pv ubuntu.iso | sudo dd of=/dev/sda bs=4M oflag=sync
> ```

**Paso 7 — Sincronizar y expulsar**

```bash
sync
sudo eject /dev/sda
```

Ya puedes retirar el USB de forma segura.

---

### 1.4 Comprobación final

Al ejecutar de nuevo `lsblk -fp` deberías ver algo similar a:

```text
/dev/sda iso9660 Ubuntu 24.04 amd64
```

---

## 💾 2. Almacenamiento y LVM

> Diagnóstico previo, extensión y reducción de volúmenes lógicos (LVM) sobre discos físicos en una VM Linux.

### 2.1 Diagnóstico previo (antes de tocar nada)

**Listar discos, particiones y puntos de montaje**

```bash
lsblk -fp
```

**Listar particiones con detalle (tabla de particiones, tamaños, tipos)**

```bash
sudo fdisk -l
```

---

### 2.2 Extender un volumen lógico (añadir disco nuevo)

**Paso 1 — Actualizar el sistema e instalar herramientas LVM**

```bash
sudo apt update
sudo apt install lvm2 -y
```

**Paso 2 — Borrar datos y preparar el nuevo disco**

Elimina cualquier dato de sistema de archivos/particiones existente e inicializa el disco como volumen físico.

```bash
sudo wipefs --all /dev/nvme1n1
sudo sgdisk --zap-all /dev/nvme1n1
sudo pvcreate /dev/nvme1n1
```

**Paso 3 — Añadir el nuevo disco al grupo de volúmenes**

Extiende el grupo de volúmenes (`ubuntu-vg`) para incluir el nuevo disco.

```bash
sudo vgextend ubuntu-vg /dev/nvme1n1
```

**Paso 4 — Confirmar los cambios en el grupo de volúmenes**

```bash
sudo vgs
```

**Paso 5 — Extender el volumen lógico con todo el espacio libre**

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

**Paso 6 — Redimensionar el sistema de archivos**

```bash
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

**Paso 7 — Verificar los cambios**

```bash
df -h /
df -T /
```

**Paso 8 — (Opcional) Ver detalle del volumen lógico**

```bash
sudo lvdisplay /dev/ubuntu-vg/ubuntu-lv
```

---

### 2.3 Reducir un volumen lógico

> [!WARNING]
> Reducir un volumen es una operación de riesgo: si el nuevo tamaño es menor que los datos existentes, **se pierden datos**. Haz siempre una copia de seguridad antes y verifica el sistema de archivos en cada paso.

**Paso 1 — Desmontar el volumen**

No se puede reducir un sistema de archivos `ext4` en caliente.

```bash
sudo umount /dev/ubuntu-vg/ubuntu-lv
```

**Paso 2 — Verificar el sistema de archivos**

```bash
sudo e2fsck -f /dev/ubuntu-vg/ubuntu-lv
```

**Paso 3 — Reducir primero el sistema de archivos**

Reduce el filesystem a un tamaño *menor o igual* al que tendrá el volumen lógico (ejemplo: 20 GB).

```bash
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv 20G
```

**Paso 4 — Reducir el volumen lógico**

```bash
sudo lvreduce -L 20G /dev/ubuntu-vg/ubuntu-lv
```

**Paso 5 — Volver a montar el volumen**

```bash
sudo mount /dev/ubuntu-vg/ubuntu-lv
```

---

### 2.4 Gestión gráfica de particiones (alternativa)

Para quien prefiera una interfaz visual en lugar de la terminal:

```bash
sudo apt install gparted -y
sudo gparted
```

---

## 🔍 3. Monitorización del sistema

> Comandos para vigilar el uso de CPU, memoria, disco, red, logs y contenedores específicos.

### 3.1 CPU y memoria

**Monitor interactivo de procesos (CPU, memoria, usuario)**

```bash
htop
```

**Resumen de memoria RAM y swap**

```bash
free -h
```

---

### 3.2 Disco

**Uso de disco por sistema de archivos**

```bash
df -h
```

**Explorador visual interactivo de uso de disco (más cómodo que `du`)**

```bash
sudo apt install ncdu -y
ncdu /
```

**Uso de disco por proceso, en tiempo real**

```bash
sudo apt install iotop -y
sudo iotop
```

---

### 3.3 Red

**Uso de red por proceso, en tiempo real**

```bash
sudo apt install nethogs -y
sudo nethogs
```

**Tráfico de red por interfaz, en tiempo real (estilo `top`)**

```bash
sudo apt install iftop -y
sudo iftop
```

---

### 3.4 Logs del sistema

**Ver logs del sistema en tiempo real (systemd)**

```bash
journalctl -f
```

**Ver logs de un servicio concreto**

```bash
journalctl -u <nombre-del-servicio> -f
```

---

### 3.5 Monitorización de contenedores específicos del nodo

**Uso de disco — contenedor Geth**

```bash
docker exec -it geth du -sh /data
```

**Uso de disco — contenedor Prysm**

```bash
docker exec -it prysm du -sh /data
```

**Uso de disco — contenedor de Aztec**

```bash
docker exec -it aztec-prover-prover-node-1 du -sh /var/lib/data
```

---

## 🐳 4. Docker

> Comandos esenciales para el día a día con contenedores, Docker Compose, volúmenes, imágenes y redes.

### 4.1 Contenedores

| Comando | Descripción |
|---------|-------------|
| `docker ps` | Muestra todos los contenedores en ejecución. |
| `docker ps -a` | Lista todos los contenedores, incluidos los detenidos. |
| `docker stats` | Métricas en tiempo real: CPU, memoria, red y disco. |
| `docker logs <id\|nombre>` | Muestra los logs de un contenedor. |
| `docker stop <id\|nombre>` | Detiene un contenedor en ejecución de forma ordenada. |
| `docker rm <id\|nombre>` | Elimina permanentemente un contenedor. |
| `docker container prune` | Elimina todos los contenedores detenidos para liberar espacio. |

**Ver contenedores activos**

```bash
docker ps
```

**Listar todos los contenedores (incluidos los detenidos)**

```bash
docker ps -a
```

**Métricas en tiempo real**

```bash
docker stats
```

**Ver logs de un contenedor**

Útil para depurar un nodo (geth, prysm, aztec, etc.) sin entrar dentro del contenedor.

```bash
docker logs -f geth
```

**Detener un contenedor**

```bash
docker stop <container_id_or_name>
```

**Eliminar un contenedor**

```bash
docker rm <container_id_or_name>
```

**Eliminar todos los contenedores detenidos**

```bash
docker container prune
```

---

### 4.2 Docker Compose

> Gestión de servicios multi-contenedor definidos en un `docker-compose.yml` (típico cuando hay varios nodos como geth, prysm y aztec en el mismo stack).

**Levantar todos los servicios en segundo plano**

```bash
docker compose up -d
```

**Detener y eliminar todos los servicios del stack**

```bash
docker compose down
```

**Ver logs de todos los servicios del stack**

```bash
docker compose logs -f
```

**Reiniciar un servicio concreto**

```bash
docker compose restart <nombre_servicio>
```

---

### 4.3 Volúmenes

**Listar volúmenes**

```bash
docker volume ls
```

**Eliminar volúmenes no utilizados**

> [!WARNING]
> Esto borra de forma permanente los datos de cualquier volumen que no esté en uso por un contenedor activo.

```bash
docker volume prune
```

---

### 4.4 Imágenes

**Listar imágenes descargadas**

```bash
docker images
```

**Eliminar todas las imágenes no utilizadas por ningún contenedor**

```bash
docker image prune -a
```

---

### 4.5 Redes

**Listar redes de Docker**

```bash
docker network ls
```

---

### 4.6 Desinstalación completa de Docker

> [!WARNING]
> Esto elimina Docker y **todos los datos** asociados (imágenes, volúmenes, contenedores) de forma irreversible.

```bash
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

---

## 🛠️ 5. Herramientas generales de sistema

> Utilidades transversales para trabajo con servidores remotos, transferencia de archivos, automatización y seguridad básica.

### 5.1 SSH — acceso remoto

**Generar un par de claves SSH**

```bash
ssh-keygen -t ed25519 -C "tu_email@ejemplo.com"
```

**Copiar la clave pública a un servidor remoto**

```bash
ssh-copy-id usuario@servidor
```

**Conectarse a un servidor remoto**

```bash
ssh usuario@servidor
```

**Conectarse usando un puerto distinto al 22**

```bash
ssh -p 2222 usuario@servidor
```

---

### 5.2 tmux — sesiones persistentes

> Permite mantener procesos corriendo en un servidor remoto aunque se cierre la conexión SSH.

**Crear una nueva sesión con nombre**

```bash
tmux new -s nombre_sesion
```

**Listar sesiones activas**

```bash
tmux ls
```

**Desconectarse de la sesión sin cerrarla**

Atajo de teclado dentro de tmux:

```text
Ctrl + b, luego d
```

**Volver a conectarse a una sesión existente**

```bash
tmux attach -t nombre_sesion
```

**Cerrar una sesión por completo**

```bash
tmux kill-session -t nombre_sesion
```

---

### 5.3 rsync — transferencia y backup de archivos

**Copiar una carpeta local a un servidor remoto**

```bash
rsync -avz /ruta/local/ usuario@servidor:/ruta/remota/
```

**Copiar desde un servidor remoto a local**

```bash
rsync -avz usuario@servidor:/ruta/remota/ /ruta/local/
```

**Sincronizar borrando en destino lo que ya no existe en origen**

> [!CAUTION]
> La opción `--delete` elimina en destino los archivos que ya no estén en origen. Revisa siempre antes con `--dry-run`.

```bash
rsync -avz --delete --dry-run /ruta/local/ usuario@servidor:/ruta/remota/
```

---

### 5.4 cron — tareas programadas

**Editar las tareas programadas del usuario actual**

```bash
crontab -e
```

**Listar las tareas programadas actuales**

```bash
crontab -l
```

**Ejemplo: ejecutar un script todos los días a las 3:00 AM**

```cron
0 3 * * * /ruta/al/script.sh
```

---

### 5.5 ufw — firewall básico (Ubuntu)

**Activar el firewall**

```bash
sudo ufw enable
```

**Ver el estado y las reglas activas**

```bash
sudo ufw status verbose
```

**Permitir un puerto concreto (ejemplo: SSH)**

```bash
sudo ufw allow 22/tcp
```

**Denegar un puerto concreto**

```bash
sudo ufw deny 8080/tcp
```

**Eliminar una regla**

```bash
sudo ufw delete allow 22/tcp
```

---

## ⟠ 6. Nodo Ethereum L1 — Geth + Prysm

> Despliegue de un nodo completo de Ethereum mainnet usando **Geth** como execution client y **Prysm** como consensus client, vía Docker Compose en Ubuntu.

> [!NOTE]
> ✅ **Verificado (28 jun 2026)**
> - Geth: última serie estable **v1.17.x** (release reciente v1.17.3). Hard fork **Fusaka** activado en mainnet el 3 dic 2025. Fuente: [github.com/ethereum/go-ethereum/releases](https://github.com/ethereum/go-ethereum/releases)
> - Prysm: el proyecto **se renombró de `prysmaticlabs` a `OffchainLabs`**. Última versión estable **v7.1.3** (marzo 2026). La imagen Docker oficial ahora es `gcr.io/offchainlabs/prysm/beacon-chain` (la antigua `gcr.io/prysmaticlabs/...` sigue funcionando pero será descontinuada). Fuente: [prysm.offchainlabs.com](https://prysm.offchainlabs.com/docs/install-prysm/install-with-docker/) y [github.com/OffchainLabs/prysm/issues/15139](https://github.com/OffchainLabs/prysm/issues/15139)
> - Ambos proyectos recomiendan usar el tag **`:stable`** en lugar de fijar un número de versión a mano, para recibir parches de seguridad automáticamente.

### 6.1 Instalar dependencias del sistema

**Actualizar el sistema**

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

**Instalar paquetes esenciales de desarrollo y utilidades**

```bash
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip -y
```

---

### 6.2 Instalar Docker Engine

**Eliminar versiones previas en conflicto**

```bash
sudo apt update -y && sudo apt upgrade -y
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

**Añadir la clave GPG oficial de Docker**

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

**Añadir el repositorio de Docker**

```bash
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**Instalar Docker Engine, CLI y plugins**

```bash
sudo apt update -y && sudo apt upgrade -y
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**Probar la instalación**

```bash
sudo docker run hello-world
```

**Habilitar Docker al arranque y reiniciar el servicio**

```bash
sudo systemctl enable docker
sudo systemctl restart docker
```

---

### 6.3 Añadir tu usuario al grupo Docker

> Permite ejecutar comandos `docker` sin necesidad de `sudo`.

**Verificar tu nombre de usuario**

```bash
whoami
```

**Verificar tu UID (debe mostrar 1000 si eres el primer usuario)**

```bash
id -u
```

**Verificar tu GID**

```bash
id -g
```

**Crear el grupo docker (si no existe ya)**

```bash
sudo groupadd docker
```

**Añadir tu usuario al grupo docker**

Sustituye `carlos` por tu nombre de usuario real.

```bash
sudo usermod -aG docker carlos
```

**Reiniciar sesión o el sistema para aplicar el cambio**

```bash
reboot
```

---

### 6.4 Crear la estructura de directorios

```bash
mkdir -p ~/ethereum-mainnet/execution ~/ethereum-mainnet/consensus
```

---

### 6.5 Generar el secreto JWT

> El JWT secret autentica la comunicación entre el execution client (Geth) y el consensus client (Prysm) a través del Engine API.

**Generar un secreto JWT de 32 bytes en hexadecimal**

```bash
openssl rand -hex 32 > ~/ethereum-mainnet/jwt.hex
```

**Proteger el archivo con permisos restrictivos**

```bash
chmod 600 ~/ethereum-mainnet/jwt.hex
```

**Verificar que se generó correctamente**

```bash
cat ~/ethereum-mainnet/jwt.hex
```

---

### 6.6 Configurar `docker-compose.yml`

**Cambiar al directorio de trabajo**

```bash
cd ~/ethereum-mainnet
```

**Abrir/crear el archivo de configuración**

```bash
nano docker-compose.yml
```

**Contenido del `docker-compose.yml`**

> 💡 Se usa el tag `stable` en ambas imágenes en lugar de una versión fija, siguiendo la recomendación oficial de ambos proyectos para recibir actualizaciones de seguridad automáticamente. Sustituye `carlos` por tu usuario real en las rutas de volúmenes.

```yaml
services:
  geth:
    image: ethereum/client-go:stable
    container_name: geth
    network_mode: host
    user: "1000:1000"
    restart: unless-stopped
    ports:
      - 30303:30303
      - 30303:30303/udp
      - 8545:8545
      - 8546:8546
      - 8551:8551
    volumes:
      - /home/carlos/ethereum-mainnet/execution:/data
      - /home/carlos/ethereum-mainnet/jwt.hex:/data/jwt.hex
    command:
      - --mainnet
      - --http
      - --http.api=eth,net,web3
      - --http.addr=127.0.0.1
      - --authrpc.addr=127.0.0.1
      - --authrpc.vhosts=*
      - --authrpc.jwtsecret=/data/jwt.hex
      - --authrpc.port=8551
      - --syncmode=snap
      - --cache=8192
      - --maxpeers=50
      - --datadir=/data
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  prysm:
    image: gcr.io/offchainlabs/prysm/beacon-chain:stable
    container_name: prysm
    network_mode: host
    user: "1000:1000"
    restart: unless-stopped
    volumes:
      - /home/carlos/ethereum-mainnet/consensus:/data
      - /home/carlos/ethereum-mainnet/jwt.hex:/data/jwt.hex
    depends_on:
      - geth
    ports:
      - 4000:4000
      - 3500:3500
      - 13000:13000
      - 12000:12000/udp
    command:
      - --mainnet
      - --accept-terms-of-use
      - --datadir=/data
      - --disable-monitoring
      - --rpc-host=127.0.0.1
      - --execution-endpoint=http://127.0.0.1:8551
      - --jwt-secret=/data/jwt.hex
      - --rpc-port=4000
      - --grpc-gateway-corsdomain=*
      - --grpc-gateway-host=127.0.0.1
      - --grpc-gateway-port=3500
      - --p2p-tcp-port=13000
      - --p2p-udp-port=12000
      - --min-sync-peers=3
      - --checkpoint-sync-url=https://mainnet.checkpoint.sigp.io
      - --genesis-beacon-api-url=https://mainnet.checkpoint.sigp.io
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

> [!TIP]
> La imagen `gcr.io/offchainlabs/prysm/beacon-chain` es la URL oficial actual. Si prefieres fijar una versión exacta en lugar de `:stable` (recomendable en producción para evitar cambios inesperados), consulta la [página de releases de Prysm](https://github.com/OffchainLabs/prysm/releases) y la de [Geth](https://github.com/ethereum/go-ethereum/releases) antes de desplegar.

---

### 6.7 Ejecutar y gestionar los nodos

**Iniciar Geth y Prysm en segundo plano**

```bash
docker compose up -d
```

**Ver los logs en tiempo real de ambos contenedores**

```bash
docker compose logs -f
```

**Detener y eliminar los contenedores**

```bash
docker compose down
```

---

### 6.8 Configurar el firewall (UFW)

**Aplicar las reglas necesarias**

```bash
# 1. Permitir acceso SSH (solo si usas SSH, si no, omite esta línea)
sudo ufw allow OpenSSH

# 2. Permitir tráfico P2P necesario: Geth P2P (TCP/UDP) y Prysm P2P/gossip
sudo ufw allow 30303/tcp
sudo ufw allow 30303/udp
sudo ufw allow 13000/tcp
sudo ufw allow 12000/udp

# 3. Política por defecto: denegar entrante
sudo ufw default deny incoming

# 4. Permitir saliente
sudo ufw default allow outgoing

# 5. Activar el firewall
sudo ufw enable
```

**Ver puertos en uso**

```bash
sudo ss -tulnp
```

**Ver las reglas activas de UFW**

```bash
sudo ufw status verbose
```

---

## 🌳 7. Aztec Prover — Broker + Agents

> Despliegue distribuido de un prover de Aztec en arquitectura separada: un nodo "Broker" (prover-node + prover-broker) y uno o varios nodos "Agent" (prover-agent) dedicados al cómputo intensivo.

> [!NOTE]
> ✅ **Verificado (28 jun 2026) contra la documentación oficial — Alpha (mainnet) v4.3.1**
> Fuente principal: [docs.aztec.network/operate/operators/setup/running_a_prover](https://docs.aztec.network/operate/operators/setup/running_a_prover) y [Prerequisites](https://docs.aztec.network/operate/operators/prerequisites)
>
> Cambios relevantes respecto a guías de comunidad antiguas (incluida `alpha-testnet` y la imagen `:latest`):
> - La imagen oficial actual es **`aztecprotocol/aztec:4.3.1`**, no `:latest`. La propia documentación advierte: *"Always refer to the docs to check that you're using the correct image"*.
> - El flag de red para producción es **`--network mainnet`** (no `alpha-testnet`, que es la red de pruebas).
> - El puerto oficial del broker es **8080** (variable `PROVER_BROKER_PORT`), no 8081.
> - Existe un puerto admin **8880** que la documentación indica explícitamente **no expone nunca** al host por motivos de seguridad.
>
> ⚠️ **Nota sobre hardware**: los requisitos de "Broker Box" (48 cores/256GB) y "Main Rig" (192 cores/256-768GB) que puedas tener en guías de comunidad **no son cifras oficiales de Aztec** — son configuraciones prácticas de operadores. Los mínimos oficiales se detallan en 7.1.

### 7.1 Arquitectura y requisitos mínimos oficiales

El prover de Aztec consta de tres componentes:

1. **Prover node**: sondea L1 en busca de épocas sin probar, crea los trabajos de prueba, los distribuye al broker y publica la prueba final del rollup en el contrato L1.
2. **Prover broker**: gestiona la cola de trabajos, distribuyéndolos a los agentes y recolectando resultados.
3. **Prover agent(s)**: ejecutan los trabajos de generación de pruebas de forma stateless (sin estado).

**Requisitos mínimos oficiales por componente:**

| Componente | CPU | RAM | Almacenamiento | Red |
|---|---|---|---|---|
| Prover Node | 16 cores / 32 vCPU (2015+) | 16 GB | 1 TB NVMe SSD | 25 Mbps |
| Prover Broker | 8 cores / 16 vCPU (2015+) | 16 GB | 10 GB SSD | — |
| Prover Agent (c/u) | 32 cores / 64 vCPU (2015+) | 128 GB | 10 GB SSD | — |

**Escalado de agentes en una misma máquina** (ajustando `PROVER_AGENT_COUNT`):

| Nº de agentes | Cores necesarios | RAM necesaria |
|---|---|---|
| 2 agentes | 64 cores | 256 GB |
| 3 agentes | 96 cores | 384 GB |
| 4 agentes | 128 cores | 512 GB |

> Estos requisitos están sujetos a cambios según crezca el throughput de la red.

---

### 7.2 Generar la clave del publicador de pruebas

> La clave del publicador (`PROVER_PUBLISHER_PRIVATE_KEY`) se usa para enviar las pruebas a L1. Esta cuenta necesita ETH para pagar el gas.

**Instalar Foundry** (si no lo tienes ya)

```bash
curl -L https://foundry.paradigm.xyz | bash
```

Más detalles en [getfoundry.sh](https://getfoundry.sh/).

**Generar una nueva clave privada de Ethereum con mnemotécnico de 24 palabras**

```bash
cast wallet new-mnemonic --words 24
```

> [!CAUTION]
> Guarda la frase mnemotécnica y la clave privada en un lugar seguro (gestor de contraseñas o almacenamiento offline). La clave privada se usará en `PROVER_PUBLISHER_PRIVATE_KEY` y la dirección derivada en `PROVER_ID`. Esta cuenta debe estar fondeada con ETH para pagar el gas de publicación de pruebas en L1.

---

### 7.3 Configurar el nodo Broker (Prover Node + Prover Broker)

> En la máquina que ejecutará el prover node y el broker (la "Broker Box").

**Paso 1 — Instalar dependencias del sistema**

```bash
sudo apt update && sudo apt upgrade -y && \
sudo apt install -y \
    build-essential \
    micro \
    libssl-dev \
    tar \
    unzip \
    wget \
    curl \
    git \
    jq \
    htop \
    ufw \
    net-tools \
    software-properties-common
```

**Paso 2 — Instalar Docker** (sigue el procedimiento de la sección [6.2](#62-instalar-docker-engine), idéntico para esta máquina)

**Paso 3 — Verificar la instalación**

```bash
docker --version
docker compose version
```

**Paso 4 — Crear la estructura de directorios**

```bash
mkdir -p aztec-prover-node/prover-node-data aztec-prover-node/prover-broker-data
cd aztec-prover-node
touch .env
```

**Paso 5 — Configurar las variables de entorno**

Añade a tu archivo `.env`:

```bash
# Configuración del Prover Node
DATA_DIRECTORY=./prover-node-data
P2P_IP=[tu IP externa]
P2P_PORT=40400
ETHEREUM_HOSTS=[tu endpoint de ejecución L1]
L1_CONSENSUS_HOST_URLS=[tu endpoint de consenso L1]
LOG_LEVEL=info
PROVER_BROKER_HOST=http://prover-broker:8080
PROVER_PUBLISHER_PRIVATE_KEY=[tu clave privada del publicador, ver sección 7.2]
AZTEC_PORT=8080
AZTEC_ADMIN_PORT=8880

# Configuración del Prover Broker
PROVER_BROKER_DATA_DIRECTORY=./prover-broker-data
PROVER_BROKER_PORT=8080
```

**Paso 6 — Crear el `docker-compose.yml`**

```bash
nano docker-compose.yml
```

```yaml
name: aztec-prover-node

services:
  prover-node:
    image: aztecprotocol/aztec:4.3.1
    entrypoint: >-
      node
      --no-warnings
      /usr/src/yarn-project/aztec/dest/bin/index.js
      start
      --prover-node
      --archiver
      --network mainnet
    depends_on:
      prover-broker:
        condition: service_started
        required: true
    environment:
      DATA_DIRECTORY: /var/lib/data
      ETHEREUM_HOSTS: ${ETHEREUM_HOSTS}
      L1_CONSENSUS_HOST_URLS: ${L1_CONSENSUS_HOST_URLS}
      LOG_LEVEL: ${LOG_LEVEL}
      PROVER_BROKER_HOST: ${PROVER_BROKER_HOST}
      PROVER_PUBLISHER_PRIVATE_KEY: ${PROVER_PUBLISHER_PRIVATE_KEY}
      P2P_IP: ${P2P_IP}
      P2P_PORT: ${P2P_PORT}
      AZTEC_PORT: ${AZTEC_PORT}
      AZTEC_ADMIN_PORT: ${AZTEC_ADMIN_PORT}
    ports:
      - ${AZTEC_PORT}:${AZTEC_PORT}
      - ${P2P_PORT}:${P2P_PORT}
      - ${P2P_PORT}:${P2P_PORT}/udp
    volumes:
      - ${DATA_DIRECTORY}:/var/lib/data
    restart: unless-stopped

  prover-broker:
    image: aztecprotocol/aztec:4.3.1
    entrypoint: >-
      node
      --no-warnings
      /usr/src/yarn-project/aztec/dest/bin/index.js
      start
      --prover-broker
      --network mainnet
    environment:
      DATA_DIRECTORY: /var/lib/data
      ETHEREUM_HOSTS: ${ETHEREUM_HOSTS}
      P2P_IP: ${P2P_IP}
      LOG_LEVEL: ${LOG_LEVEL}
    ports:
      - ${PROVER_BROKER_PORT}:8080
    volumes:
      - ${PROVER_BROKER_DATA_DIRECTORY}:/var/lib/data
    restart: unless-stopped
```

> [!IMPORTANT]
> **Seguridad — puerto admin no expuesto**: el puerto admin (8880) se omite intencionadamente del mapeo de `ports` por seguridad. La API admin permite operaciones sensibles (cambios de configuración, rollbacks de base de datos) que nunca deben ser accesibles desde fuera del contenedor. Si necesitas usarla, hazlo vía `docker exec`:
>
> ```bash
> docker exec -it prover-node curl -X POST http://localhost:8880 \
>   -H 'Content-Type: application/json' \
>   -d '{"jsonrpc":"2.0","method":"nodeAdmin_getConfig","params":[],"id":1}'
> ```
>
> El broker expone el puerto 8080 para que sea accesible desde las máquinas de los agentes — asegúrate de que ese puerto es alcanzable desde ellas.

**Paso 7 — Configurar el firewall**

```bash
sudo ufw allow 22/tcp
sudo ufw allow 8080/tcp
sudo ufw allow 40400/tcp
sudo ufw allow 40400/udp
sudo ufw enable
```

**Paso 8 — Iniciar el broker y el prover node**

```bash
docker compose up -d
```

---

### 7.4 Configurar los nodos Agent

> En cada máquina dedicada a ejecutar agentes de prueba (las máquinas de alto rendimiento — "Main Rig").

**Paso 1 — Instalar dependencias del sistema**

```bash
sudo apt update && sudo apt upgrade -y && \
sudo apt install -y \
    build-essential \
    micro \
    libssl-dev \
    tar \
    unzip \
    wget \
    curl \
    git \
    jq \
    htop \
    ufw \
    net-tools \
    software-properties-common
```

**Paso 2 — Instalar Docker** (sección [6.2](#62-instalar-docker-engine))

**Paso 3 — Verificar la instalación**

```bash
docker --version
docker compose version
```

**Paso 4 — Crear el directorio de trabajo**

```bash
mkdir aztec-prover-agent
cd aztec-prover-agent
touch .env
```

**Paso 5 — Configurar las variables de entorno**

> [!TIP]
> Ajusta `PROVER_AGENT_COUNT` según el hardware real de la máquina, siguiendo la tabla de escalado de la sección 7.1: 64 cores/256 GB → 2 agentes; 96 cores/384 GB → 3 agentes; 128 cores/512 GB → 4 agentes.

```bash
PROVER_AGENT_COUNT=1
PROVER_AGENT_POLL_INTERVAL_MS=10000
PROVER_BROKER_HOST=http://[IP_DE_LA_MAQUINA_BROKER]:8080
PROVER_ID=[dirección correspondiente a PROVER_PUBLISHER_PRIVATE_KEY]
```

Sustituye `[IP_DE_LA_MAQUINA_BROKER]` por la IP real de la máquina que ejecuta el broker.

**Prueba de conectividad antes de arrancar:**

```bash
curl http://[IP_DE_LA_MAQUINA_BROKER]:8080
```

Si falla, revisa la configuración de red, las reglas de firewall, y que el broker esté efectivamente en ejecución.

**Paso 6 — Crear el `docker-compose.yml`**

```bash
nano docker-compose.yml
```

```yaml
name: aztec-prover-agent

services:
  prover-agent:
    image: aztecprotocol/aztec:4.3.1
    entrypoint: >-
      node
      --no-warnings
      /usr/src/yarn-project/aztec/dest/bin/index.js
      start
      --prover-agent
      --network mainnet
    environment:
      PROVER_AGENT_COUNT: ${PROVER_AGENT_COUNT}
      PROVER_AGENT_POLL_INTERVAL_MS: ${PROVER_AGENT_POLL_INTERVAL_MS}
      PROVER_BROKER_HOST: ${PROVER_BROKER_HOST}
      PROVER_ID: ${PROVER_ID}
    restart: unless-stopped
```

**Paso 7 — Configurar el firewall**

```bash
sudo ufw allow 22/tcp
sudo ufw allow out 8080/tcp
sudo ufw enable
```

**Paso 8 — Iniciar el agente**

```bash
docker compose up -d
```

**Escalado de capacidad de prueba:**
- **Horizontal**: añade más máquinas de agentes repitiendo este setup.
- **Vertical**: incrementa `PROVER_AGENT_COUNT` en máquinas existentes (asegurando hardware suficiente).

---

### 7.5 Verificación y comandos de gestión

**Ver el estado de los contenedores (en cualquier máquina)**

```bash
docker compose ps
```

**Logs del Prover Node**

```bash
docker compose logs -f prover-node
```

**Logs del Broker**

```bash
docker compose logs -f prover-broker
```

**Logs de un Agent**

```bash
docker compose logs -f prover-agent
```

**Filtrar logs para ver pruebas enviadas correctamente**

```bash
docker compose logs prover-node 2>&1 | grep --line-buffered -E 'Submitted'
```

**Reiniciar el stack completo (broker o agente)**

```bash
docker compose down && docker compose up -d
```

---

### 7.6 Resolución de problemas habituales

**Problema: el agente no puede conectar con el broker**

- Verifica que la IP en `PROVER_BROKER_HOST` es correcta.
- Comprueba que el puerto 8080 de la máquina broker es accesible desde las máquinas agente.
- Revisa las reglas de firewall entre máquinas.
- Prueba la conectividad: `curl http://[IP_BROKER]:8080`
- Verifica que el contenedor del broker está corriendo: `docker compose ps`

**Problema: el agente se cae o rinde mal**

- Verifica que el hardware cumple los mínimos oficiales (32 cores y 128 GB RAM por agente).
- Comprueba el uso de recursos: `docker stats`
- Reduce `PROVER_AGENT_COUNT` si corres varios agentes en la misma máquina.

**Problema: el agente no recibe trabajos**

- Verifica que el broker está recibiendo trabajos del prover node.
- Confirma que `PROVER_ID` coincide con la dirección del publicador.
- Prueba la conectividad con el broker: `curl http://[IP_BROKER]:8080`

---

### 7.7 Herramientas adicionales de monitorización para provers

**Instalar `bpytop` (monitor de recursos del sistema con interfaz visual)**

```bash
sudo apt update
sudo apt install -y python3-pip
sudo pip3 install bpytop
```

**Ejecutar**

```bash
bpytop
```

---

### 7.8 Recursos oficiales y de comunidad

> [!NOTE]
> Los siguientes enlaces los aportaste tú; los marco como **comunidad** (no documentación oficial de Aztec Labs), salvo el primero que sí es la doc oficial.

- **Documentación oficial de Aztec**: [docs.aztec.network](https://docs.aztec.network)
- Estadísticas de provers (comunidad): [dune.com/rhum/aztec-nb-proofs-tx-new-rollup](https://dune.com/rhum/aztec-nb-proofs-tx-new-rollup)
- Bot de Telegram para provers (comunidad): [t.me/Aztec_prover_bot](https://t.me/Aztec_prover_bot)
- Explorador de provers (comunidad): [aztec-prover-client.vercel.app](https://aztec-prover-client.vercel.app/)
- Alquiler de cómputo (proveedor externo): [hostkey.com](https://hostkey.com/dedicated-servers/instant/)

---

<div align="center">

**📌 Repositorio de referencia personal — comandos verificados en Ubuntu 24.04 (Linux)**
**⟠ Sección de nodos blockchain verificada contra documentación oficial el 28 de junio de 2026 — comprobar siempre la versión vigente antes de desplegar en mainnet**

⭐ Si te resulta útil, considera dejar una estrella al repositorio

</div>
