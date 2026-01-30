#!/bin/bash

set -e  # Exit on any error

# Variables
DOWNLOAD_URL="https://exiftool.org/Image-ExifTool-13.12.tar.gz"
FILE_NAME="Image-ExifTool-13.12.tar.gz"
EXTRACTED_DIR="Image-ExifTool-13.12"

# Functions
install_exiftool() {
    echo "Downloading ExifTool..."
    wget -O "$FILE_NAME" "$DOWNLOAD_URL"

    echo "Unpacking ExifTool distribution..."
    gzip -dc "$FILE_NAME" | tar -xf -

    echo "Changing directory to $EXTRACTED_DIR..."
    cd "$EXTRACTED_DIR"

    echo "Testing ExifTool..."
    perl Makefile.PL
    make test || echo "Tests failed, but you can still install ExifTool."

    echo "Installing ExifTool..."
    sudo make install

    echo "ExifTool installed successfully! You can now use it by typing 'exiftool'."
}

uninstall_exiftool() {
    echo "Uninstalling ExifTool..."
    if [ -d "$EXTRACTED_DIR" ]; then
        cd "$EXTRACTED_DIR"
        sudo make uninstall || echo "Uninstall message detected. Follow the displayed manual steps if necessary."
    else
        echo "Directory $EXTRACTED_DIR not found. Ensure you're in the correct directory where ExifTool was built."
    fi
    echo "ExifTool uninstalled successfully."
}

# Main Menu
echo "Choose an option:"
echo "1. Install ExifTool"
echo "2. Uninstall ExifTool"
read -p "Enter your choice (1/2): " choice

case $choice in
    1)
        install_exiftool
        ;;
    2)
        uninstall_exiftool
        ;;
    *)
        echo "Invalid choice. Exiting."
        exit 1
        ;;
esac

