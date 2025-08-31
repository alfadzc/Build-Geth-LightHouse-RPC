# ✨Build Geth-LightHouse RPC✨
- Build Privat RPC for Aztec Sequencer
- Step by step guide for setting up a systemd for running a Sepolia Ethereum Pruned mode using Geth as the execution client and Lighthouse as the consensus client on an Ubuntu-based system.

### **#Hardware Requirements**

<img width="555" height="129" alt="Hardware requirement" src="https://github.com/user-attachments/assets/e4d03b55-20f9-4bd1-9b26-4b149284eb72" />

## #Step 1. Update packages

```bash
sudo apt update && sudo apt-get upgrade -y
```

## #Step **2. Install Dependencies**

```bash
sudo apt install -y curl git unzip wget openssl
```

```bash
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev -y

```

## #Step 3. Install Geth

```bash
sudo add-apt-repository -y ppa:ethereum/ethereum
sudo apt update
sudo apt install -y geth
```

### #Step 4. Generate JWT secret

```bash
mkdir -p /root/data/geth
openssl rand -hex 32 | tr -d "\n" | sudo tee /root/data/geth/jwtsecret > /dev/null
sudo chmod 600 /root/data/geth/jwtsecret
```

### #Step 5. **Create Directories**

```bash
mkdir -p /root/data/geth
mkdir -p /root/data/lighthouse
```

### #Step 6.  CONFIGURE GETH Systemd service

- Create Geth service file

```bash
sudo nano /etc/systemd/system/geth.service
```

```bash
[Unit]
Description=Geth Sepolia Node (Optimized Snap Sync + AuthRPC)
After=network.target

[Service]
ExecStart=/usr/bin/geth \
  --sepolia \
  --syncmode=snap \
  --datadir /root/data/geth \
  --http \
  --http.addr=0.0.0.0 \
  --http.port=8545 \
  --http.api=eth,net,web3 \
  --http.vhosts="*" \
  --http.corsdomain="*" \
  --ws \
  --ws.addr=0.0.0.0 \
  --ws.port=8546 \
  --authrpc.addr=127.0.0.1 \
  --authrpc.port=8551 \
  --authrpc.vhosts="*" \
  --authrpc.jwtsecret=/root/data/geth/jwtsecret \
  --cache=4096 \
  --maxpeers=300
Restart=always
RestartSec=5
LimitNOFILE=8192

[Install]
WantedBy=default.target
```

### #Step 7. Run and start Geth

```bash
sudo systemctl daemon-reload
sudo systemctl enable geth
sudo systemctl start geth
sudo systemctl status geth.service
```

- Check Logs Geth

```bash
journalctl -u geth.service -n 50
```

```bash
journalctl -u geth.service -f
```

## #Step 8. Install LIGHTHOUSE Beacon node

### **#8.1. Install Dependencies**

```bash
sudo apt update
sudo apt install -y curl git build-essential pkg-config libssl-dev clang cmake
```

### #8.2. Install Rust & Cargo

```bash
curl https://sh.rustup.rs -sSf | sh
```

```bash
source $HOME/.cargo/env
```

- Select the default option (1) when the installation starts.
- Once complete, run

### #8.3. Clone Repository Lighthouse

```bash
git clone https://github.com/sigp/lighthouse.git
cd lighthouse 
```

### #8.4. Build Lighthouse

```bash
make
```

### #8.5. Copy Binary to /usr/local/bin

```bash
sudo cp ./target/release/lighthouse /usr/local/bin/
```

### **#8.6. Create Directories**

```bash
mkdir -p /root/data/lighthouse
sudo chown -R root:root /root/data/lighthouse
sudo chmod 755 /root/data/lighthouse
```

### #8.7. CONFIGURE LIGHTHOUSE Systemd service

- Create Lighthouse service  file

```bash
sudo nano /etc/systemd/system/lighthouse.service
```

```bash
[Unit]
Description=Lighthouse Beacon Node (Pruned + Connected to Geth)
Wants=network-online.target geth.service
After=network-online.target geth.service

[Service]
User=root
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/lighthouse bn \
  --network sepolia \
  --datadir /root/data/lighthouse \
  --http \
  --http-address 0.0.0.0 \
  --http-port 5052 \
  --execution-endpoint http://localhost:8551 \
  --jwt-secrets /root/data/geth/jwtsecret \
  --checkpoint-sync-url https://checkpoint-sync.sepolia.ethpandaops.io \

[Install]
WantedBy=multi-user.target
```

### #8.8 Run and start Lighthouse

```bash
sudo systemctl daemon-reload
sudo systemctl enable lighthouse
sudo systemctl start lighthouse
sudo systemctl status lighthouse.service
```

- Check logs Lighthouse

```bash
journalctl -u lighthouse -n 100 --no-pager
```

```bash
journalctl -u lighthouse -f
```

### #Check Geth & Lighthouse service

```bash
sudo systemctl status geth.service
sudo systemctl status lighthouse.service
```

## **#Step 9. Checking If Nodes are Synced**

- **➡️**Verify syncing status Geth

```bash
 curl -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'
```

- ✅Response Geth if fully synced:

```bash
{"jsonrpc":"2.0","id":1,"result":false}
```

<img width="742" height="173" alt="Screenshot_2" src="https://github.com/user-attachments/assets/5c08b512-6057-4784-869b-fe683f8e2d00" />


- 🚫Response Geth if still syncing:

```bash
{"jsonrpc":"2.0","id":1,"result":{"currentBlock":"0x1a2b3c","highestBlock":"0x1a2b4d","startingBlock":"0x0"}}
```

- You'll see an object with `startingBlock`, `currentBlock`, and `highestBlock`, indicating the sync progress.
- **➡️**Verify syncing status Lighthouse

```bash
curl http://localhost:5052/eth/v1/node/syncing
```

- ✅Response Lighthouse if fully synced

```bash
{"data":{"is_syncing":false,"is_optimistic":false,"el_offline":false,"head_slot":"7734746","sync_distance":"0"}}
```

- If `is_syncing` is `false` and `sync_distance` is `0`, the Lighthouse Beacon node is fully synced..
- Response Lighthouse  if still syncing

```bash
{"data":{"head_slot":"12345","sync_distance":"100","is_syncing":true}}
```

- If `is_syncing` is `true`, the node is still syncing, and `sync_distance` indicates how many slots behind it is.

## **#Step 10. Getting the RPC Endpoints**

- **Execution Node (Geth)**
- Geth provides an HTTP RPC endpoint for interacting with the execution layer of Ethereum. Based on `docker-compose.yml` setup, Geth exposes port `8545` for HTTP RPC. The endpoints are:
- Inside the VPS: `http://localhost:8545` for SEPOLIA_RPC_URL
- Inside the VPS: `http://localhost:5052` for BEACON_RPC_URL
- Outside the VPS: `http://<your-vps-ip>:8545` (replace `<your-vps-ip>` with your VPS’s public IP address, e.g., `http://204.0.112.6:8545`)

## #FAGs:

#Check Storage Geth and Lighthouse

```bash
du -sh /root/data/geth/geth/chaindata
```

```bash
du -sh /root/data/lighthouse
```

```bash
du -h /root | grep '[0-9\.]\+G' | sort -hr | head -20
```

## #⛔️Uninstall Geth & Lighthouse service

### #Stop Geth & Lighthouse service

```bash
sudo systemctl stop geth.service
sudo systemctl stop lighthouse.service
```

### #Disable Service (agar tidak auto-start saat reboot)

```bash
sudo systemctl disable geth.service
sudo systemctl disable lighthouse.service
```

### #**Remove File Service**

```bash
sudo rm /etc/systemd/system/geth.service
sudo rm /etc/systemd/system/lighthouse.service
```

### 🔄 **Reload systemd**

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
```

### #Check status Geth & Lighthouse service

```bash
systemctl status geth.service
systemctl status lighthouse.service
```

### #Check Directory data

```bash
ls -lh /root/data/geth
ls -lh /root/data/lighthouse
```

### #Remove Directory data

```bash
sudo rm -rf /root/data/geth/*
sudo rm -rf /root/data/lighthouse/*
```


