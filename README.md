# Advanced-Guide-Prover-Set-up-on-Aztec
This guide provides a step-by-step, production-grade walkthrough for deploying and optimizing an Aztec Prover in a split-architecture configuration. Designed for operators, it covers everything from hardware selection and environment preparation to multi-server orchestration using Docker Compose.

----
# PLEASE TAKE NOTE! ! !  
## AZTEC SPECIFICALLY ANNOUCED YOU SHOULDN'T RUN NODE (PROVER/SEQUENCER) IN ANTICIPATION TO BE REWARDED. 
### RUN ONLY FOR THE TECH IF YOU CHOOSE TO!!! PASSING THAT INFO ! 


----
### The setup leverages a dedicated Broker + Prover Node server for coordination and proof publishing, and a high-performance agent rig for computation-heavy proving tasks. It includes:
  * Detailed hardware requirements and sizing recommendations for large-scale proving.
  * Docker Compose configurations for both broker and agent servers.
  * .env templates with clear variable descriptions.
  * Troubleshooting advice for common prover-node and agent issues.
### By following this guide, operators can maximize throughput, minimize downtime, and maintain a stable presence on the Aztec network, positioning themselves for optimal proof rewards in current and future network phases.

------
## 1. Preparation and Requirements 
- [ ] Read the [Aztec Prover Docs](https://docs.aztec.network/the_aztec_network/guides/run_nodes/how_to_run_prover)
- [ ] You'll need **two servers**: (these are my specifications for optimal performance)
  - **Broker Box** : 48–64 cores, 256 GB RAM, NVMe
  - **Main Rig (Agents)** : 64 - 192 cores, 256 – 768 GB RAM, NVMe **( I used a 2x EPYC 96 Cores = 192 cores in total)**
- [ ] Take note of **public IP** for Broker NODE
- [ ] Open required ports on Broker Box:
  - `8081` (TCP)
  - `40400` (TCP & UDP)
- [ ] Install **Docker**, **Aztec CLI** & **Docker Compose v2** on both servers
- [ ] Generate a **Prover Publisher Private Key** and corresponding **Prover ID** (EVM address from private key)
