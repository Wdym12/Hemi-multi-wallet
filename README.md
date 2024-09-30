**Hemi Multi Wallet**
1. **Use this command to create a dir heminetwork.sh**
	```bash
	nano heminetwork.sh

2. **After that paste below code inside**
	```bash
	 #!/bin/bash
	
	curl -s https://raw.githubusercontent.com/zunxbt/logo/main/logo.sh | bash
	
	sleep 3
	
	ARCH=$(uname -m)
	
	show() {
	    echo -e "\033[1;35m$1\033[0m"
	}
	
	if ! command -v jq &> /dev/null; then
	    show "jq not found, installing..."
	    sudo apt-get update
	    sudo apt-get install -y jq > /dev/null 2>&1
	    if [ $? -ne 0 ]; then
	        show "Failed to install jq. Please check your package manager."
	        exit 1
	    fi
	fi
	
	check_latest_version() {
	    for i in {1..3}; do
	        LATEST_VERSION=$(curl -s https://api.github.com/repos/hemilabs/heminetwork/releases/latest | jq -r '.tag_name')
	        if [ -n "$LATEST_VERSION" ]; then
	            show "Latest version available: $LATEST_VERSION"
	            return 0
	        fi
	        show "Attempt $i: Failed to fetch the latest version. Retrying..."
	        sleep 2
	    done
	
	    show "Failed to fetch the latest version after 3 attempts. Please check your internet connection or GitHub API limits."
	    exit 1
	}
	
	check_latest_version
	
	download_required=true
	
	if [ "$ARCH" == "x86_64" ]; then
	    if [ -d "heminetwork_${LATEST_VERSION}_linux_amd64" ]; then
	        show "Latest version for x86_64 is already downloaded. Skipping download."
	        cd "heminetwork_${LATEST_VERSION}_linux_amd64"  { show "Failed to change directory."; exit 1; }
	        download_required=false  # Set flag to false
	    fi
	elif [ "$ARCH" == "arm64" ]; then
	    if [ -d "heminetwork_${LATEST_VERSION}_linux_arm64" ]; then
	        show "Latest version for arm64 is already downloaded. Skipping download."
	        cd "heminetwork_${LATEST_VERSION}_linux_arm64"  { show "Failed to change directory."; exit 1; }
	        download_required=false  # Set flag to false
	    fi
	fi
	
	if [ "$download_required" = true ]; then
	    if [ "$ARCH" == "x86_64" ]; then
	        show "Downloading for x86_64 architecture..."
	        wget --quiet --show-progress "https://github.com/hemilabs/heminetwork/releases/download/$LATEST_VERSION/heminetwork_${LATEST_VERSION}_linux_amd64.tar.gz" -O "heminetwork_${LATEST_VERSION}_linux_amd64.tar.gz"
	        tar -xzf "heminetwork_${LATEST_VERSION}_linux_amd64.tar.gz" > /dev/null
	        cd "heminetwork_${LATEST_VERSION}_linux_amd64"  { show "Failed to change directory."; exit 1; }
	    elif [ "$ARCH" == "arm64" ]; then
	        show "Downloading for arm64 architecture..."
	        wget --quiet --show-progress "https://github.com/hemilabs/heminetwork/releases/download/$LATEST_VERSION/heminetwork_${LATEST_VERSION}_linux_arm64.tar.gz" -O "heminetwork_${LATEST_VERSION}_linux_arm64.tar.gz"
	        tar -xzf "heminetwork_${LATEST_VERSION}_linux_arm64.tar.gz" > /dev/null
	        cd "heminetwork_${LATEST_VERSION}_linux_arm64"  { show "Failed to change directory."; exit 1; }
	    else
	        show "Unsupported architecture: $ARCH"
	        exit 1
	    fi
	else
	    show "Skipping download as the latest version is already present."
	fi
	
	echo
	show "How many wallets do you want to create?"
	read -p "Enter the number of wallets: " wallet_count
	
	> wallet.txt
	
	for i in $(seq 1 $wallet_count); do
	    echo
	    show "Generating wallet $i..."
	    ./keygen -secp256k1 -json -net="testnet" > "wallet_$i.json"
	
	    if [ $? -ne 0 ]; then
	        show "Failed to generate wallet $i."
	        exit 1
	    fi
	
	    pubkey_hash=$(jq -r '.pubkey_hash' "wallet_$i.json")
	    priv_key=$(jq -r '.private_key' "wallet_$i.json")
	    ethereum_address=$(jq -r '.ethereum_address' "wallet_$i.json")
	
	    echo "Wallet $i - Ethereum Address: $ethereum_address" >> ~/PoP-Mining-Wallets.txt
	    echo "Wallet $i - BTC Address: $pubkey_hash" >> ~/PoP-Mining-Wallets.txt
	    echo "Wallet $i - Private Key: $priv_key" >> ~/PoP-Mining-Wallets.txt
	    echo "--------------------------------------" >> ~/PoP-Mining-Wallets.txt
	
	    show "Wallet $i details saved in wallet.txt"
	show "Join: https://discord.gg/hemixyz"
	    show "Request faucet from faucet channel to this address: $pubkey_hash"
	    echo
	    read -p "Have you requested faucet for wallet $i? (y/N): " faucet_requested
	    if [[ ! "$faucet_requested" =~ ^[Yy]$ ]]; then
	        show "Please request faucet before proceeding."
	        exit 1
	    fi
	done
	sleep 3
	echo
	read -p "Enter static fee (numerical only, recommended: 100-200): " static_fee
	echo
	
	for i in $(seq 1 $wallet_count); do
	    priv_key=$(jq -r '.private_key' "wallet_$i.json")
	
	    cat << EOF | sudo tee /etc/systemd/system/hemi_wallet_$i.service > /dev/null
	[Unit]
	Description=Hemi Network PoP Mining for Wallet $i
	After=network.target
	
	[Service]
	WorkingDirectory=$(pwd)
	ExecStart=$(pwd)/popmd
	Environment="POPM_BTC_PRIVKEY=$priv_key"
	Environment="POPM_STATIC_FEE=$static_fee"
	Environment="POPM_BFG_URL=wss://testnet.rpc.hemi.network/v1/ws/public"
	Restart=on-failure
	
	[Install]
	WantedBy=multi-user.target
	EOF
	
	    sudo systemctl daemon-reload
	    sudo systemctl enable hemi_wallet_$i.service
	    sudo systemctl start hemi_wallet_$i.service
	    show "PoP mining started for Wallet $i."
	done
	
	show "PoP mining is successfully started for all wallets."
**Now use Ctrl + X + Y and then press Enter to save this file**

3. **Use this command to run this script, make sure to send faucet in each wallet**
	```bash
	chmod +x heminetwork.sh && ./heminetwork.sh

4. **Check Logs**
	```bash
	journalctl -u hemi_wallet_<NUMBER>.service -n 50 -f

**Let’s say, u want to check out the logs of 1st wallet, so u need to use 1 in the place of <NUMBER>**

	journalctl -u hemi_wallet_1.service -n 50 -f

**If you want to check logs of 2nd wallet, in the similar way, u need to use 2 there**

	journalctl -u hemi_wallet_2.service -n 50 -f


