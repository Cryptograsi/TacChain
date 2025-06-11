# TacChain

### Sistem Gereksinimleri
```
CPU	4 Cores
RAM	8+ GB
Storage	400 GB SSD
Go	Above or 1.23.6
```

## Required Installations
```
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip clang cmake -y
sudo apt -qy upgrade
```

## Go Installation
```
cd $HOME
curl -LO https://go.dev/dl/go1.23.6.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.23.6.linux-amd64.tar.gz
rm go1.23.6.linux-amd64.tar.gz
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

## Build tacchaind v0.0.11
```
cd $HOME
rm -rf tacchain
git clone https://github.com/TacBuild/tacchain.git
cd tacchain
git checkout v0.0.11
make build
```

## Setup Cosmovisor Directory Structure
```
mkdir -p $HOME/.tacchaind/cosmovisor/genesis/bin
cp build/tacchaind $HOME/.tacchaind/cosmovisor/genesis/bin/

mkdir -p $HOME/.tacchaind/cosmovisor/upgrades/v0.0.11/bin
cp build/tacchaind $HOME/.tacchaind/cosmovisor/upgrades/v0.0.11/bin/
chmod +x $HOME/.tacchaind/cosmovisor/upgrades/v0.0.11/bin/tacchaind
```

## Symlink Cosmovisor Binary
```
sudo ln -sfn $HOME/.tacchaind/cosmovisor/genesis $HOME/.tacchaind/cosmovisor/current
sudo ln -sfn $HOME/.tacchaind/cosmovisor/current/bin/tacchaind /usr/local/bin/tacchaind
```

## Install Cosmovisor
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0
echo 'export PATH=$PATH:$(go env GOPATH)/bin' >> ~/.bash_profile
source ~/.bash_profile
```

## Create Systemd Service for tacchaind
```
sudo tee /etc/systemd/system/tacchaind.service > /dev/null << EOF
[Unit]
Description=tacchaind node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.tacchaind"
Environment="DAEMON_NAME=tacchaind"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$HOME/.tacchaind/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
```
```
sudo systemctl daemon-reload
sudo systemctl enable tacchaind.service
```

## Initialize Node and Set Port ( Chande node-name with yours )
```
tacchaind init node-name --chain-id tacchain_2391-1
```
```
echo "export TAC_PORT=60" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

##  Download Genesis and Set Seeds
```
curl -Ls https://raw.githubusercontent.com/TacBuild/tacchain/refs/heads/main/networks/tacchain_2391-1/genesis.json > $HOME/.tacchaind/config/genesis.json

SEEDS=""
PEERS="9c32b3b959a2427bd2aa064f8c9a8efebdad4c23@206.217.210.164:45130,04a2152eed9f73dc44779387a870ea6480c41fe7@206.217.210.164:45140,5aaaf8140262d7416ac53abe4e0bd13b0f582168@23.92.177.41:45110,ddb3e8b8f4d051e914686302dafc2a73adf9b0d2@23.92.177.41:45120"

sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.tacchaind/config/config.toml
```

## Set Pruning Options
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.tacchaind/config/app.toml
```

## Set Timeout Commit
```
sudo sed -i 's/timeout_commit = "5s"/timeout_commit = "2s"/' $HOME/.tacchaind/config/config.toml
```

## Update Port Configurations
```
sed -i.bak -e "s%:1317%:${TAC_PORT}423%g;
s%:8080%:${TAC_PORT}345%g;
s%:9090%:${TAC_PORT}210%g;
s%:9091%:${TAC_PORT}211%g;
s%:8545%:${TAC_PORT}123%g;
s%:8546%:${TAC_PORT}124%g;
s%:6065%:${TAC_PORT}001%g" $HOME/.tacchaind/config/app.toml

sed -i.bak -e "s%:26658%:${TAC_PORT}002%g;
s%:26657%:${TAC_PORT}001%g;
s%:6060%:${TAC_PORT}701%g;
s%:26656%:${TAC_PORT}000%g;
s%:26660%:${TAC_PORT}005%g" $HOME/.tacchaind/config/config.toml
```

## Download and restore snapshot
```
mkdir -p $HOME/.tacchaind
wget -O tacchain_1302405.tar.lz4 https://snapshots.polkachu.com/testnet-snapshots/tacchain/tacchain_1302405.tar.lz4 --inet4-only
lz4 -c -d tacchain_1302405.tar.lz4 | tar -x -C $HOME/.tacchaind
```

## Start the Node
```
sudo systemctl start tacchaind
journalctl -fu tacchaind -o cat
```

## Create a new wallet
```
tacchaind keys add <wallet-name>
```

##  Recover wallet from mnemonic
```
tacchaind keys add <wallet-name> --recover
```

## Go to Faucet
```
https://spb.faucet.tac.build/
```

## Create validator JSON file (customize amount, moniker, etc.)
```
echo "{\"pubkey\":{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"$(tacchaind tendermint show-validator | grep -Po '\"key\":\\s*\"\\K[^\"]*')\"},
    \"amount\": \"9000000000000000000utac\",
    \"moniker\": \"test\",
    \"identity\": \"\",
    \"website\": \"\",
    \"security\": \"\",
    \"details\": \"CR\",
    \"commission-rate\": \"0.1\",
    \"commission-max-rate\": \"0.2\",
    \"commission-max-change-rate\": \"0.01\",
    \"min-self-delegation\": \"1\"}" > validatortx.json
```

## Submit create-validator transaction (replace wallet, amount, and chain-id accordingly)
```
tacchaind tx staking create-validator validatortx.json \
    --from wallet \
    --chain-id tacchain_2391-1 \
    --node http://localhost:${TAC_PORT}001 \
    --gas auto --gas-adjustment 1.4 --fees 9503625000000000utac -y
```

## Edit Validator
```
tacchaind tx staking edit-validator \
  --new-moniker "MONIKER ADIN" \
  --identity "XXXXXXXXXXXXXXXXX" \
  --details "https://x.com/XXXXXXXXXXXXXX" \
  --website "https://github.com/XXXXXXXXXXXXXXXXX" \
  --chain-id tacchain_2391-1 \
  --from wallet \
  --node http://localhost:60001 \
  --gas auto --gas-adjustment 1.4 \
  --fees 9503625000000000utac -y
```

## Staking Delegate
```
tacchaind tx staking delegate tacvaloperXXXXXXX 9900000000000000000utac \
  --from wallet \
  --chain-id tacchain_2391-1 \
  --node http://localhost:59657 \
  --gas auto --gas-adjustment 1.4 \
  --fees 9503625000000000utac -y
```
