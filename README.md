# HowTo-Remote-configuration-debian-server-from-windows
This guide provides step-by-step instructions for configuring and managing a remote Debian server from a Windows client using Windows Subsystem for Linux (WSL). It covers SSH setup and package installation.

## Prerequisites
You need windows system with administrator rights.

- A running linux server with debian 12 and at least 4 gb ram

This example using ip-adress 12.34.56.78 to connect server and a root user on server.

### Install WSL Debian using PowerShell
To install Windows Subsystem for Linux (WSL) with Debian, open PowerShell as Administrator and run:

```powershell
wsl --install -d Debian
```

### Needed packages
First we need to ensure we are running latest WSL. Therefor we update and upgrade WSL.

```bash
sudo apt update
sudo apt upgrade
```

### Optionel: Setup envoriment variables in .bash_aliases (Debian WSL)
```bash
nano ~/.bash_aliases
```

```bash
# Adding globals to ~/.bash_aliases

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
# Add more aliases as needed
```

## SSH (secure connection with server)

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

## Add Packaged as needed

### MSSQL

#### Setting Up Repository
Below is a step-by-step explanation of how to set up the MSSQL Server repository on your Debian server. Each command is explained so you know exactly what is happening and why:

1. **Install required dependencies**
	These packages are needed to securely download and manage repositories and keys:
	```bash
	apt install gnupg2 apt-transport-https wget curl
	```

2. **Download and add the Microsoft GPG key**
	This command downloads the official Microsoft signing key and converts it to a format usable by Debian/Ubuntu:
	```bash
	curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | \
	gpg --dearmor | tee /usr/share/keyrings/microsoft.gpg > /dev/null 2>&1
	```

3. **Add the MSSQL Server repository**
	This command adds the Microsoft SQL Server repository for Ubuntu 22.04 to your Debian system. This is necessary because Microsoft does not provide a native Debian package:
	```bash
	echo "deb [arch=amd64,arm64,armhf signed-by=/usr/share/keyrings/microsoft.gpg] https://packages.microsoft.com/ubuntu/22.04/mssql-server-2022 jammy main" | \
	tee /etc/apt/sources.list.d/mssql-server-2022.list > /dev/null
	```

#### Installation and setup MSSQL

1. **Update package index**
	This command refreshes the list of available packages and their versions, ensuring you get the latest version when installing:
	```bash
	apt update
	```

2. **Install MSSQL Server**
	This command installs the Microsoft SQL Server package using the repository you added earlier:
	```bash
	apt-get install -y mssql-server
	```

3. **Run initial MSSQL configuration**
	This command starts the setup wizard for MSSQL Server, allowing you to choose edition, set the SA password, and configure other options:
	```bash
	/opt/mssql/bin/mssql-conf setup
	```

4. **Check MSSQL Server status**
	This command verifies that the MSSQL Server service is running correctly:
	```bash
	systemctl status mssql-server --no-pager
	```