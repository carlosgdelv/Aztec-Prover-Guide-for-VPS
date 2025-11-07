# Aztec-Prover-Guide-for-VPS
Step-by-step deployment and operation guide for running Aztec provers on a VPS.

___

- Prover Stats: https://dune.com/rhum/aztec-nb-proofs-tx-new-rollup 

- Telegram Prover bot: https://t.me/Aztec_prover_bot?start=_tgr_u7UubixkOGQ0…

- Principal Guide: https://github.com/mztacat/Advanced-Guide-Prover-Set-up-on-Aztec?tab=readme-ov-file

- Second guide: https://github.com/Jaytechent/Aztec-Prover-Node-Guide?tab=readme-ov-file 

- Aztec Prover Explorer: https://aztec-prover-client.vercel.app/ 

- Computation rent: https://hostkey.com/dedicated-servers/instant/ 

___









___

# ⟠  Eth-Prysm-node Mainnet
Step by step guide for setting up a `docker-compose.yml` for running a `Sepolia` Ethereum full node using **Geth** as the `execution client` and **Prysm** as the `consensus client` on an Ubuntu-based system.

___

## Step 1. 🔧 Install Dependecies
**Packages:**
Refreshes the local package index and then upgrades all installed packages to their latest available versions without prompting for confirmation.
```bash
sudo apt-get update && sudo apt-get upgrade -y
```
This command installs a wide range of essential development tools, system utilities, and libraries required to build, run, and manage blockchain nodes and Docker-based environments on a Linux system.
```bash
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev  -y
```
**Docker:**
The system is first updated and any previous Docker-related packages are removed to prevent conflicts. Then, necessary certificates and tools like `gnupg` are installed to securely add Docker’s official GPG key. A trusted keyring directory is created, and Docker’s repository is added to the system’s sources list. After updating the package list again, the official Docker Engine, CLI, and plugins are installed from Docker’s repository. Finally, Docker is tested with `hello-world`, enabled to start on boot, and restarted to ensure it runs properly.
```bash
sudo apt update -y && sudo apt upgrade -y
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y && sudo apt upgrade -y

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Test Docker
sudo docker run hello-world

sudo systemctl enable docker
sudo systemctl restart docker
```


---

## Step 2. 👤➕🐳 Add Your User to the Docker Group

It must show "carlos"
```bash
whoami
```  
It must show "1000"
```bash
id -u
```
It must show "1000"
```bash
id -g        
```
It should say: groupadd: the group "docker" already exists
```bash
sudo groupadd docker
```
The following command adds the user carlos to the docker group so they can use Docker without sudo
```bash
sudo usermod -aG docker carlos
```
Restart the session or the system
```bash
reboot
```


## Step 3. 📁 Create Directories
These commands create the necessary directory structure for Ethereum's execution and consensus clients under the user's home directory (`~/ethereum`):
```bash
mkdir -p ~/ethereum-mainnet/execution ~/ethereum-mainnet/consensus
```

---

## Step 4. 🔐 Generate the JWT secret:
Generates a 32-byte random JWT secret in hexadecimal format and saves it to a file used for secure communication between clients.

```bash
openssl rand -hex 32 > ~/ethereum-mainnet/jwt.hex
```

Protect the JWT secret with strict permissions
```bash
chmod 600 /home/carlos/ethereum-mainnet/jwt.hex
```

Prints the contents of the jwt.hex file to verify it was correctly generated.
```bash
cat ~/ethereum-mainnet/jwt.hex
```

---

## Step 5. 🐳 Configure `docker-compose.yml`
Changes the current working directory to the `ethereum` folder where you will place the `docker-compose.yml ` configuration.
```bash
cd ~/ethereum
```
Opens a new or existing `docker-compose.yml` file in the Nano text editor to write or edit the service definitions.
```bash
nano docker-compose.yml
```
Replace the following code into your `docker-compose.yml` file:
```yaml
services:
  geth:
    image: ethereum/client-go:v1.16.4
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
    image: gcr.io/prysmaticlabs/prysm/beacon-chain:v6.1.2
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
      - --min-sync-peers=3
      - --subscribe-all-data-subnets
      - --checkpoint-sync-url=https://mainnet.checkpoint.sigp.io
      - --genesis-beacon-api-url=https://mainnet.checkpoint.sigp.io
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```


___

## Step 6. ▶️ Run Geth & Prysm Nodes

▶️ Start Geth & Prysm Nodes:
Starts the Geth and Prysm containers in detached mode (running in the background).
```bash
docker compose up -d
```

🧾 Node Logs
Continuously displays the real-time logs from both containers.
```bash
docker compose logs -f
```
⛔ Stop node. 
Run `docker compose down` to stop and remove all running Aztec containers before updating.
```bash
docker compose down
```
## Step 7. 🔥 UFW


✅ Aplicar reglas UFW:

```bash
# 1. Allow SSH access (only if you use SSH, otherwise omit)
sudo ufw allow OpenSSH

# 2. Allow P2P traffic required for Geth P2P, Geth P2P, Prysm P2P, Prysm gossip/block sync

sudo ufw allow 30303/tcp      
sudo ufw allow 30303/udp      
sudo ufw allow 13000/tcp     
sudo ufw allow 12000/udp      

# 3. Default policy: deny incoming traffic
sudo ufw default deny incoming

# 4. Allow outgoing traffic
sudo ufw default allow outgoing

# 5. Enable the firewall
sudo ufw enable

```

Used Ports 
```bash
sudo ss -tulnp
```
UFW rules
```bash
sudo ufw status verbose
```

___

## Step 8. 🔄 Checking If Nodes are Synced
**Execution Node (Geth)**
```
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' http://localhost:8545
```
✅Response if fully synced:
```json
{"jsonrpc":"2.0","id":1,"result":false}
```
🚫Response if still syncing:
```json
{"jsonrpc":"2.0","id":1,"result":{"currentBlock":"0x1a2b3c","highestBlock":"0x1a2b4d","startingBlock":"0x0"}}
```
You'll see an object with `startingBlock`, `currentBlock`, and `highestBlock`, indicating the sync progress.

**Beacon Node (Prysm)**
```bash
curl http://localhost:3500/eth/v1/node/syncing
```
✅Response if fully synced:
```json
{"data":{"head_slot":"12345","sync_distance":"0","is_syncing":false}}
```
If `is_syncing` is `false` and `sync_distance` is `0`, the beacon node is fully synced.

🚫Response if still syncing:
```json
{"data":{"head_slot":"12345","sync_distance":"100","is_syncing":true}}
```
If `is_syncing` is `true`, the node is still syncing, and `sync_distance` indicates how many slots behind it is.


Split-architecture configuration.
   
You'll need two servers: (these are my specifications for optimal performance)
Broker Box : 48 cores, 256 GB RAM, NVMe
Main Rig (Agents) : 64 - 192 cores, 256 – 768 GB RAM, NVMe ( I used a 2x EPYC 96 Cores = 192 cores in total)

Open required ports on Broker Box:
8081 (TCP)
40400 (TCP & UDP)


___

#   🚀 Step 1. Set Up the Broker Node
This server will run two roles:

Prover Broker → coordinates and assigns jobs to agents
Prover Node → fetches jobs, generates partial proofs, and submits final proofs


___


### 🧱 Install Essentials

On the Broker NODE (48c/256 GB), update and install the required packages:
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

### 📦 Install dependencies
```bash
sudo apt install -y ca-certificates curl gnupg lsb-release
```

### 🔐 Add Docker’s official GPG key

```bash
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
### 🗂️ Set up repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 🐳 Install Docker Engine & Compose
```bash
sudo apt update && \
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```


### ✅ Verify installation
```bash
docker --version
docker compose version
```

___

##  2️⃣ 📥 Install Aztec CLI

### Run the following command to install the Aztec CLI on your server:

```bash
bash -i <(curl -s https://install.aztec.network)
```
### Add the CLI to your shell’s PATH:
```bash
echo 'export PATH="$HOME/.aztec/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```
### You can verify installation with:
```bash
aztec --version 
```
### Update AZTEC to latest
```bash
aztec-up latest
```

___

##  3️⃣ 📁 Create Prover Directory & Configure Firewall

### 📁 Create the Prover Directory

This will hold your Docker Compose files, ".env", and data volumes:
```bash
mkdir -p ~/broker
cd ~/broker
```
### 🔓Allow ports in UFW and enable
Open the necessary TCP and UDP ports for the broker, prover node, and SSH access:

```bash
sudo ufw allow 22/tcp     
sudo ufw allow 8080/tcp   
sudo ufw allow 8081/tcp     
sudo ufw allow 40400/tcp   
sudo ufw allow 40400/udp 
```
```bash
sudo ufw enable
```
### 🧾 Create a .env file

⚠️ Important: Do NOT reuse the same private key as your Sequencer Node — this will cause nonce conflicts.

```bash
nano .env
```
### 📋 Paste content in .env

```bash
P2P_IP=
ETHEREUM_HOSTS=
L1_CONSENSUS_HOST_URLS=
PROVER_PUBLISHER_PRIVATE_KEY=
PROVER_ID=
```
### 🐳 Create a docker-compose.yml file

### Create a docker file with
```bash
nano docker-compose.yml
```

### 💾 Paste content and save
```bash
name: aztec-prover
services:
  broker:
    image: aztecprotocol/aztec:latest
    command:
      - node
      - --no-warnings
      - /usr/src/yarn-project/aztec/dest/bin/index.js
      - start
      - --prover-broker
      - --network
      - alpha-testnet
    environment:
      DATA_DIRECTORY: /data-broker
      LOG_LEVEL: info
      ETHEREUM_HOSTS: ${ETHEREUM_HOSTS}
      P2P_IP: ${P2P_IP}
    volumes:
      - ./data-broker:/data-broker
    ports:
      - "8081:8080"   # Expose broker's internal 8080 as external 8081
    entrypoint: >
      sh -c 'node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js start --network alpha-testnet --prover-broker'

  prover-node:
    image: aztecprotocol/aztec:latest
    depends_on:
      - broker
    environment:
      P2P_ENABLED: "true"
      DATA_DIRECTORY: /data-prover
      P2P_IP: ${P2P_IP}
      DATA_STORE_MAP_SIZE_KB: "134217728"
      ETHEREUM_HOSTS: ${ETHEREUM_HOSTS}
      L1_CONSENSUS_HOST_URLS: ${L1_CONSENSUS_HOST_URLS}
      LOG_LEVEL: info
      PROVER_BROKER_HOST: http://broker:8080  # within docker network
      PROVER_PUBLISHER_PRIVATE_KEY: ${PROVER_PUBLISHER_PRIVATE_KEY}
    volumes:
      - ./data-prover:/data-prover
    ports:
      - "40400:40400"
      - "40400:40400/udp"
    entrypoint: >
      sh -c 'node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js start --network alpha-testnet --archiver --prover-node'
```

___

# 🧠 Step 2 — Set Up the Agents (Main Prover Rig)
 
This server will run Prover Agents only, connecting to the Broker to receive proving jobs. [Remember,this is the NODE with 256GB+ RAMS]. 

### 1️⃣ Install Essentials

Update and install required packages:

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
### 2️⃣ 🐳 Install Docker & Docker Compose


```bash
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg


echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null


sudo apt update && \
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### ✅ Verify installation

```bash
docker --version
docker compose version
```

### 3️⃣ Create Agents Directory & Configure Firewall

```bash
mkdir -p ~/agents
cd ~/agents
```

### 🔥 Allow required ports for agent communication

```bash
sudo ufw allow 22/tcp    
sudo ufw allow out 8081/tcp
```


### 4️⃣ Create .env File

```bash
nano .env
```

### Paste content in .env

```bash
PROVER_ID=0xAddress
PROVER_BROKER_HOST=http://<broker_public_ip>
BROKER_IP=IPAddressoftheBrokerNode
AGENT_COUNT=30
```


___


### Agents ".env" Variables

- "BROKER_IP"
The public IP address of your Broker (where prover-node and broker are running).
Agents use this to locate the broker for job requests.


- "AGENT_COUNT"
The number of agent processes to run in parallel.
Example: On a 192-core server, 25-30 agents is optimal.

- "PROVER_ID"
The public prover address linked to your PROVER_PUBLISHER_PRIVATE_KEY on the broker.

- "PROVER_BROKER_HOST"
The full connection URL to your broker, including the port.
Example: http://<BROKER_IP>:8081

___


### 🐳Create a docker file with

```bash
nano docker-compose.yml
```

### 🐳 Paste on the docker compose of Agent
```bash
name: aztec-agent-only
services:
  agent:
    image: aztecprotocol/aztec:latest
    command:
      - node
      - --no-warnings
      - /usr/src/yarn-project/aztec/dest/bin/index.js
      - start
      - --prover-agent
      - --network
      - alpha-testnet
    environment:
      PROVER_AGENT_COUNT: ${AGENT_COUNT:-30}
      PROVER_AGENT_POLL_INTERVAL_MS: "10000"
      PROVER_BROKER_HOST: http://${BROKER_IP}:8081
      PROVER_ID: ${PROVER_ID}
    volumes:
      - ./data-prover:/data-prover
    entrypoint: >
      sh -c 'node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js start --network alpha-testnet --prover-agent'

```

___

## 🚀 Run & Manage

### ▶️ Then, run the Prover + Broker (the first one)

```bash
docker compose up -d
```

### ▶️ Now go to the Agent NODE and also run that too
```bash
docker compose up -d
```
### ⛔ Stop and remove containers (with volumes)

```bash
docker compose down -v
```

### 🔄 Restart

```bash
docker compose down -v && docker compose up -d
```

___

## 🔧 Useful Commands for Prover+Broker

### 🧾Monitor the Prover logs

```bash
docker logs -f aztec-prover-prover-node-1
```
### ✅ Check submitted proofs

```bash
docker logs -f aztec-prover-prover-node-1 2>&1 | grep --line-buffered -E 'Submitted'
```

## 🧰 Useful Command for Agents

### 📊 See container status

```bash
docker ps
```

### ⛔ Optional: Stop and remove containers (with volumes)
```bash
docker compose down -v
```
### 🔄 Restart
```bash
docker compose down -v && docker compose up -d
```
### 🧾 Monitor Agent logs
```bash
docker logs -f aztec-agent-only-agent-1
```
### ✅ Check agent-broker connection & job processing

```bash
docker logs -f aztec-agent-only-agent-1 2>&1 | grep --line-buffered -E 'Connected to broker|Received job|Starting job|Submitting result'
```

___


## 🛠 Extra Tooling for Monitoring

It’s recommended to install some additional tools for monitoring your prover and agent servers.

### 📈Install "bpytop" (system resource monitor)

```bash
sudo apt update
sudo apt install -y python3-pip
sudo pip3 install bpytop
```

### ▶️ Run

```bash
bpytop
```













