# Babylon-Service

### Babylon node Installation Instructions.

[Official documentation](https://docs.babylonchain.io/docs/introduction/overview)

System requirements:</br>
CPU: 4 Core</br>
RAM: 8 Gb</br>
SSD: 160 Gb</br>
OS: Ubuntu 20.04 or 22.04</br>

### Network configurations: </br>
Outbound - allow all traffic </br>
Inbound - open the following ports :</br>
1317 - REST </br>
26657 - TENDERMINT_RP </br>
26656 - Cosmos </br>
Running as a Provider? Add these specific ports: </br>
22221 - provider port </br>
22231 - provider port </br>
22241 - provider port </br>

### Installing the Babylon Node

1. Preparing the server/Required packages installation</br>
```
sudo apt update
sudo apt upgrade -y
sudo apt-get install libclang-dev
sudo apt install -y unzip logrotate git jq sed wget curl coreutils systemd
```
### Go installation.
```
cd $HOME
curl https://dl.google.com/go/go1.20.1.linux-amd64.tar.gz | sudo tar -C/usr/local -zxvf -
cat <<'EOF' >>$HOME/.profile
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
source $HOME/.profile
go version
```

### Download and build binaries
```
MONIKER="YOUR_MONIKER_GOES_HERE"
cd $HOME
rm -rf babylon
git clone https://github.com/babylonchain/babylon.git
cd babylon
git checkout v0.7.2
```

### Build binaries
```
make build
```
### Prepare binaries for Cosmovisor
```
mkdir -p $HOME/.babylond/cosmovisor/genesis/bin
mv build/babylond $HOME/.babylond/cosmovisor/genesis/bin/
rm -rf build
```

### Create application symlinks
```
sudo ln -s $HOME/.babylond/cosmovisor/genesis $HOME/.babylond/cosmovisor/current -f
sudo ln -s $HOME/.babylond/cosmovisor/current/bin/babylond /usr/local/bin/babylond -f
```

### Download and install Cosmovisor
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0
```

### Create service
```
sudo tee /etc/systemd/system/babylon.service > /dev/null << EOF
[Unit]
Description=babylon node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.babylond"
Environment="DAEMON_NAME=babylond"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.babylond/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
```

### Initialize the node
```
sudo systemctl daemon-reload
sudo systemctl enable babylon.service
```
### Set node configuration
```
babylond config chain-id bbn-test-2
babylond config keyring-backend test
babylond config node tcp://localhost:16457
```

### Initialize the node
```
babylond init $MONIKER --chain-id bbn-test-2
```

### Download genesis and addrbook
```
curl -Ls https://snapshots.kjnodes.com/babylon-testnet/genesis.json > $HOME/.babylond/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/babylon-testnet/addrbook.json > $HOME/.babylond/config/addrbook.json
```

### Add seeds
```
sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@babylon-testnet.rpc.kjnodes.com:16459\"|" $HOME/.babylond/config/config.toml
```

### Set minimum gas price
```
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.00001ubbn\"|" $HOME/.babylond/config/app.toml
```

### Set pruning
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.babylond/config/app.toml
```

### Set custom ports
```
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:16458\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:16457\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:16460\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:16456\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":16466\"%" $HOME/.babylond/config/config.toml
sed -i -e "s%^address = \"tcp://localhost:1317\"%address = \"tcp://0.0.0.0:16417\"%; s%^address = \":8080\"%address = \":16480\"%; s%^address = \"localhost:9090\"%address = \"0.0.0.0:16490\"%; s%^address = \"localhost:9091\"%address = \"0.0.0.0:16491\"%; s%:8545%:16445%; s%:8546%:16446%; s%:6065%:16465%" $HOME/.babylond/config/app.toml
```

### Download latest chain snapshot
```
curl -L https://snapshots.kjnodes.com/babylon-testnet/snapshot_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.babylond
[[ -f $HOME/.babylond/data/upgrade-info.json ]] && cp $HOME/.babylond/data/upgrade-info.json $HOME/.babylond/cosmovisor/genesis/upgrade-info.json
```

### Start service and check the logs
```
sudo systemctl start babylon.service && sudo journalctl -u babylon.service -f --no-hostname -o cat
```

### Becoming a Validator

# Create wallet key new
```
babylond keys add wallet
```

(OPTIONAL) RECOVER EXISTING KEY
```
babylond keys add wallet --recover
```

### We receive tokens from the tap in the [discord](https://discord.gg/babylonglobal)
```
In the #get-a-role branch, you get all possible roles, then go to the #faucet branch and write!faucet your babylon address(bbn....)
```

### Create a BLS key
Validators are expected to submit a BLS signature at the end of each epoch. To do that, a validator needs to have a BLS key pair to sign information with.

Using the address that you created on the previous step.
```
babylond create-bls-key $(babylond keys show wallet -a)
```
This command will create a BLS key and add it to the $HOME/.babylond/config/priv_validator_key.json. This is the same file that stores the private key that the validator uses to sign blocks. Please ensure that this file is secured properly.

After creating a BLS key, you need to restart your node to load this key into memory.
```
sudo systemctl restart babylon.service
```

### Modify the Configuration

Furthermore, you need to specify the name of the key that the validator will be using to submit BLS signature transactions under the $HOME/.babylond/config/app.toml file. Edit this file and set the key name to the one that holds funds on your keyring:
```
sed -i -e "s|^key-name *=.*|key-name = \"wallet\"|" $HOME/.babylond/config/app.toml
```
Finally, it is strongly recommended to modify the timeout_commit value under $HOME/.babylond/config/config.toml. This value specifies how long a validator will wait before commiting a block before starting on a new height. Given that Babylon aims to have a 10 second time between blocks, set this value to:
```
sed -i -e "s|^timeout_commit *=.*|timeout_commit = \"10s\"|" $HOME/.babylond/config/config.toml
```

### Create the Validator

Before creating a validator, enter the command and check that you have false. This means that the Node has synchronized and you can create a validator:
```
babylond status 2>&1 | jq .SyncInfo
```
Please enter your details below in quotation marks where required

```
babylond tx checkpointing create-validator \
--amount 1000000ubbn \
--pubkey $(babylond tendermint show-validator) \
--moniker "YOUR_MONIKER_NAME" \
--identity "YOUR_KEYBASE_ID" \
--details "YOUR_DETAILS" \
--website "YOUR_WEBSITE_URL" \
--chain-id bbn-test-2 \
--commission-rate 0.05 \
--commission-max-rate 0.20 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--from wallet \
--gas-adjustment 1.4 \
--gas auto \
--gas-prices 0.00001ubbn \
-y
```

### Update
```
There have been no updates at the moment, as soon as they come out, we will immediately add them to this section.

Current network:bbn-test-2
Current version:v0.7.2
```

### Useful commands

Check balance
```
babylond q bank balances $(babylond keys show wallet -a)
```

CHECK SERVICE LOGS
```
sudo journalctl -u babylon.service -f --no-hostname -o cat
```

RESTART SERVICE
```
sudo systemctl restart babylon.service
```

GET VALIDATOR INFO
```
babylond status 2>&1 | jq .ValidatorInfo
```

DELEGATE TOKENS TO YOURSELF
```
babylond tx epoching delegate $(babylond keys show wallet --bech val -a) 1000000ubbn --from wallet --chain-id bbn-test-2 --gas-adjustment 1.4 --gas auto --fees 10ubbn -y
```

REMOVE NODE
Please, before proceeding with the next step! All chain data will be lost! Make sure you have backed up your priv_validator_key.json!
```
cd $HOME
sudo systemctl stop babylon.service
sudo systemctl disable babylon.service
sudo rm /etc/systemd/system/babylon.service
sudo systemctl daemon-reload
rm -f $(which babylond)
rm -rf $HOME/.babylond
rm -rf $HOME/babylon
```

