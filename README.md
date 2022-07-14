# NEAR validator setup

## Ubuntu server setup

### 1. Wifi

1. Wifi packages don't come pre-installed. Load them in a USB from another PC

  https://yping88.medium.com/how-to-enable-wi-fi-on-ubuntu-server-20-04-without-a-wired-ethernet-connection-42e0b71ca198

I required two packages in addition to those mentioned in guide.

2. Netplan: A yaml based tool to setup network

  - Create config
    ```
    sudo nano /etc/netplan/01-config.yaml
    ```

    ```yaml
    network:
    wifis:
      wlx503eaa5848a0: # logical name
        dhcp4: true
        optional: true
        access-points:
          "wifi network name":
            password: "**********"
    version: 2
    renderer: networkd
    ```

    - For android tethering- https://askubuntu.com/questions/1336651/how-to-connect-ubuntu-server-20-04-to-android-tethering-usb-using-netplan

  - Run commands

  ```sh
  sudo netplan --debug generate
  sudo netplan --debug apply
  ```

  - Debugging

  ```sh
  # View devices
  lshw -C network

  # Device statuses
  networkctl

  # View devices on local network
  nmap -sP 192.168.29.0/22
  ```

### 2. SSH

- On server

  ```sh
  sudo apt install openssh-server

  # check if up, otherwise run 'sudo systemctl enable --now ssh'
  service ssh status

  # Find local IP address of server under inet section
  ip a
  ```

- On client (must be on same network)

  ```sh
  ssh validatoor@192.168.29.24 # new dongle, USB 2.0 port
  ssh validatoor@192.168.29.23 # new dongle, USB 3.0 port
  ssh validatoor@192.168.29.122 # old dongle

  # to exit
  logout
  ```

### 3. tmux

- Lets you run multiple terminal windows, that don't close on loss of SSH connection.
- tmux can run multiple sessions. A session has one or more windows. The session name is shown in the bottom as `[0]`. Window names are displaye as `0:bash, 1:bash` and so on. A window has one or more panes.


```sh
# Create new session
tmux new

# To run commands, press Ctrl + b + prefix, ie C-b prefix

# Help
C-b ?

# View list of sessions, and their windows
tmux ls

# Detach session to make it run in the background
C-b d

# Attach session 0
tmux attach -t0

# new window
C-b c

# Select window
C-b 0
C-b 1

# Create new pane horizontally
C-b %

# Create new pane vertically
C-b \"

# Close current pane
C-b x

# Close current window
C-b &

# Kill session
C-b :kill-session

# Stop tmux
C-b :kill-server
```

## Validator setup

```sh
# clone latest version of Nearcore
git clone https://github.com/near/nearcore
cd nearcore
git fetch origin --tags
git checkout tags/1.28.0-rc.1

# Compile binary- it is generated in target/release/neard
make neard

# Init working directory- generates config.json and node_key.json; downloads genesis.json
./target/release/neard --home ~/.near init --chain-id <network> --download-genesis

# Folder structure in ~/.near
# - config.json: consensus details
# - genesis.json: genesis file
# - node_key.json: Contains the node's public-private keypair and account_id.
# - data: for blockchain state

# Replace config.json
rm ~/.near/config.json
wget -O ~/.near/config.json https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/<network>/config.json

# Install AWS CLI

# Clone backup
aws s3 --no-sign-request cp s3://near-protocol-public/backups/<testnet|mainnet>/rpc/latest .
LATEST=$(cat latest)
aws s3 --no-sign-request cp --no-sign-request --recursive s3://near-protocol-public/backups/<testnet|mainnet>/rpc/$LATEST ~/.near/data
```

## Common issues

1. The SSD has 500GB storage but the `/` partition was initialized with 100GB. Extend size to occupy entire 100% storage.

```sh
lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

https://askubuntu.com/questions/1269493/ubuntu-server-20-04-1-lts-not-all-disk-space-was-allocated-during-installation

2. Resource monitoring

```sh
# Hard disk usage
df -h

# Processor
top -bn1 | grep "Cpu(s)" | \
           sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | \
           awk '{print 100 - $1"%"}'

# RAM
free
```