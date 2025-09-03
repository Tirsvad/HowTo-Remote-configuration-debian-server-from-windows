# HowTo-Remote-configuration-debian-server-from-windows
Remote configuration of a debian server from windows client using WSL

## Prerequisites
You need windows system with administrator rights.

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

## Setting up global variables in .bashrc (Debian WSL)
To make SSH key generation and SSH connections non-interactive, you can set global variables in your `.bash_aliases` file:


### Non-interactive SSH key generation
```bash
generate-ssh-key
```

### SSH passwordless connection
First check if we have connection
```bash
ssh-connect-myserver-root
```

```bash
cat ~/.ssh/id_rsa.pub | ssh-connect-myserver-root "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Now you can connect server with key instead of password

```bash
ssh-connect-myserver-root
```
