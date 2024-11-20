Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.23.1"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export XION_CHAIN_ID="xion-testnet-1"" >> $HOME/.bash_profile
echo "export XION_PORT="55"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
wget -O xiond https://github.com/burnt-labs/xion/releases/download/v13.0.1/xiond-linux-amd64
chmod +x $HOME/xiond
mv $HOME/xiond $HOME/go/bin/xiond
```

**config and init app**
```
xiond init $MONIKER --chain-id $XION_CHAIN_ID
xiond config set client chain-id $XION_CHAIN_ID
xiond config set client node tcp://localhost:${XION_PORT}657
sed -i -e "s|^node *=.*|node = \"tcp://localhost:${XION_PORT}657\"|" $HOME/.xiond/config/client.toml
```

**download genesis and addrbook**
```
wget -O $HOME/.xiond/config/genesis.json https://server-5.itrocket.net/testnet/burnt/genesis.json
wget -O $HOME/.xiond/config/addrbook.json  https://server-5.itrocket.net/testnet/burnt/addrbook.json
```

**set seeds and peers**
```
SEEDS="69e1aa5800ffa82615986eac5f99b77c2b8f1ccb@burnt-testnet-seed.itrocket.net:55656"
PEERS="a386af218bd4e5d0a5f2dcfbcc1051eff63d059f@burnt-testnet-peer.itrocket.net:55656,eda838e8e0a162667a6d1b9a304f06d4996b6c97@[2001:41d0:203:e4db::5]:26656,f10f5001726e95c62a763f2d020df6b53b19f44a@65.108.134.47:22356,a921118b0ada2c220833e63f648a9e58fcef19ef@65.108.71.140:26756,65e8c0dd01f486121dbd355e406e57492fea9106@15.235.87.88:56656,85e868567ca46f8d94b1fba87b2fa5b42a271439@141.94.240.117:22356,009335a23ee0971519af088e6931a69bbd9e681d@5.9.107.249:15256,f28fcb2d8d4c9c4388e5fdee5b4206fbe5d645f4@144.76.28.47:15256,cf8caf0b8d1cfdde6ea6d497d273639c19767ad5@190.2.149.38:26656,8c810d034caaccfcfc9f6bd57e40f6443609abf1@162.55.130.164:46656,7c0f6b24f3920dcb4a6c37e689d101fd68a09556@78.46.40.246:39656,81b6f3ef9529ed1a8f3ac2c1f848a19566a88baa@65.109.112.144:1020"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.xiond/config/config.toml
```

**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${XION_PORT}317%g;
s%:8080%:${XION_PORT}080%g;
s%:9090%:${XION_PORT}090%g;
s%:9091%:${XION_PORT}091%g;
s%:8545%:${XION_PORT}545%g;
s%:8546%:${XION_PORT}546%g;
s%:6065%:${XION_PORT}065%g" $HOME/.xiond/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${XION_PORT}658%g;
s%:26657%:${XION_PORT}657%g;
s%:6060%:${XION_PORT}060%g;
s%:26656%:${XION_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${XION_PORT}656\"%;
s%:26660%:${XION_PORT}660%g" $HOME/.xiond/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.xiond/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.xiond/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.xiond/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0uxion"|g' $HOME/.xiond/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.xiond/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.xiond/config/config.toml
```

**create service file**
```
sudo tee /etc/systemd/system/xiond.service > /dev/null <<EOF
[Unit]
Description=Burnt node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.xiond
ExecStart=$(which xiond) start --home $HOME/.xiond
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**reset and download snapshot**
```
xiond tendermint unsafe-reset-all --home $HOME/.xiond
if curl -s --head curl https://server-5.itrocket.net/testnet/burnt/burnt_2024-10-30_10572786_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-5.itrocket.net/testnet/burnt/burnt_2024-10-30_10572786_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.xiond
    else
  echo "no snapshot found"
fi
```

**enable and start service**
```
sudo systemctl daemon-reload
sudo systemctl enable xiond
sudo systemctl restart xiond && sudo journalctl -u xiond -f
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/burnt/autoinstall/)
```

**Create wallet**
to create a new wallet, use the following command. don’t forget to save the mnemonic
```
xiond keys add $WALLET
```

**to restore exexuting wallet, use the following command**
```
xiond keys add $WALLET --recover
```

**save wallet and validator address**
```
WALLET_ADDRESS=$(xiond keys show $WALLET -a)
VALOPER_ADDRESS=$(xiond keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**check sync status, once your node is fully synced, the output from above will print "false"**
```
xiond status 2>&1 | jq 
```

# before creating a validator, you need to fund your wallet and check balance
xiond query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.xiond/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://burnt-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  if [ "$blocks_left" -lt 0 ]; then
    blocks_left=0
  fi

  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"

  sleep 5
done
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, uxion
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
cd $HOME
# Create validator.json file
echo "{\"pubkey\":{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"$(xiond comet show-validator | grep -Po '\"key\":\s*\"\K[^"]*')\"},
    \"amount\": \"1000000uxion\",
    \"moniker\": \"test\",
    \"identity\": \"\",
    \"website\": \"\",
    \"security\": \"\",
    \"details\": \"I love blockchain ❤️\",
    \"commission-rate\": \"0.1\",
    \"commission-max-rate\": \"0.2\",
    \"commission-max-change-rate\": \"0.01\",
    \"min-self-delegation\": \"1\"
}" > validator.json
# Create a validator using the JSON configuration
xiond tx staking create-validator validator.json \
    --from $WALLET \
    --chain-id xion-testnet-1 \
	--gas auto --gas-adjustment 1.5
	
Monitoring
If you want to have set up a monitoring and alert system use our cosmos nodes monitoring guide with tenderduty

Security
To protect you keys please don`t share your privkey, mnemonic and follow basic security rules

Set up ssh keys for authentication
You can use this guide to configure ssh authentication and disable password authentication on your server

Firewall security
Set the default to allow outgoing connections, deny all incoming, allow ssh and node p2p port

sudo ufw default allow outgoing 
sudo ufw default deny incoming 
sudo ufw allow ssh/tcp 
sudo ufw allow ${XION_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop xiond
sudo systemctl disable xiond
sudo rm -rf /etc/systemd/system/xiond.service
sudo rm $(which xiond)
sudo rm -rf $HOME/.xiond
sed -i "/XION_/d" $HOME/.bash_profile
