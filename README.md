# Advanced-Guide-Prover-Set-up-on-Aztec
This guide provides a step-by-step, production-grade walkthrough for deploying and optimizing an Aztec Prover in a split-architecture configuration. Designed for operators, it covers everything from hardware selection and environment preparation to multi-server orchestration using Docker Compose.

----
# PLEASE TAKE NOTE! ! !  
## AZTEC SPECIFICALLY ANNOUCED YOU SHOULDN'T RUN NODE (PROVER/SEQUENCER) IN FOR AIRDROP EXPECTATIONS!
### RUN ONLY FOR THE TECH IF YOU CHOOSE TO OR SKIP!!! PASSING THAT INFO !

<img width="1024" height="262" alt="image" src="https://github.com/user-attachments/assets/1cb3ff4f-e1d0-49ea-b07a-58dc09098ea6" />

----
### The setup leverages a dedicated Broker + Prover Node server for coordination and proof publishing, and a high-performance agent rig for computation-heavy proving tasks. It includes:
  * Detailed hardware requirements and sizing recommendations for large-scale proving.
  * Docker Compose configurations for both broker and agent servers.
  * .env templates with clear variable descriptions.
  * Troubleshooting advice for common prover-node and agent issues.
### By following this guide, operators can maximize throughput, minimize downtime, and maintain a stable presence on the Aztec network, positioning themselves for optimal proof rewards in current and future network phases.

------
> **⚠️ Important Notes**
> 
> - **Do NOT use the same private key** for your Prover as you do for a Sequencer Node as this will cause **nonce conflicts** and failed transactions.
> - **Do NOT run a Sequencer Node and a Prover on the same server** — they use overlapping ports, which will result in **port binding conflicts**.
> - To be eligible for the **Prover Role**, you must complete at least **750 epochs**

------
## 1. Preparation and Requirements 
- [ ] Read the [Aztec Prover Docs](https://docs.aztec.network/the_aztec_network/guides/run_nodes/how_to_run_prover)
- [ ] You'll need **two servers**: (these are my specifications for optimal performance)
  - **Broker Box** : 48 cores, 256 GB RAM, NVMe
  - **Main Rig (Agents)** : 64 - 192 cores, 256 – 768 GB RAM, NVMe **( I used a 2x EPYC 96 Cores = 192 cores in total)**
- [ ] Take note of **public IP** for Broker NODE
- [ ] Open required ports on Broker Box:
  - `8081` (TCP)
  - `40400` (TCP & UDP)
- [ ] Install **Docker**, **Aztec CLI** & **Docker Compose v2** on both servers
- [ ] Generate a **Prover Publisher Private Key** and corresponding **Prover ID** (EVM address from private key)











------
## Getting Started 
## 🚀 Step 1 — Set Up the Broker NODE

This server will run **two roles**:
- **Prover Broker** → coordinates and assigns jobs to agents
- **Prover Node** → fetches jobs, generates partial proofs, and submits final proofs
---
### 1️⃣ Install Essentials
On the **Broker NODE** (48c/256 GB), update and install the required packages:

```
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

### Install dependencies
```
sudo apt install -y ca-certificates curl gnupg lsb-release
```

### Add Docker’s official GPG key
```
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### Set up repository
```
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install Docker Engine & Compose
```
sudo apt update && \
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Verify installation
```
docker --version
docker compose version
```

------

## 2️⃣ 📥 Install Aztec CLI
### Run the following command to install the **Aztec CLI** on your server:

```
bash -i <(curl -s https://install.aztec.network)
```

### Add the CLI to your shell’s PATH:
```
echo 'export PATH="$HOME/.aztec/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### You can verify installation with: 
```
aztec --version 
```

### Update AZTEC to latest 
```
aztec-up latest
```

## 3️⃣ Create Prover Directory & Configure Firewall
---
### Create the Prover Directory
This will hold your Docker Compose files, `.env`, and data volumes:
```
mkdir -p ~/broker
cd ~/broker
```

### Allow ports in UFW  and enable 
Open the necessary TCP and UDP ports for the broker, prover node, and SSH access:

```
sudo ufw allow 22/tcp     
sudo ufw allow 8080/tcp   
sudo ufw allow 8081/tcp     
sudo ufw allow 40400/tcp   
sudo ufw allow 40400/udp 
```

```
sudo ufw enable
```

### Create a .env file 
⚠️  **Important: Do NOT reuse the same private key as your Sequencer Node — this will cause nonce conflicts.**

```
nano .env
```

### Paste content in .env 
```
P2P_IP=
ETHEREUM_HOSTS=
L1_CONSENSUS_HOST_URLS=
PROVER_PUBLISHER_PRIVATE_KEY=
PROVER_ID=
```
`CTRL` + `X` select `Y` and ENTER

### Create a `docker-compose.yml` file 
### Creat a docker file with 
```
nano docker-compose.yml
```

### Paste content and save
```
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




## Step 2 — Set Up the Agents (Main Prover Rig) 
This server will run **Prover Agents** only, connecting to the Broker to receive proving jobs. **[Remember,this is the NODE with 256GB+ RAMS].**

---

### 1️⃣ Install Essentials
Update and install required packages:

```
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


### 2️⃣ Install Docker & Docker Compose 
```
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

### Verify installation
```
docker --version
docker compose version
```

### 3️⃣ Create Agents Directory & Configure Firewall
```
mkdir -p ~/agents
cd ~/agents
```

### Allow required ports for agent communication
```
sudo ufw allow 22/tcp    
sudo ufw allow out 8081/tcp
```

### 4️⃣ Create .env File
```
nano .env
```

### Paste content in .env 
```
PROVER_ID=
PROVER_BROKER_HOST=http://<broker_public_ip>:8081
BROKER_IP=IPAddressoftheBrokerNode
AGENT_COUNT=30
```
-----
### Agents `.env` Variables

- **`BROKER_IP`**  
  The public IP address of your **Broker** (where `prover-node` and `broker` are running).  
  Agents use this to locate the broker for job requests.

- **`AGENT_COUNT`**  
  The number of agent processes to run in parallel.  
  Example: On a 192-core server, 25-30 agents is optimal.

- **`PROVER_ID`**  
  The public prover address linked to your `PROVER_PUBLISHER_PRIVATE_KEY` on the broker.  


- **`PROVER_BROKER_HOST`**  
  The full connection URL to your broker, including the port.  
  Example:  http://<BROKER_IP>:8081
------










