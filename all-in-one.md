#!/bin/bash

print_header() {
    # Alpha Storage Node header
    echo -e "\033[1;33m╔════════════════════════════════════════════════╗\033[0m"
    echo -e "\033[1;36m║          ★★ 👑 Alpha Storage Node 👑 ★★        ║\033[0m"
    echo -e "\033[1;33m╚════════════════════════════════════════════════╝\033[0m"
}

show_menu() {
    print_header
    echo "1. Install 0g-storage-node"
    echo "2. Update 0g-storage-node"
    echo "3. Turbo Mode(Reset Config.toml & Systemctl)"
    echo "4. Select RPC Endpoint"
    echo "5. Set Miner Key"
    echo "6. Snapshot Install"
    echo "7. Node Run & Show Logs"
    echo "8. Exit"
    echo "============================"
}

install_node() {
    print_header
    echo "Installing 0g-storage-node..."
    rm -r $HOME/0g-storage-node
    sudo apt-get update
    sudo apt-get install -y cargo git clang cmake build-essential openssl pkg-config libssl-dev
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
    source $HOME/.cargo/env
    git clone https://github.com/0glabs/0g-storage-node.git
    cd $HOME/0g-storage-node
    git fetch --all --tags
    git submodule update --init
    cargo build --release
    rm -rf $HOME/0g-storage-node/run/config.toml
    curl -o $HOME/0g-storage-node/run/config.toml https://raw.githubusercontent.com/zstake-xyz/test/refs/heads/main/0g_storage_turbo.toml
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
    sudo systemctl daemon-reload
    sudo systemctl enable zgs
    echo "Installation completed. You can start the service with 'sudo systemctl start zgs'."
}

update_node() {
    print_header
    echo "Updating 0g-storage-node..."
    sudo systemctl stop zgs
    cp $HOME/0g-storage-node/run/config.toml $HOME/0g-storage-node/run/config.toml.backup
    cd $HOME/0g-storage-node
    git stash
    git fetch --all --tags
    git checkout 7972bf8
    git submodule update --init
    cargo build --release
    cp $HOME/0g-storage-node/run/config.toml.backup $HOME/0g-storage-node/run/config.toml
    sudo systemctl daemon-reload
    sudo systemctl enable zgs
    sudo systemctl start zgs
    echo "Node update completed."
}

reset_config_systemctl() {
    print_header
    echo "Resetting Config.toml and Systemctl (Turbo Mode)..."
    rm -rf $HOME/0g-storage-node/run/config.toml
    curl -o $HOME/0g-storage-node/run/config.toml https://raw.githubusercontent.com/zstake-xyz/test/refs/heads/main/0g_storage_turbo.toml
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
    sudo systemctl daemon-reload
    sudo systemctl enable zgs
    echo "Config.toml and Systemctl have been reset to Turbo Mode. You can start the service with 'sudo systemctl start zgs'."
}

select_rpc() {
    print_header
    echo "Select an RPC Endpoint:"
    echo "1. https://0g-evmrpc.sr20de.xyz/"
    echo "2. https://0g-evm.maouam.nodelab.my.id/"
    echo "3. https://evmrpc-testnet.0g.ai"
    read -p "Enter your choice (1-3): " rpc_choice
    case $rpc_choice in
        1) rpc="https://0g-evmrpc.sr20de.xyz/" ;;
        2) rpc="https://0g-evm.maouam.nodelab.my.id/" ;;
        3) rpc="https://evmrpc-testnet.0g.ai" ;;
        *) echo "Invalid choice. Exiting."; return ;;
    esac
    sed -i "s|^blockchain_rpc_endpoint = .*|blockchain_rpc_endpoint = \"$rpc\"|g" ~/0g-storage-node/run/config.toml
    sudo systemctl stop zgs
    sudo systemctl daemon-reload
    sudo systemctl enable zgs
    echo "RPC Endpoint set to $rpc. You can start the service with 'sudo systemctl start zgs'."
}

set_miner_key() {
    print_header
    echo "Please enter your Miner Key:"
    read miner_key
    sed -i "s|^miner_key = .*|miner_key = \"$miner_key\"|g" ~/0g-storage-node/run/config.toml
    sudo systemctl daemon-reload
    sudo systemctl enable zgs
    sudo systemctl stop zgs
    sudo systemctl daemon-reload
    sudo systemctl enable zgs
    echo "Miner Key updated. You can start the service with 'sudo systemctl start zgs'."
}

snapshot_install() {
    print_header
    echo "===== Snapshot Install Menu ====="
    echo "1. Turbo Mode Snapshot Install"
    echo "2. Back to Main Menu"
    echo "============================"
    read -p "Select an option (1-2): " snap_choice
    case $snap_choice in
        1) 
            print_header
            echo "Installing Standard Mode Snapshot..."
            source <(curl -s https://raw.githubusercontent.com/zstake-xyz/test/refs/heads/main/0g_zgs_standard_snapshot.sh)
            echo "Turbo Mode Snapshot installation completed."
            ;;
        2) 
            echo "Returning to main menu..."
            return
            ;;
        *) 
            echo "Invalid option. Returning to main menu."
            return
            ;;
    esac
}

show_logs() {
    print_header
    echo "Displaying logs..."
    sudo systemctl daemon-reload && sudo systemctl enable zgs && sudo systemctl start zgs
    tail -f ~/0g-storage-node/run/log/zgs.log.$(TZ=UTC date +%Y-%m-%d)
}

while true; do
    show_menu
    read -p "Select an option (1-9): " choice
    case $choice in
        1) install_node ;;
        2) update_node ;;
        3) reset_config_systemctl ;;
        4) select_rpc ;;
        5) set_miner_key ;;
        6) snapshot_install ;;
        7) show_logs ;;
        8) echo "Exiting..."; exit 0 ;;
        *) echo "Invalid option. Please try again." ;;
    esac
    echo ""
done
