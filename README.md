# Gnoland Testnet Node Installation Guide (`test11`)

## 📋 Hardware Requirements

| Component | Minimum | Recommended |
| :--- | :--- | :--- |
| **CPU** | 4 Cores | 8 Cores |
| **RAM** | 8 GB | 16 GB |
| **SSD** | 200 GB NVMe | 500 GB NVMe |
| **OS** | Ubuntu 22.04 | Ubuntu 22.04 |

## 🛠 Installation Steps

### 1\. System Update & Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl unzip clang pkg-config libssl-dev jq build-essential tar wget bsdmainutils git make ncdu gcc htop tmux chrony liblz4-tool fail2ban lz4 aria2 -y
```

### 2\. Install Go

```bash
cd $HOME
VER="1.23.8"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
mkdir -p ~/go/bin
```

### 3\. Install Binary & Cosmovisor

We use **Cosmovisor** to ensure the node remains stable during upgrades.

```bash
# Install Cosmovisor
go install github.com/cosmos/cosmos-sdk/cosmovisor/cmd/cosmovisor@latest

# Build Gnoland
cd $HOME
rm -rf gno
git clone https://github.com/gnolang/gno.git
cd gno
git checkout chain/test11
make -C gno.land install.gnoland && make -C contribs/gnogenesis install && make install_gnokey

# Prepare Cosmovisor structure
mkdir -p $HOME/.gnoland/cosmovisor/genesis/bin
mkdir -p $HOME/.gnoland/cosmovisor/upgrades
cp $(which gnoland) $HOME/.gnoland/cosmovisor/genesis/bin/
```

### 4\. Initialize & Configuration

```bash
gnoland secrets init && gnoland config init

# Custom Port & Node Settings
gnoland config set rpc.laddr tcp://0.0.0.0:42657
gnoland config set p2p.laddr tcp://0.0.0.0:42656
gnoland config set moniker "Your_Moniker"
gnoland config set p2p.persistent_peers g10l8g7l5u0ahysgj0jczkvwdvct506khfrvf2yz@gnolan-testnet-rpc.shazoes.xyz:42656,g1vgvqg94xy8qj23dc8zpw6wns7q0hj9g8mx03ha@gno-core-sen-01.test11.testnets.gno.land:26656

# Genesis
wget -O $HOME/gnoland-data/config/genesis.json https://files.shazoes.xyz/testnets/gnoland/genesis.json
```

### 5\. Create Systemd Service (Cosmovisor)

```bash
sudo tee /etc/systemd/system/gnoland.service > /dev/null <<EOF
[Unit]
Description=Gnoland Node (Cosmovisor)
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start --genesis $HOME/gnoland-data/config/genesis.json --data-dir $HOME/gnoland-data --skip-genesis-sig-verification
Restart=on-failure
RestartSec=3
LimitNOFILE=65535
Environment="DAEMON_NAME=gnoland"
Environment="DAEMON_HOME=$HOME/.gnoland"
Environment="DAEMON_ALLOW_STRUCTURAL_ADDITIONS=true"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable gnoland
```

### 6\. Snapshot (Optional)

```bash
sudo systemctl stop gnoland
rm -rf $HOME/gnoland-data/db $HOME/gnoland-data/wal
aria2c -x 8 -s 8 https://snapshot.shazoes.xyz/testnets/snapshot-gnoland.tar.lz4 
lz4 -c -d snapshot-gnoland.tar.lz4 | tar -x -C $HOME/gnoland-data 
rm snapshot-gnoland.tar.lz4

sudo systemctl start gnoland && sudo journalctl -fu gnoland -o cat
```
Special Thanks Shazoes

-----

## 🔑 Wallet & Validator Operations

### Wallet Management

  * **Create:** `gnokey add wallet`
  * **Restore:** `gnokey add wallet --recover`
  * **Check Balance:** `gnokey query -remote https://rpc.test11.testnets.gno.land/ auth/accounts/$ADDRESS`

### Validator Registration

First, export your metadata:

```bash
ADDRESS="Your_Wallet_Address"
RPC="https://rpc.test11.testnets.gno.land/"
VALOPER=$(gnoland secrets get validator_key | jq -r '.address')
PUBKEY=$(gnoland secrets get validator_key | jq -r '.pub_key')
```

**Register:**

```bash
gnokey maketx call -pkgpath "gno.land/r/gnops/valopers" -func "Register" \
  -args "Your_Moniker" -args "Description" -args "$VALOPER" -args "$PUBKEY" \
  -gas-fee 1000000ugnot -gas-wanted 30000000 -send "" $ADDRESS > call.tx

# Sign and Broadcast
gnokey sign -tx-path call.tx -chainid "test11" -account-number [NUM] -account-sequence [SEQ] $ADDRESS
gnokey broadcast -remote $RPC call.tx
```

-----

## 📊 Useful Commands

  * **Log Check:** `journalctl -u gnoland -fo cat`
  * **Service Restart:** `sudo systemctl restart gnoland`
  * **Sync Status:** `gnoland status 2>&1 | jq`
