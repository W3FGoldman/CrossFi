sudo apt update && sudo apt upgrade -y

sudo apt install curl git wget build-essential jq lz4 gcc unzip -y

# install go

cd $HOME

sudo rm -rf /usr/local/go

version="1.22.2"

wget "https://golang.org/dl/go$version.linux-amd64.tar.gz"

sudo tar -C /usr/local -xzf "go$version.linux-amd64.tar.gz"

sudo rm "go$version.linux-amd64.tar.gz"

echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile

source $HOME/.bash_profile

sudo mkdir -p $HOME/go/bin

# set var
echo "export WALLET="your wallet"" >> $HOME/.bash_profile

echo "export MONIKER="your moniker"" >> $HOME/.bash_profile

echo "export CROSSFI_CHAIN_ID="crossfi-evm-testnet-1"" >> $HOME/.bash_profile

source $HOME/.bash_profile

# download binary
cd $HOME

wget https://github.com/crossfichain/crossfi-node/releases/download/v0.3.0-prebuild3/crossfi-node_0.3.0-prebuild3_linux_amd64.tar.gz && tar -xvf crossfi-node_0.3.0-prebuild3_linux_amd64.tar.gz

sudo chmod +x $HOME/bin/crossfid

sudo mv $HOME/bin/crossfid $HOME/go/bin

sudo rm -rf crossfi-node_0.3.0-prebuild3_linux_amd64.tar.gz $HOME/bin

# set configuration 
crossfid config node tcp://localhost:26657

crossfid config chain-id crossfi-evm-testnet-1

crossfid config keyring-backend os

crossfid init "Your Moniker" --chain-id crossfi-evm-testnet-1

# download genesis and addrbook
curl -L https://tools.web3forces.com/genesis.json > $HOME/.mineplex-chain/config/genesis.json

curl -L https://tools.web3forces.com/addrbook.json > $HOME/.mineplex-chain/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "565fa7a934c083867e9c11fde1b841fa066479d4@35.240.115.172:26656,89752fa7945a06e972d7d860222a5eeaeab5c357@128.140.70.97:26656"|' $HOME/.mineplex-chain/config/config.toml

# set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "50"|' \
  $HOME/.mineplex-chain/config/app.toml

# set min.gas price 
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "10000000000000mpx"|g' $HOME/.mineplex-chain/config/app.toml

# enable prometheus
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.mineplex-chain/config/config.toml

# disable indexing
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.mineplex-chain/config/config.toml

# create service
sudo tee /etc/systemd/system/crossfid.service > /dev/null <<EOF
[Unit]
Description=Crossfi_TestNet
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.mineplex-chain
ExecStart=$(which crossfid) start --home $HOME/.mineplex-chain
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

# download snapshot
curl "https://tools.web3forces.com/snap_testnet-crossfi.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.mineplex-chain"

# start service
sudo systemctl daemon-reload

sudo systemctl enable crossfid

sudo systemctl restart crossfid && sudo journalctl -u crossfid -f
