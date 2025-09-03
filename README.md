# HowTo-Remote-configuration-debian-server-from-windows
This beginner-friendly guide shows how to manage a remote Debian server from a Windows PC using Windows Subsystem for Linux (WSL). You’ll set up SSH keys, connect securely, and install software like Microsoft SQL Server (MSSQL).

## Table of Contents
- [Prerequisites](#prerequisites)
	- [Install WSL (Debian) on Windows](#install-wsl-debian-on-windows)
	- [Update packages in WSL (Debian)](#update-packages-in-wsl-debian)
	- [Optional: Set up convenient aliases in ~/.bash_aliases (WSL)](#optional-set-up-convenient-aliases-in-bash_aliases-wsl)
- [SSH (secure connection with your server)](#ssh-secure-connection-with-your-server)
	- [Non-interactive SSH key generation](#non-interactive-ssh-key-generation)
	- [SSH passwordless connection](#ssh-passwordless-connection)
- [Create a non-privileged user (on the server)](#create-a-non-privileged-user-on-the-server)
- [Install software on the server](#install-software-on-the-server)
	- [MSSQL](#mssql)
		- [Setting Up Repository](#setting-up-repository)
		- [Installation and setup MSSQL (on the server)](#installation-and-setup-mssql-on-the-server)
		- [Allowing MSSQL Traffic via SSH Tunnel](#allowing-mssql-traffic-via-ssh-tunnel)
			- [1. Open the MSSQL port in the firewall (Debian server)](#1-open-the-mssql-port-in-the-firewall-debian-server)
			- [2. Set up SSH tunnel from Windows client](#2-set-up-ssh-tunnel-from-windows-client)
			- [3. Connect to MSSQL from Windows](#3-connect-to-mssql-from-windows)

## Prerequisites
- A Windows system with administrator rights.
- A Debian 12 server (remote) with at least 4 GB RAM.

Placeholders used below—replace with your details:
- 12.34.56.78 → your server IP or DNS name
- user → your Linux username on the server (avoid using root when possible)
- newusername → other user Linux username on the server

**Note:**
Get a powerfull server from contabo at low price.
(https://contabo.com/en/)

### Install WSL (Debian) on Windows
Open PowerShell as Administrator and install Debian in WSL.

Run in: Windows PowerShell (Admin)

```powershell
wsl --install -d Debian
```

### Update packages in WSL (Debian)
Make sure your Debian (inside WSL) is up to date.

Run in: WSL (Debian)

```bash
sudo apt update
sudo apt upgrade
```

### Optional: Set up convenient aliases in ~/.bash_aliases (WSL)
These aliases make SSH and key generation simpler.

Run in: WSL (Debian)
```bash
nano ~/.bash_aliases
```

```bash
# Adding convenience aliases to ~/.bash_aliases

# Set your email address for SSH key comment
SSHKEYGEN_EMAIL_ADDRESS="user@example.com"
# Set default SSH key file location
SSH_KEY_FILE="$HOME/.ssh/id_rsa"

# Alias for generating SSH keys
alias generate-ssh-key='ssh-keygen -t rsa -b 4096 -C "$SSHKEYGEN_EMAIL_ADDRESS" -f "$SSH_KEY_FILE" -N ""'

# Alias for server SSH connection
# Replace 'user' with your actual username on the server
# Replace '12.34.56.78' with your server's IP address
alias ssh-connect-myserver='ssh -i "$SSH_KEY_FILE" user@12.34.56.78'
alias ssh-connect-myserver-root='ssh -i "$SSH_KEY_FILE" root@12.34.56.78'
alias myserver-tunnel='ssh -L 1433:localhost:1433 user@12.34.56.78'
# Add more aliases as needed
```

After saving, reload your shell to activate the aliases (or open a new WSL terminal):

Run in: WSL (Debian)
```bash
source ~/.bashrc || source ~/.profile || true
```

## SSH (secure connection with your server)

### Non-interactive SSH key generation
Below is a step-by-step explanation of how to set up SSH key-based authentication for passwordless server access:

1. **Generate SSH key pair**
	This command creates a new SSH key pair (private and public key) for secure authentication:
	```bash
	generate-ssh-key
	```

### SSH passwordless connection
2. **Test SSH connection**
	This command checks if you can connect to your server as root using SSH:
	```bash
	ssh-connect-myserver-root
	```

3. **Copy public key to server**
	This command appends your public key to the server's list of authorized keys, enabling passwordless login:
	```bash
	cat ~/.ssh/id_rsa.pub | ssh-connect-myserver-root "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
	```

4. **Connect with SSH key**
	Now you can connect to the server without entering a password:
	```bash
	ssh-connect-myserver-root
	```

## Create a non-privileged user (on the server)
1. **Connect Server**
	Connect to the server (should no longer prompt for a password if keys are set up):
	```bash
	ssh-connect-myserver-root
	```

1. **Get sudo packages**
	`sudo` stands for "superuser do" and allows a regular (non-root) user to run commands with administrative (root) privileges. This is safer than logging in as root, because you only use elevated permissions when needed. For installation do:

	```bash
	'apt-get install -y sudo'
	```

After installing, you can add users to the `sudo` group so they can run admin commands safely.

1. **Create user**
	Add as many users as you need (replace newusername):
    ```bash
    adduser newusername    
	```

Run in: WSL (Debian) where newusername are login and had created a ssh key by `ssh-keygen` command  
2. **Each user needs to copy public key to server**
	```bash
	cat ~/.ssh/id_rsa.pub | ssh newusername@12.34.56.78 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
	```    

3. **Optional: Give a user sudo privileges**
	At least one user should have sudo privileges so we can close access for root through ssh.

	```bash
    usermod -aG sudo newusername
	```

## Configure firewall (nftables)
Debian uses nftables as its firewall system. Here’s how to reset the firewall, allow only SSH (port 22), and make the rules persistent:

### 1. Stop and flush any existing firewall rules
This ensures you start with a clean slate (run as root or with sudo):

```bash
sudo systemctl stop nftables || true
sudo nft flush ruleset
```

### 2. Allow only SSH (port 22)
Create a new table and chain, set default policy to drop, and allow only SSH:

```bash
sudo nft add table inet filter
sudo nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
sudo nft add rule inet filter input tcp dport 22 accept
```

### 3. Make nftables rules persistent
Save the current ruleset so it loads on boot:

```bash
sudo nft list ruleset | sudo tee /etc/nftables.conf > /dev/null
sudo systemctl enable nftables
sudo systemctl start nftables
```

Now, only port 22 (SSH) is accessible from outside. All other incoming connections are blocked by default.

## Install software on the server

### MSSQL

#### Setting Up Repository
Below is a step-by-step explanation of how to set up the MSSQL Server repository on your Debian server. Run these commands on the server (over SSH). Use sudo with a non-root user.

1. **Install required dependencies**
	These packages are needed to securely download and manage repositories and keys:
	```bash
	sudo apt install -y gnupg2 apt-transport-https wget curl
	```

2. **Download and add the Microsoft GPG key**
	This command downloads the official Microsoft signing key and converts it to a format usable by Debian/Ubuntu:
	```bash
	curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | \
	gpg --dearmor | sudo tee /usr/share/keyrings/microsoft.gpg > /dev/null 2>&1
	```

3. **Add the MSSQL Server repository**
	This command adds the Microsoft SQL Server repository for Ubuntu 22.04 to your Debian system. This is necessary because Microsoft does not provide a native Debian package:
	```bash
	echo "deb [arch=amd64,arm64,armhf signed-by=/usr/share/keyrings/microsoft.gpg] https://packages.microsoft.com/ubuntu/22.04/mssql-server-2022 jammy main" | \
	sudo tee /etc/apt/sources.list.d/mssql-server-2022.list > /dev/null
	```

#### Installation and setup MSSQL (on the server)

1. **Update package index**
	This command refreshes the list of available packages and their versions, ensuring you get the latest version when installing:
	```bash
	sudo apt update
	```

2. **Install MSSQL Server**
	This command installs the Microsoft SQL Server package using the repository you added earlier:
	```bash
	sudo apt-get install -y mssql-server
	```

3. **Run initial MSSQL configuration**
	This command starts the setup wizard for MSSQL Server, allowing you to choose edition, set the SA password, and configure other options:
	```bash
	sudo /opt/mssql/bin/mssql-conf setup
	```

4. **Check MSSQL Server status**
	This command verifies that the MSSQL Server service is running correctly:
	```bash
	sudo systemctl status mssql-server --no-pager
	```

#### Allowing MSSQL Traffic via SSH Tunnel

To securely access your MSSQL server from a Windows client, you can use SSH tunneling. This allows you to forward a local port on your Windows machine to the remote MSSQL port (default 1433) on your Debian server, without exposing the database port directly to the internet.

##### 1. Open the MSSQL port in the firewall (Debian server)
If you want to allow only local connections (recommended for SSH tunneling), allow traffic on port 1433 for localhost only:

```bash
sudo nft add rule inet filter input ip saddr 127.0.0.1 tcp dport 1433 accept
sudo nft list ruleset | sudo tee /etc/nftables.conf > /dev/null
```

##### 2. Set up SSH tunnel from Windows client
On your Windows machine, use PowerShell or Command Prompt to create an SSH tunnel. This forwards your local port 1433 to the remote server's port 1433:

```powershell
ssh -L 1433:localhost:1433 user@your-server-ip
```

Replace `user` with your server username and `your-server-ip` with the server's IP address. Leave this terminal open while you need the tunnel.

Alternative (from WSL): if you added the alias earlier, you can run:

Run in: WSL (Debian)
```bash
myserver-tunnel
```

##### 3. Connect to MSSQL from Windows
In your SQL client (e.g., SSMS, Azure Data Studio), connect to `localhost,1433` as the server name. The connection will be securely tunneled to your Debian server.

**Note:**
- Make sure the SSH port (default 22) is open on your server.
- The MSSQL service must be running and listening on port 1433 (default).