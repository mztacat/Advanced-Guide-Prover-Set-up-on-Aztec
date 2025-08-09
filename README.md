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



## 🖥 Recommended Hostkey Server Specs

---
### ⚙️ Agent Node
- [ ] **CPU:** EPYC 9354 (128 Cores - 200+ / 3.25 GHz)  
- [ ] **RAM:** 256GB+  
- [ ] **Storage:** 2×2 TB NVMe SSD  
- [ ] **Price Range:** €600–€800/month  

**Why we chose this:**  
- Designed for **heavy parallel proving** with high per-core speed  
- 256+ GB RAM supports **25–30 prover agents** without memory bottlenecks.  
- Fast NVMe storage ensures quick read/write for large proof data.  
<img width="4020" height="2250" alt="image" src="https://github.com/user-attachments/assets/9f83ad6e-c9ef-4cab-8f6f-112a9eaf14ae" />


### ⚙️ Prover + Broker
- [ ] **CPU:** 2×EPYC 7451 (48 Cores / 2.3 GHz)  
- [ ] **RAM:** 256 GB  
- [ ] **Storage:** 2×960 GB SSD  
- [ ] **Price Range:** €160–€250/month  

**Why we chose this:**  
- Meets the **48–64 core** recommendation for broker + prover-node tasks.  
- 256 GB RAM is sufficient for coordination, archiving, and proof publishing.  
- SSD storage is reliable for broker data and state storage.  
- Affordable option compared to higher-tier EPYC plans, keeping costs low while meeting performance needs.

<img width="3076" height="2556" alt="image" src="https://github.com/user-attachments/assets/32889a7c-0de9-4ebe-8fa7-3f6a5289bd4a" />

### 💰 Total Monthly Cost for me (if you're able to get the 750 Epoch in a month, then you claim ROLE), with my setup, I get ~40 epoch/day
- **Agent Node:** €800/month  
- **Prover + Broker:** €260/month  
- **Total:** **€1,060/month**

**General Range for Others:** €760–€1,050/month depending on chosen hardware.


> **Note:**  
> I use **Hostkey** because they’re reliable and offer instant-deploy EPYC servers at competitive rates.  
> You can also consider **Hetzner**, **OVH**, or **Leaseweb** as long as they meet the CPU, RAM, storage, and network requirements above.
-------

## 🏅 How to Get the Prover Role on Discord

To qualify for the **Prover Role**, you must:

- [ ] **Run a Prover node** that has successfully submitted **750+ epochs**.
- [ ] Contact **@dinhdat | Mod** on the Aztec Discord via DM with the following:
  - [ ] **Wallet Address** (same one used by your prover node)
  - [ ] **Discord Username**
  - [ ] **Transaction Hash** proving wallet ownership

### 📜 How to Generate the Transaction Hash

1. Use your **Prover wallet** (the one running on your broker) on the **Sepolia network**.
2. Send **0 ETH** **to yourself** (same address as sender and recipient).
3. In the transaction’s **message/memo** (data field), include YourDiscordUsername + WalletAddress
e.g `mztacat_ 0xAddress 
4. You can do this easily in [Rabby Wallet](https://rabby.io/) by enabling the **"Add Data"** option.
5. Once the transaction is confirmed, copy the **Transaction Hash** from [Sepolia Etherscan](https://sepolia.etherscan.io/).
6. Send that hash to the mod along with your wallet address and Discord username.

---
> **Tip:** Make sure your prover is actively running and has completed **at least 750 epochs** before requesting the role


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

`CTRL` + `X` select `Y` and ENTER



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
PROVER_ID=0xAddress
PROVER_BROKER_HOST=http://<broker_public_ip>
BROKER_IP=IPAddressoftheBrokerNode
AGENT_COUNT=30
```

`CTRL` + `X` select `Y` and ENTER

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


### Paste on the docker compose of Agent 
```
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

`CTRL` + `X` select `Y` and ENTER



## ▶️ Run & Manage

### Then, run the Prover + Broker (the first one) 
```
docker compose up -d
```

### Now go to the Agent NODE and also run that too 
```
docker compose up -d
```

### Stop and remove containers (with volumes)
```
docker compose down -v
```

### Restart 
```
docker compose down -v && docker compose up -d
```

------
## Useful Commands for Prover+Broker

### Monitor the Prover logs 
```
docker logs -f aztec-prover-prover-node-1
```


### Check submitted proofs
```
docker logs -f aztec-prover-prover-node-1 2>&1 | grep --line-buffered -E 'Submitted'
```

<img width="4020" height="1084" alt="image" src="https://github.com/user-attachments/assets/66eea99c-fd2a-4564-8afe-64d44b012ae6" />

-------

## Useful Command for Agents 

### See container status
```
docker ps
```
<img width="4072" height="324" alt="image" src="https://github.com/user-attachments/assets/20401bbd-6247-4c49-ab03-7621ac1ea624" />


### Optional: Stop and remove containers (with volumes)
```
docker compose down -v
```

### Restart 
```
docker compose down -v && docker compose up -d
```

### Monitor Agent logs 
```
docker logs -f aztec-agent-only-agent-1
```

### Check agent-broker connection & job processing 
```
docker logs -f aztec-agent-only-agent-1 2>&1 | grep --line-buffered -E 'Connected to broker|Received job|Starting job|Submitting result'
```

<img width="4020" height="1076" alt="image" src="https://github.com/user-attachments/assets/277b3d54-2818-4628-94f5-1cfcf06e517c" />


# Check Address on Sepolia Scan  -  https://sepolia.etherscan.io/

In the **Transactions** tab, look for:
- **Method**: `0xc38f2a6d` (Aztec proof submission method)
- **Value**: `0 ETH`
- **Txn Fee**: ~0.0000034 ETH
- Regular intervals (proofs being submitted)
<img width="4020" height="1236" alt="image" src="https://github.com/user-attachments/assets/20519836-88bd-420c-82d1-70f1049075b8" />

-------

## 🛠 Extra Tooling for Monitoring
It’s recommended to install some additional tools for monitoring your prover and agent servers.

---

### Install `bpytop` (system resource monitor)
```
sudo apt update
sudo apt install -y python3-pip
sudo pip3 install bpytop
```
### RUN
```
bpytop
```
<img width="4020" height="1824" alt="image" src="https://github.com/user-attachments/assets/c494d1bd-5f70-494c-b2f6-379e6a38ec8a" />

**What it shows:**
- [ ] CPU usage per core
- [ ] RAM usage
- [ ] Network activity
- [ ] Active processes

**This is useful for checking:**
- [ ] If your `AGENT_COUNT` is fully utilizing CPU
- [ ] If memory usage is approaching limits
- [ ] If network traffic matches expected proof job transfers


----
##  More Info & Resources

- **Aztec Discord:** [https://discord.gg/dz4NS6pbJW](https://discord.gg/dz4NS6pbJW)  
- **Sepolia Etherscan:** [https://sepolia.etherscan.io/](https://sepolia.etherscan.io/)  
- **Rabby Wallet:** [https://rabby.io/](https://rabby.io/)  
- **My Twitter:** [https://twitter.com/mztacat](https://twitter.com/mztacat)  
- **My Discord:** `mztacat_`
