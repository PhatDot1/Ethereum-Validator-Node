# Ethereum Validator Node Setup (Holesky Testnet Example)

This guide walks you through setting up a **non-archival (full) Ethereum execution node** and **consensus node/validator** on a Linux (AMD/Intel) droplet. We use the Holesky testnet as our running example and focus on **Geth** (execution layer) and **Lighthouse** (consensus/validator client). Alternative clients (e.g., Prysm, Nimbus) are mentioned where appropriate.

For RPC node setup, refer to #RPC.md

---

## Table of Contents

1. [Prerequisites & System Requirements](#prerequisites--system-requirements)  
2. [Directory Structure & Environment Setup](#directory-structure--environment-setup)  
3. [Execution Layer: Geth (Holesky)](#execution-layer-geth-holesky)  
    - 3.1. Install Geth  
    - 3.2. Download & Initialize Holesky Genesis  
    - 3.3. Run Geth as a Service (systemd)  
    - 3.4. Bootnodes & Peer Connectivity  
    - 3.5. Monitoring Geth Sync Status  
4. [Consensus Layer: Lighthouse Beacon Node (Holesky)](#consensus-layer-lighthouse-beacon-node-holesky)  
    - 4.1. Install Lighthouse  
    - 4.2. Run Lighthouse Beacon Node  
    - 4.3. Lighthouse as a Service (systemd)  
    - 4.4. Monitoring Beacon Sync Status  
5. [Deposit CLI & Generating Validator Keys](#deposit-cli--generating-validator-keys)  
    - 5.1. Download & Verify Ethereum Staking Deposit CLI  
    - 5.2. Generate Validator Keys (Holesky)  
    - 5.3. Import Validator Keys into Lighthouse  
    - 5.4. Submitting the Deposit (Holesky Launchpad)  
6. [Validator Client: Lighthouse Validator (Holesky)](#validator-client-lighthouse-validator-holesky)  
    - 6.1. Preparing the Validator Wallet & Password File  
    - 6.2. Run Lighthouse Validator Client  
    - 6.3. Lighthouse Validator as a Service (systemd)  
    - 6.4. Monitoring Validator Status  
7. [Advanced Considerations & Tips](#advanced-considerations--tips)  
    - 7.1. Alternative Clients (Prysm, Nimbus, Teku)  
    - 7.2. Resource Usage (CPU, RAM, Disk)  
    - 7.3. Security & Firewall  
    - 7.4. Logs & Troubleshooting  
    - 7.5. RPC Node
    - 7.6. Rate Limiting & Auth
8. [Useful Links](#useful-links)  

---

## 1. Prerequisites & System Requirements

1. **Linux droplet/server** (e.g., Ubuntu 22.04 LTS or Debian 12).  
2. **Hardware**:  
   - **RAM**: ≥ 16 GB for mainnet; Holesky/testnets can run on 8 GB, but 16 GB recommended for future-proofing.  
   - **CPU**: ≥ 2–4 vCPUs (higher core-count improves block-processing performance).  
   - **Disk**: ≥ 500 GB SSD ( > 1 TB for long-term mainnet usage). Use at least NVMe or SSD.  
   - **Network**: Unmetered bandwidth with reliable uptime; open ports 30303 (P2P), 8545/8551 (HTTP/RPC), 9000 (beacon HTTP), 5052 (validator HTTP).  
3. **Root or sudo** privileges.  
4. **OpenSSL**, **curl**, **tar**, **git** installed (most distros include these by default).  
5. **Timezone**: (Optional) Set to UTC or your preferred zone. Example:  
   ```bash
   sudo timedatectl set-timezone Europe/London
   ```  

---

## 2. Directory Structure & Environment Setup

1. **Create a dedicated base directory** for Ethereum-related binaries/data. We will use `/home/eth-validator` (adjust to your username/path):  
   ```bash
   sudo mkdir -p /home/eth-validator
   sudo chown $USER:$USER /home/eth-validator
   cd /home/eth-validator
   ```  
2. **Subdirectories**:  
   ```text
   /home/eth-validator
   ├── geth/                # Geth data & genesis files
   ├── lighthouse/          # Lighthouse beacon-node data
   ├── validator/           # Staking-deposit-cli & validator-keys
   └── secrets/             # JWT secrets, passwords
   ```  
   Create them:  
   ```bash
   mkdir -p geth lighthouse validator secrets
   ```  
3. **Set secure permissions** for `secrets/`:  
   ```bash
   chmod 700 secrets
   ```  

---

## 3. Execution Layer: Geth (Holesky)

The **execution layer** (EL) is responsible for processing transactions, maintaining Ethereum state, and exposing JSON-RPC/Engine API for the consensus client.  

### 3.1. Install Geth

1. **Add Ethereum PPA (Ubuntu/Debian)**:  
   ```bash
   sudo apt update
   sudo apt install -y software-properties-common
   sudo add-apt-repository -y ppa:ethereum/ethereum
   sudo apt update
   ```  
2. **Install Geth**:  
   ```bash
   sudo apt install -y geth
   ```  
3. **Verify installation**:  
   ```bash
   geth version
   ```  
   You should see something like:  
   ```
   Geth
   Version: 1.13.x-stable
   Git Commit: ...
   Architecture: amd64
   Protocol Versions: [ ... ]
   Go Version: go1.20.x
   Operating System: linux
   GOPATH=
   GOBIN=
   ```  

### 3.2. Download & Initialize Holesky Genesis

Ethereum testnets (including Holesky) require a custom genesis file and chain configuration. Prysm clients typically consume an SSZ genesis, while Geth/Lighthouse prefer JSON.

1. **Fetch Holesky genesis JSON** into `/home/eth-validator/geth/`:  
   ```bash
   cd /home/eth-validator/geth
   curl -o holesky-genesis.json https://raw.githubusercontent.com/eth-clients/holesky/main/metadata/genesis.json
   ```  
2. **Initialize Geth with Holesky genesis**:  
   ```bash
   sudo geth init /home/eth-validator/geth/holesky-genesis.json --datadir /home/eth-validator/geth
   ```  
   - `--datadir /home/eth-validator/geth` tells Geth where to store chain data.  
   - The `networkid` for Holesky is `17000` (as defined in the genesis metadata).  

### 3.3. Run Geth as a Service (systemd)

To ensure **100% uptime** and automatic restart on reboot/failure, configure Geth as a `systemd` service.

1. **Generate an `authrpc` JWT secret** for EL⇄CL communication:  
   ```bash
   dd if=/dev/urandom bs=32 count=1 2>/dev/null | sha256sum | awk '{print $1}' | tee /home/eth-validator/secrets/jwt.hex
   chmod 600 /home/eth-validator/secrets/jwt.hex
   ```  
2. **Create the `geth.service` file**:  
   ```bash
   sudo tee /etc/systemd/system/geth.service > /dev/null << 'EOF'
   [Unit]
   Description=Geth Execution Layer for Ethereum (Holesky)
   After=network.target

   [Service]
   User=root
   Group=root
   Type=simple
   ExecStart=/usr/bin/geth        --datadir /home/eth-validator/geth        --networkid 17000        --http        --http.api "engine,eth,net,web3"        --http.addr 0.0.0.0        --http.port 8545        --http.corsdomain "*"        --authrpc.jwtsecret /home/eth-validator/secrets/jwt.hex        --syncmode "snap"        --allow-insecure-unlock
   Restart=always
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   EOF
   ```  
3. **Enable & Start Geth service**:  
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable geth.service
   sudo systemctl start geth.service
   ```  
4. **Verify service status**:  
   ```bash
   sudo systemctl status geth.service
   ```  

### 3.4. Bootnodes & Peer Connectivity

To accelerate peer discovery, you may manually configure bootnodes for Holesky in `geth`’s flags. Example bootnodes (copy into `ExecStart` if needed):

```text
--bootnodes enode://ac906289e4b7f12df423d654c5a962b6ebe5b3a74cc9e06292a85221f9a64a6f1cfdd6b714ed6dacef51578f92b34c60ee91e9ede9c7f8fadc4d347326d95e2b@146.190.13.128:30303,enode://a3435a0155a3e837c02f5e7f5662a2f1fbc25b48e4dc232016e1c51b544cb5b4510ef633ea3278c0e970fa8ad8141e2d4d0f9f95456c537ff05fdf9b31c15072@178.128.136.233:30303,enode://7fa09f1e8bb179ab5e73f45d3a7169a946e7b3de5ef5cea3a0d4546677e4435ee38baea4dd10b3ddfdc1f1c5e869052932af8b8aeb6f9738598ec4590d0b11a6@65.109.94.124:30303,enode://3524632a412f42dee4b9cc899b946912359bb20103d7596bddb9c8009e7683b7bff39ea20040b7ab64d23105d4eac932d86b930a605e632357504df800dba100@172.174.35.249:30303
```

**Tip**: If you add bootnodes, your `ExecStart` becomes:

```text
ExecStart=/usr/bin/geth     --datadir /home/eth-validator/geth     --networkid 17000     --bootnodes enode://...     --http --http.api "engine,eth,net,web3" --http.addr 0.0.0.0 --http.port 8545 --http.corsdomain "*"     --authrpc.jwtsecret /home/eth-validator/secrets/jwt.hex     --syncmode "snap" --allow-insecure-unlock
```

After editing, run `sudo systemctl daemon-reload && sudo systemctl restart geth.service`.

### 3.5. Monitoring Geth Sync Status

1. **Attach to Geth console** (optional):  
   ```bash
   geth attach http://127.0.0.1:8545
   ```  
   - Check syncing status:  
     ```js
     > eth.syncing
     ```
     - `false` means fully synced.  
   - Check current block:  
     ```js
     > eth.blockNumber
     ```
   - Check peer count:  
     ```js
     > net.peerCount
     ```
2. **View logs** with `journalctl` (live tail):  
   ```bash
   sudo journalctl -u geth.service -f
   ```  
3. **Disk usage & data**:  
   ```bash
   df -h /home/eth-validator/geth
   ```  

---

## 4. Consensus Layer: Lighthouse Beacon Node (Holesky)

The **consensus layer** (CL) is responsible for attesting, proposing blocks, and maintaining proof‐of‐stake consensus. We use **Lighthouse** (Rust-based client).

### 4.1. Install Lighthouse

1. **Download the latest Lighthouse release** for Linux x86_64 (example uses v6.0.0; check GitHub for newer versions):  
   ```bash
   cd /home/eth-validator
   curl -LO https://github.com/sigp/lighthouse/releases/download/v6.0.0/lighthouse-v6.0.0-x86_64-unknown-linux-gnu.tar.gz
   ```  
2. **Extract & move the binary**:  
   ```bash
   tar -xvzf lighthouse-v6.0.0-x86_64-unknown-linux-gnu.tar.gz
   sudo mv lighthouse /usr/local/bin/
   sudo chmod +x /usr/local/bin/lighthouse
   ```  
3. **Verify installation**:  
   ```bash
   lighthouse --version
   ```  
   You should see something like:  
   ```
   lighthouse 6.0.0
   ```  

### 4.2. Run Lighthouse Beacon Node

1. **Create Lighthouse data directory**:  
   ```bash
   mkdir -p /home/eth-validator/lighthouse
   ```  
2. **Run Beacon Node (interactive)**:  
   ```bash
   lighthouse bn      --network holesky      --execution-endpoint http://127.0.0.1:8551      --execution-jwt /home/eth-validator/secrets/jwt.hex      --datadir /home/eth-validator/lighthouse      --http      --http-address 0.0.0.0      --http-allow-origin '*'      --metrics      --metrics-address 0.0.0.0      --checkpoint-sync-url https://holesky.beaconstate.ethstaker.cc/
   ```
   - `--network holesky` selects the Holesky chain parameters.  
   - `--execution-endpoint http://127.0.0.1:8551` points to Geth’s Engine API (`geth` by default exposes Engine API on 8551).  
   - `--execution-jwt /home/eth-validator/secrets/jwt.hex` for authentication.  
   - `--http-address 0.0.0.0` & `--http-allow-origin '*'` let you query the Beacon Node REST API remotely if needed.  
   - `--metrics` exposes Prometheus metrics on port `5054` by default, and you can customize with `--metrics-address`.  
   - `--checkpoint-sync-url` (optional but speeds up syncing from trusted source).  

### 4.3. Lighthouse as a Service (systemd)

1. **Create the `lighthouse-beacon.service` file**:  
   ```bash
   sudo tee /etc/systemd/system/lighthouse-beacon.service > /dev/null << 'EOF'
   [Unit]
   Description=Lighthouse Beacon Node (Holesky)
   After=network.target geth.service

   [Service]
   User=root
   Group=root
   Type=simple
   Environment="PATH=/usr/local/bin:/usr/bin:/bin"
   ExecStart=/usr/local/bin/lighthouse bn        --network holesky        --execution-endpoint http://127.0.0.1:8551        --execution-jwt /home/eth-validator/secrets/jwt.hex        --datadir /home/eth-validator/lighthouse        --http        --http-address 0.0.0.0        --http-allow-origin '*'        --metrics        --metrics-address 0.0.0.0        --checkpoint-sync-url https://holesky.beaconstate.ethstaker.cc/
   Restart=always
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   EOF
   ```  
2. **Enable & Start Beacon Node**:  
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable lighthouse-beacon.service
   sudo systemctl start lighthouse-beacon.service
   ```  
3. **Verify status**:  
   ```bash
   sudo systemctl status lighthouse-beacon.service
   ```  

### 4.4. Monitoring Beacon Sync Status

1. **Beacon CLI commands** (attach to Lighthouse’s built-in HTTP API using `lighthouse bn local` or use Prometheus metrics). For simplicity:  
   ```bash
   lighthouse bn sync-status --network holesky --datadir /home/eth-validator/lighthouse
   ```
   - Displays whether the node is synced, head slot, finalized slot, etc.  

2. **View logs**:  
   ```bash
   sudo journalctl -u lighthouse-beacon.service -f
   ```  

3. **Metrics**: If you have a Prometheus/Grafana setup, scrape from `http://<server-ip>:5054/metrics` and build dashboards.  

---

Note: At this point you have set up the RPC node, proceed from here for full validator setup, skip to 7.5 for RPC node setup.

## 5. Deposit CLI & Generating Validator Keys

To become a validator, you must generate BLS keys and create deposit data for the Holesky launchpad (similar to mainnet workflow).

### 5.1. Download & Verify Ethereum Staking Deposit CLI

1. **Download the Linux AMD64 binary** (example uses v2.8.0):  
   ```bash
   cd /home/eth-validator/validator
   curl -LO https://github.com/ethereum/staking-deposit-cli/releases/download/v2.8.0/staking_deposit-cli-948d3fc-linux-amd64.tar.gz
   ```  
2. **(Optional) Verify checksum**:  
   ```bash
   sha256sum staking_deposit-cli-948d3fc-linux-amd64.tar.gz
   ```
   - Expected:  
     ```
     ef021252abd2591ef6d3558fb3258b35f478c20333f2dff4a17cc79b573c3879
     ```  
   - If they match, proceed.  
3. **Extract the tarball**:  
   ```bash
   tar -xzf staking_deposit-cli-948d3fc-linux-amd64.tar.gz
   ls -l staking_deposit-cli-948d3fc-linux-amd64
   ```
   - You should see a `deposit` binary. If not, check extraction.  

### 5.2. Generate Validator Keys (Holesky)

1. **Run the deposit CLI** to generate 1 validator key (adjust `--num_validators` as needed):  
   ```bash
   cd staking_deposit-cli-948d3fc-linux-amd64
   ./deposit new-mnemonic --chain holesky --num_validators 1
   ```  
2. **Follow the interactive prompts**:  
   - Choose the directory to save keys (default `./validator_keys/`).  
   - Backup your mnemonic/seed phrase carefully.  
   - Set a strong password to encrypt your keystore.  
   - Choose to generate exactly `1` validator (Holesky requires `32 ETH` equivalent in testnet tokens).  
3. **After completion**, you will see a prompt for your generated keys:  
   ```text
   ┌────────────────────────────────────────────┐
   │ Validator Keys Generated & Saved in:      │
   │ /home/eth-validator/validator/.../keys   │
   └────────────────────────────────────────────┘

   Deposit JSON file(s):
   deposit_data-XXXXXXXX.json
   ```
4. **Locate the deposit data file** (e.g., `deposit_data-1733140047.json`):  
   ```bash
   ls -l /home/eth-validator/validator/staking_deposit-cli-948d3fc-linux-amd64/validator_keys
   ```

### 5.3. Import Validator Keys into Lighthouse

1. **Import the validator keystore** (after deposit is confirmed on-chain, OR earlier if you want to see the key in your local Lighthouse keystore directory):  
   ```bash
   lighthouse account validator import      --directory /home/eth-validator/validator/staking_deposit-cli-948d3fc-linux-amd64/validator_keys      --datadir /home/eth-validator/lighthouse      --network holesky
   ```
   - This command copies the encrypted keystore JSONs to Lighthouse’s validator directory.  
2. **List imported validator public keys**:  
   ```bash
   lighthouse account validator list      --datadir /home/eth-validator/lighthouse      --network holesky
   ```
   - Ensure your validator’s public key (`0x…`) appears.  

### 5.4. Submitting the Deposit (Holesky Launchpad)

1. **Copy the deposit JSON** to your local machine (optional, if you prefer to use a browser on your laptop to submit; otherwise, `wget`/`curl` can be used). Example with `scp`:  
   ```bash
   scp root@<droplet-ip>:/home/eth-validator/validator/staking_deposit-cli-948d3fc-linux-amd64/validator_keys/deposit_data-1733140047.json .
   ```
2. **Go to the Holesky Launchpad** (e.g., `https://launchpad.holesky.ethstaker.cc` or official Holesky staking page).  
3. **Follow deposit instructions**:  
   - Upload `deposit_data-1733140047.json`.  
   - Confirm the deposit (requires Holesky testnet tokens).  
   - Wait for deposit confirmation.  
4. **Verify on-chain**: After a few epochs, check that your validator is `active` using a testnet block explorer or `lighthouse bn sync-status`.  

---

## 6. Validator Client: Lighthouse Validator (Holesky)

Once your beacon node is synced and your deposit is activated, spin up the **validator client (VC)**.

### 6.1. Preparing the Validator Wallet & Password File

1. **Create a password file** containing your keystore passphrase:  
   ```bash
   echo "YourStrongPassphraseHere" > /home/eth-validator/secrets/validator-password.txt
   chmod 600 /home/eth-validator/secrets/validator-password.txt
   ```  
2. **Ensure Lighthouse knows where to find your validator directory**:  
   - Lighthouse default layout:  
     ```
     /home/eth-validator/lighthouse
     ├── beacon_db/
     ├── validators/             # (automatically created after import)
     └── secrets/
          └── validator-password.txt
     ```  

### 6.2. Run Lighthouse Validator Client

1. **Interactive run** (test only; for production, use systemd below):  
   ```bash
   lighthouse vc      --network holesky      --datadir /home/eth-validator/lighthouse      --validators-dir /home/eth-validator/validator/staking_deposit-cli-948d3fc-linux-amd64/validator_keys      --unencrypted-http-transport      --http      --http-address 0.0.0.0      --metrics      --metrics-address 0.0.0.0      --graffiti "MyHoleskyValidator"      --suggested-fee-recipient 0xa341b0F69359482862Ed4422c6057cd59560D9E4      --password-file /home/eth-validator/secrets/validator-password.txt
   ```  
   - `--validators-dir` points to where the decrypted keystore JSONs live (post-import).  
   - `--password-file` tells Lighthouse where to read your keystore passphrase.  
   - `--graffiti` is optional metadata tag for block proposals.  
   - `--suggested-fee-recipient` designates an optional MEV fee recipient address (Holesky-specific).  

2. **Check that your validator is recognized** (in another terminal):  
   ```bash
   lighthouse vc list      --network holesky      --datadir /home/eth-validator/lighthouse
   ```
   - Shows each validator’s public key, status (e.g., `pending`, `active`).  

### 6.3. Lighthouse Validator as a Service (systemd)

To maintain uptime, set up the validator as a `systemd` service:

1. **Create `lighthouse-validator.service`**:  
   ```bash
   sudo tee /etc/systemd/system/lighthouse-validator.service > /dev/null << 'EOF'
   [Unit]
   Description=Lighthouse Validator Client (Holesky)
   After=network.target lighthouse-beacon.service

   [Service]
   User=root
   Group=root
   Type=simple
   Environment="PATH=/usr/local/bin:/usr/bin:/bin"
   ExecStart=/usr/local/bin/lighthouse vc        --network holesky        --datadir /home/eth-validator/lighthouse        --validators-dir /home/eth-validator/validator/staking_deposit-cli-948d3fc-linux-amd64/validator_keys        --unencrypted-http-transport        --http        --http-address 0.0.0.0        --metrics        --metrics-address 0.0.0.0        --graffiti "MyHoleskyValidator"        --suggested-fee-recipient 0xa341b0F69359482862Ed4422c6057cd59560D9E4        --password-file /home/eth-validator/secrets/validator-password.txt
   Restart=always
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   EOF
   ```  
2. **Enable & start**:  
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable lighthouse-validator.service
   sudo systemctl start lighthouse-validator.service
   ```  
3. **Verify status**:  
   ```bash
   sudo systemctl status lighthouse-validator.service
   ```  

### 6.4. Monitoring Validator Status

1. **List active validators**:  
   ```bash
   lighthouse vc list      --network holesky      --datadir /home/eth-validator/lighthouse
   ```  
   - Check each validator’s status (e.g., `pending`, `active`, `slashed`).  
2. **View Validator Logs**:  
   ```bash
   sudo journalctl -u lighthouse-validator.service -f
   ```  
3. **Prometheus Metrics**: Default port `5062` exposes metrics for validator duties. Scrape it if you have monitoring.  
4. **Check Beacon Node Finalization & Sync**:  
   ```bash
   lighthouse bn sync-status --network holesky --datadir /home/eth-validator/lighthouse
   ```  
5. **Check Peer Count (EL & CL)**:  
   - **EL**: `geth attach http://127.0.0.1:8545` → `net.peerCount`  
   - **CL (Beacon)**:  
     ```bash
     lighthouse bn peers --network holesky --datadir /home/eth-validator/lighthouse
     ```  

---

## 7. Advanced Considerations & Tips

### 7.1. Alternative Clients

- **Prysm** (Go; requires SSZ genesis blob):  
  - Download `prysm.sh`, run `prysm.sh beacon-chain` with `--genesis-state` or `--genesis-beacon-api-url`.  
  - Validator via `prysm.sh validator accounts import`.  
- **Nimbus** (Nim-based; lightweight)  
- **Teku** (Java-based; enterprise)  
- **Lodestar** (TypeScript-based)  

All clients adhere to the same EL⇄CL Engine API specifications on mainnet and testnets. Choose based on language preference, resource footprint, and community support.

### 7.2. Resource Usage

- **Execution Layer (Geth)**:  
  - Disk: ~ 400 GB (Holesky is smaller, but mainnet grows > 1 TB).  
  - RAM: ~ 8–12 GB (syncing).  
- **Consensus Layer (Lighthouse)**:  
  - Disk: ~ 50 GB (store state & chain data).  
  - RAM: ~ 4–8 GB.  
- **Validator Client**:  
  - Minimal (< 1 GB) but CPU spikes occur during block-production windows.  

If your droplet has only 8 GB RAM, consider running Lighthouse with `--minimal-config`, or prune states occasionally (not recommended for production).

### 7.3. Security & Firewall

- **Linux Firewall** (`ufw` or `iptables`):  
  - Allow inbound:
    - **8545/tcp** (Geth HTTP) – restrict to your IP if possible.  
    - **8551/tcp** (Geth Engine API) – restrict to localhost or CL host.  
    - **9000–9002/tcp** (Lighthouse HTTP/metrics) – restrict to trusted sources.  
  - Allow **30303/tcp, udp** (Geth P2P).  
- **SSH**: Use key‐based authentication; disable password login.  
- **Regular Updates**: Keep Ubuntu/Debian updated (`sudo apt upgrade`).  
- **Permissions**: Keep `secrets/` locked down (`chmod 600`); validator keystore JSONs should also be owned by root and not world‐readable.  

### 7.4. Logs & Troubleshooting

- **Geth logs**:  
  ```bash
  sudo journalctl -u geth.service --since "1 hour ago"
  ```
- **Lighthouse Beacon logs**:  
  ```bash
  sudo journalctl -u lighthouse-beacon.service --since "1 hour ago"
  ```
- **Lighthouse Validator logs**:  
  ```bash
  sudo journalctl -u lighthouse-validator.service --since "1 hour ago"
  ```
- **Common Errors**:  
  - **JWT mismatch**: Ensure `--authrpc.jwtsecret` path in Geth matches `--execution-jwt` path in Lighthouse BN.  
  - **No peers**: Confirm open P2P ports, add bootnodes, and networkid flag matches genesis.  
  - **Deposit not found**: Check chain explorers for your deposit transaction; ensure deposit was submitted to the correct chain (Holesky vs. mainnet).  
  - **Validator stuck in `pending`**: Wait for at least one epoch after block finalization; check `lighthouse bn sync-status` to confirm beacon is fully synced and finalized.  

---

### 7.5. RPC Setup [Optional]

If, instead of a validator, you want to set up an RPC node, follow the instructions below:

To configure your existing Geth node to serve as a secure HTTPS RPC endpoint using Nginx as a reverse proxy and Let's Encrypt for SSL certificates.

---
---

## 1. Install Nginx

```bash
sudo apt update
sudo apt install nginx
```

## 2. Create Nginx Configuration

1. Create a new Nginx site file:

    ```bash
    sudo nano /etc/nginx/sites-available/eth-holesky
    ```

2. Add the following content (replace `eth-holesky.rpc.encode.club` with your domain/subdomain):

    ```nginx
    server {
        listen 80;
        server_name eth-holesky.rpc.encode.club;

        location / {
            proxy_pass http://127.0.0.1:8545;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Increase buffer and timeout settings for large RPC requests
            client_max_body_size 10M;
            proxy_read_timeout 90s;
            proxy_send_timeout 90s;
        }
    }
    ```

3. Enable the configuration:

    ```bash
    sudo ln -s /etc/nginx/sites-available/eth-holesky /etc/nginx/sites-enabled/
    ```

4. Check syntax and restart Nginx:

    ```bash
    sudo nginx -t
    sudo systemctl restart nginx
    ```

## 3. Obtain SSL Certificates with Certbot

1. Install Certbot and the Nginx plugin:

    ```bash
    sudo apt install certbot python3-certbot-nginx
    ```

2. Prepare the webroot for ACME challenges (optional if Nginx already serves `/.well-known`):

    ```bash
    sudo mkdir -p /var/www/html/.well-known/acme-challenge
    sudo chown -R www-data:www-data /var/www/html
    ```

3. Request a certificate for your domain:

    ```bash
    sudo certbot --nginx -d eth-holesky.rpc.encode.club
    ```

4. Verify HTTPS:

    ```bash
    curl -I https://eth-holesky-rpc.encode.club
    ```

5. Automate renewal (Certbot sets up a systemd timer by default):

    ```bash
    sudo systemctl list-timers | grep certbot
    ```

6. (Optional) Test renewal manually:

    ```bash
    sudo certbot renew --dry-run
    ```

7. Restart Nginx after renewal if needed:

    ```bash
    sudo systemctl restart nginx
    ```

8. Confirm HTTP redirects to HTTPS:

    ```bash
    curl -I http://eth-holesky-rpc.encode.club
    ```

## 4. Configure Geth to Serve RPC Locally

Ensure your `geth` service binds the HTTP-RPC to localhost only. Modify your Geth startup command (e.g., in `/etc/systemd/system/geth.service`):

```ini
ExecStart=/usr/bin/geth     --datadir /home/eth-validator/geth     --networkid 17000     --http     --http.addr "127.0.0.1"     --http.port "8545"     --http.api "eth,web3,net"     --authrpc.jwtsecret /home/eth-validator/secrets/jwt.hex     --syncmode "snap"     --allow-insecure-unlock
```

- `--http`: Enables the HTTP-RPC server.  
- `--http.addr "127.0.0.1"`: Binds the RPC to localhost (secure).  
- `--http.port "8545"`: The port that Nginx proxies to.  
- `--http.api "eth,web3,net"`: Restricts RPC API exposure to only required methods.

After modifying, reload and restart the Geth service:

```bash
sudo systemctl daemon-reload
sudo systemctl restart geth.service
```

## 5. Test Local and Public RPC Endpoints

1. **Local test** (direct to Geth):

    ```bash
    curl -X POST --data '{"jsonrpc":"2.0","method":"web3_clientVersion","params":[],"id":1}' http://127.0.0.1:8545
    ```

2. **Public test** (through Nginx + HTTPS):

    ```bash
    curl -X POST --data '{"jsonrpc":"2.0","method":"web3_clientVersion","params":[],"id":1}' https://eth-holesky-rpc.encode.club
    ```

---

### 7.5. RPC Rate Limiting and Auth

After setting up a secure HTTPS RPC endpoint using Nginx and Certbot, you may want to further harden and control access. Consider the following advanced layers:

1. **Rate Limiting**  
   - Use Nginx’s `limit_req` directives to throttle excessive requests. Example within your server block:
     ```nginx
     http {
         limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;
         ...
         server {
             ...
             location / {
                 limit_req zone=one burst=20 nodelay;
                 proxy_pass http://127.0.0.1:8545;
                 ...
             }
         }
     }
     ```
   - This limits each IP to 10 requests per second with a small burst allowance, helping mitigate DDoS or brute-force attempts.

2. **Basic Authentication**  
   - Enable Nginx HTTP Basic Auth to protect the endpoint. Create a password file:
     ```bash
     sudo apt install apache2-utils
     sudo htpasswd -c /etc/nginx/.htpasswd username
     ```
   - In your Nginx server block:
     ```nginx
     location / {
         auth_basic "Restricted";
         auth_basic_user_file /etc/nginx/.htpasswd;
         proxy_pass http://127.0.0.1:8545;
         ...
     }
     ```

3. **JWT or API Key Validation**  
   - For more granular access control, layer an application or middleware (e.g., a small Go/Python proxy) in front of Nginx to validate JWTs or API keys. Only allow requests with valid tokens to reach the Ethereum node.

4. **Firewall & Network ACLs**  
   - Use UFW or iptables to restrict access to the Nginx port (443), allowing only trusted IP ranges or known clients.

5. **Monitoring & Logging**  
   - Enable and parse Nginx access logs (`/var/log/nginx/access.log`) to detect unusual patterns.  
   - Use Prometheus exporters or third-party services to track request metrics and raise alerts.


## 8. Useful Links

- **Holesky Testnet Resources**  
  - Holesky Genesis Data: https://github.com/eth-clients/holesky  
  - Holesky Launchpad (Staking): https://launchpad.holesky.ethstaker.cc  
  - Holesky Beaconstate (Checkpoint Sync): https://holesky.beaconstate.ethstaker.cc/  
- **Clients & Documentation**  
  - **Geth**: https://geth.ethereum.org/docs  
  - **Lighthouse**: https://docs.sigp.io/lighthouse  
  - **Prysm**: https://docs.prylabs.network  
  - **Nimbus**: https://nimbus.guide  
  - **Teku**: https://docs.teku.consensys.net  
- **Staking Deposit CLI**:  
  - GitHub Releases: https://github.com/ethereum/staking-deposit-cli/releases  
- **Ethereum Network IDs**:  
  - Mainnet: `1`  
  - Holesky: `17000`  
  - Other Testnets: Goerli (`5`), Sepolia (`11155111`), etc.  

---

- Guide adapted from personal notes and official Ethereum client documentation.  

---
