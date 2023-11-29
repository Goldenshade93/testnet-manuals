### Kyve install

```bash
cd $HOME
sudo apt update
sudo apt install make clang pkg-config libssl-dev build-essential git jq ncdu bsdmainutils htop -y < "/dev/null"


cd $HOME
wget -O go1.18.1.linux-amd64.tar.gz https://golang.org/dl/go1.18.1.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.18.1.linux-amd64.tar.gz && rm go1.18.1.linux-amd64.tar.gz
echo 'export GOROOT=/usr/local/go' >> $HOME/.bash_profile
echo 'export GOPATH=$HOME/go' >> $HOME/.bash_profile
echo 'export GO111MODULE=on' >> $HOME/.bash_profile
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile && . $HOME/.bash_profile
go version


git clone https://github.com/KYVENetwork/chain.git
cd $HOME/chain
git checkout v0.4.0
curl https://get.ignite.com/cli! | bash
ignite chain build --release --release.prefix kyve
tar -xvzf $HOME/chain/release/kyve_linux_amd64.tar.gz
sudo mv $HOME/chain/chaind /usr/local/bin/kyved
cd $HOME


kyved init $NODENAME --chain-id korellia
wget https://github.com/KYVENetwork/chain/releases/download/v0.0.1/genesis.json -O ~/.kyve/config/genesis.json


pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="10"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.kyve/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.kyve/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.kyve/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.kyve/config/app.toml
sed -i.bak -e "s/indexer *=.*/indexer = \"null\"/g" $HOME/.kyve/config/config.toml


sed -i.bak -e "s/minimum-gas-prices  *=.*/minimum-gas-prices  = \"0.0tkyve\"/g" $HOME/.kyve/config/app.toml
peers="a24fc4dfdf780931a9d3c1f5082431fea7a33dca@65.108.225.158:10656"
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.kyve/config/config.toml


echo "[Unit]
Description=kyve Node
After=network.target

[Service]
User=$USER
Type=simple
ExecStart=$(which kyved) start
Restart=on-failure
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target" > $HOME/kyved.service
sudo mv $HOME/kyved.service /etc/systemd/system
sudo tee <<EOF >/dev/null /etc/systemd/journald.conf
Storage=persistent
EOF
sudo systemctl restart systemd-journald
sudo systemctl daemon-reload
sudo systemctl enable kyved


sudo systemctl stop kyved
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1false|" $HOME/.kyve/config/config.toml
kyved tendermint unsafe-reset-all --home $HOME/.kyve/
SNAP_RPC="https://kyve-testnet-rpc.polkachu.com:443"
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); BLOCK_HEIGHT=$((LATEST_HEIGHT - 2000)); TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ; \
s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" $HOME/.kyve/config/config.toml

sudo systemctl restart kyved
sudo journalctl -u kyved -f -o cat
```
