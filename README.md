# Clone project repository
cd && rm -rf seda-chain
git clone https://github.com/sedaprotocol/seda-chain
cd seda-chain
git checkout v0.1.1

# Build binary
make install

# Set node CLI configuration
sedad config set client chain-id seda-1
sedad config set client keyring-backend file
sedad config set client node tcp://localhost:25857

# Initialize the node
sedad init "Your Node Name" --chain-id seda-1

# Download genesis and addrbook files
curl -L https://snapshots.nodejumper.io/seda/genesis.json > $HOME/.sedad/config/genesis.json
curl -L https://snapshots.nodejumper.io/seda/addrbook.json > $HOME/.sedad/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "31f54fbcf445a9d9286426be59a17a811dd63f84@18.133.231.208:26656,ade4d8bc8cbe014af6ebdf3cb7b1e9ad36f412c0@seeds.polkachu.com:25856,cec848e7d4c5a7ae305b27cda133d213435c110f@seed-seda.ibs.team:16679,400f3d9e30b69e78a7fb891f60d76fa3c73f0ecc@seda.rpc.kjnodes.com:17359,20e1000e88125698264454a884812746c2eb4807@seeds.lavenderfive.com:25856,cec848e7d4c5a7ae305b27cda133d213435c110f@seed-seda.ibs.team:16679,ebc272824924ea1a27ea3183dd0b9ba713494f83@seda-mainnet-seed.autostake.com:26866,c28827cb96c14c905b127b92065a3fb4cd77d7f6@seeds.whispernode.com:25856,b85358e035343a3b15e77e1102857dcdaf70053b@seeds.bluestake.net:24656,31f54fbcf445a9d9286426be59a17a811dd63f84@18.133.231.208:26656"|' $HOME/.sedad/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "10000000000aseda"|' $HOME/.sedad/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.sedad/config/app.toml

# Change ports
sed -i -e "s%:1317%:25817%; s%:8080%:25880%; s%:9090%:25890%; s%:9091%:25891%; s%:8545%:25845%; s%:8546%:25846%; s%:6065%:25865%" $HOME/.sedad/config/app.toml
sed -i -e "s%:26658%:25858%; s%:26657%:25857%; s%:6060%:25860%; s%:26656%:25856%; s%:26660%:25861%" $HOME/.sedad/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots.nodejumper.io/seda/seda_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.sedad"

# Create a service
sudo tee /etc/systemd/system/sedad.service > /dev/null << EOF
[Unit]
Description=SEDA node service
After=network-online.target
[Service]
User=$USER
ExecStart=$(which sedad) start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable sedad.service

# Start the service and check the logs
sudo systemctl start sedad.service
sudo journalctl -u sedad.service -f --no-hostname -o cat
