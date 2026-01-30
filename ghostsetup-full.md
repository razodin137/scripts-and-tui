#!/bin/bash

# Prompt for username and directory name
read -p "Enter the username: " USERNAME
read -p "Enter the directory name for Ghost: " SITENAME

# Check if the user exists, create if not
if id "$USERNAME" &>/dev/null; then
    echo "User $USERNAME exists."
else
    echo "User $USERNAME does not exist. Creating user $USERNAME."
    sudo adduser $USERNAME
    sudo usermod -aG sudo $USERNAME
    echo "User $USERNAME created and added to sudo group."
fi

# Update packages
echo "Updating packages..."
sudo apt-get update -y
sudo apt-get upgrade -y

# Install required dependencies
echo "Installing required dependencies..."
sudo apt-get install -y curl ca-certificates gnupg lsb-release

# Install Node.js (v18 or higher) and npm if not installed
echo "Installing Node.js and npm..."
if ! command -v node &>/dev/null; then
    curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
    sudo apt-get install -y nodejs
else
    echo "Node.js is already installed. Checking version..."
    NODE_VERSION=$(node -v)
    if [[ "$NODE_VERSION" < "v18" ]]; then
        echo "Your Node.js version is outdated. Updating to v18..."
        sudo apt-get install -y nodejs
    else
        echo "Node.js is up to date."
    fi
fi

# Install npm if it's missing
if ! command -v npm &>/dev/null; then
    echo "npm is not installed. Installing npm..."
    sudo apt-get install -y npm
else
    echo "npm is already installed."
fi

# Install Ghost-CLI
echo "Installing Ghost-CLI..."
sudo npm install ghost-cli@latest -g

# Create the directory for Ghost if it doesn't exist
echo "Creating directory for Ghost site..."
sudo mkdir -p /var/www/$SITENAME

# Set directory owner and permissions
echo "Setting up permissions for /var/www/$SITENAME..."
sudo chown $USERNAME:$USERNAME /var/www/$SITENAME
sudo chmod 775 /var/www/$SITENAME

# Navigate into the directory
cd /var/www/$SITENAME

# Run Ghost install
echo "Installing Ghost..."
ghost install local

# After installation, check for SSL setup (optional)
read -p "Would you like to set up SSL with Let's Encrypt? (y/n): " SSL_SETUP
if [[ "$SSL_SETUP" == "y" ]]; then
    ghost setup ssl
fi

echo "Ghost installation and setup complete."

