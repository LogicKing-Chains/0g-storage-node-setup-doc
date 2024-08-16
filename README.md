# 0g Storage Node Setup Guide

## System Requirements

| CPU | MEMORY | STORAGE |
|----|----|----|
| 4 Cores | 16GB RAM | 1TB NVME |


### Update and Install dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git jq build-essential gcc unzip wget lz4 -y
```


## Install go

```bash
cd $HOME && \
ver="1.22.4" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile && \
source ~/.bash_profile && \
go version
```


## Install rustup

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

## Clone the repository
```bash
git clone -b v0.3.1 https://github.com/0glabs/0g-storage-node.git
```

## Build Storage Node
In this step, you need to wait for the installation to complete.
```bash
cd $HOME/0g-storage-node
git submodule update --init
sudo apt install cargo
cargo build --release
```

## Create and Use your own RPC

```bash
nano $HOME/.0gchain/config/app.toml
```
Change 127.0.0.1:8545 to 0.0.0.0:8545.

Change api = "eth,net,web3" to api = "eth,txpool,personal,net,debug,web3"


Open port `8545`
```bash
sudo ufw allow 8545/tcp
```

## Setup Variables

Replace `http://x.x.x.x:8545` with your RPC endpoint 

```bash
ENR_ADDRESS=$(wget -qO- eth0.me)

echo "export ENR_ADDRESS=${ENR_ADDRESS}" >> ~/.bash_profile
echo 'export LOG_CONTRACT_ADDRESS="0xb8F03061969da6Ad38f0a4a9f8a86bE71dA3c8E7"' >> ~/.bash_profile
echo 'export MINE_CONTRACT="0x96D90AAcb2D5Ab5C69c1c351B0a0F105aae490bE"' >> ~/.bash_profile
echo 'export ZGS_LOG_SYNC_BLOCK="334797"' >> ~/.bash_profile
echo 'export BLOCKCHAIN_RPC_ENDPOINT="http://x.x.x.x:8545"' >> ~/.bash_profile

source ~/.bash_profile

echo -e "\n\033[31mCHECK YOUR VARIABLES\033[0m\n\nENR_ADDRESS: $ENR_ADDRESS\n\n\nLOG_CONTRACT_ADDRESS: $LOG_CONTRACT_ADDRESS\nMINE_CONTRACT: $MINE_CONTRACT\nZGS_LOG_SYNC_BLOCK: $ZGS_LOG_SYNC_BLOCK\nBLOCKCHAIN_RPC_ENDPOINT: $BLOCKCHAIN_RPC_ENDPOINT\n\n\033[33mwith love.\033[0m"
```

## Extract and Store Private Key

```bash
PRIVATE_KEY=$(0gchaind keys unsafe-export-eth-key $WALLET_NAME)
sed -i 's|^miner_key = ""|miner_key = "'"$PRIVATE_KEY"'"|' $HOME/0g-storage-node/run/config.toml
```
When running a validator node, ensure you create the wallet with the - eth option to be able to export the private key.

Example:
```bash
0gchaind keys add $WALLET_NAME --eth
```
Check your configuration ensure your settings are correct:
```bash
echo $BLOCKCHAIN_RPC_ENDPOINT
echo $LOG_CONTRACT_ADDRESS
echo $PRIVATE_KEY
```

## Update your config.toml
```bash
sed -i '
s|^\s*#\?\s*network_dir\s*=.*|network_dir = "network"|
s|^\s*#\?\s*network_enr_address\s*=.*|network_enr_address = "'"$ENR_ADDRESS"'"|
s|^\s*#\?\s*network_enr_tcp_port\s*=.*|network_enr_tcp_port = 1234|
s|^\s*#\?\s*network_enr_udp_port\s*=.*|network_enr_udp_port = 1234|
s|^\s*#\?\s*network_libp2p_port\s*=.*|network_libp2p_port = 1234|
s|^\s*#\?\s*network_discovery_port\s*=.*|network_discovery_port = 1234|
s|^\s*#\?\s*rpc_enabled\s*=.*|rpc_enabled = true|
s|^\s*#\?\s*db_dir\s*=.*|db_dir = "db"|
s|^\s*#\?\s*log_config_file\s*=.*|log_config_file = "log_config"|
s|^\s*#\?\s*log_directory\s*=.*|log_directory = "log"|
s|^\s*#\?\s*network_boot_nodes\s*=.*|network_boot_nodes = \["/ip4/54.219.26.22/udp/1234/p2p/16Uiu2HAmTVDGNhkHD98zDnJxQWu3i1FL1aFYeh9wiQTNu4pDCgps","/ip4/52.52.127.117/udp/1234/p2p/16Uiu2HAkzRjxK2gorngB1Xq84qDrT4hSVznYDHj6BkbaE4SGx9oS"\]|
s|^\s*#\?\s*log_contract_address\s*=.*|log_contract_address = "'"$LOG_CONTRACT_ADDRESS"'"|
s|^\s*#\?\s*mine_contract_address\s*=.*|mine_contract_address = "'"$MINE_CONTRACT"'"|
s|^\s*#\?\s*log_sync_start_block_number\s*=.*|log_sync_start_block_number = '"$ZGS_LOG_SYNC_BLOCK"'|
s|^\s*#\?\s*blockchain_rpc_endpoint\s*=.*|blockchain_rpc_endpoint = "'"$BLOCKCHAIN_RPC_ENDPOINT"'"|
s|^\s*miner_id\s*=\s*""|# miner_id = ""|
' $HOME/0g-storage-node/run/config.toml
```

## Create service file
```bash
sudo tee /etc/systemd/system/zgs.service > /dev/null <<EOF
[Unit]
Description=ZGS Node
After=network.target

[Service]
User=$USER
WorkingDirectory=$HOME/0g-storage-node/run
ExecStart=$HOME/0g-storage-node/target/release/zgs_node --config $HOME/0g-storage-node/run/config.toml
Restart=on-failure
RestartSec=10
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

## Start service
```bash
sudo systemctl daemon-reload && \
sudo systemctl enable zgs && \
sudo systemctl start zgs
```
