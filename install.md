# Omarchy Install

## Update

```bash
omarchy-update
```

## Packages

```bash
pacman -S --noconfirm \
    yq \
    jq \
    powershell \
    ghostty \
    aws-cli-v2 \
    file \
    visual-studio-code-bin \
    opencode
```

## Certificates

```bash
#!/bin/bash

# Configuration
URL="https://dl.dod.cyber.mil/wp-content/uploads/pki-pke/zip/unclass-certificates_pkcs7_DoD.zip"
ZIP_FILE="dod_certs.zip"
TEMP_DIR=$(mktemp -d)

# cleanup function: removes the temp directory and files on exit (success or error)
cleanup() {
    echo "--- Cleaning up temporary files ---"
    rm -rf "$TEMP_DIR"
}

# Register the cleanup function to trigger on exit, interrupt, or termination
trap cleanup EXIT

echo "--- Starting DoD Certificate Import ---"

# Move to temp directory to keep the system clean
cd "$TEMP_DIR" || { echo "Error: Could not enter temp directory"; exit 1; }

# 1. Download with error checking
echo "Downloading certificates..."
if ! curl -s -O -L "$URL"; then
    echo "Error: Failed to download certificates from $URL"
    exit 1
fi

# Rename the downloaded file to a known name for consistency
mv unclass-certificates_pkcs7_DoD.zip "$ZIP_FILE"

# 2. Unzip the PKCS7 file
echo "Extracting .p7b file..."
if ! unzip -j -q -o "$ZIP_FILE" "*DoD.pem.p7b"; then
    echo "Error: Failed to unzip PKCS7 file. It might not exist in the archive."
    exit 1
fi

# 3. Locate the extracted file and convert to PEM
P7B_FILE=$(find . -maxdepth 1 -name "*DoD.pem.p7b" | head -n 1)
if [[ -z "$P7B_FILE" ]]; then
    echo "Error: Could not find the extracted .p7b file."
    exit 1
fi

echo "Converting $P7B_FILE to PEM format..."
if ! openssl pkcs7 -print_certs -in "$P7B_FILE" -out DoD.pem; then
    echo "Error: OpenSSL conversion failed."
    exit 1
fi

# 4. Install the certs (requires sudo)
echo "Storing trust anchors (requires sudo)..."
if ! sudo trust anchor --store DoD.pem; then
    echo "Error: Failed to store trust anchors. Check sudo permissions."
    exit 1
fi

# 5. Verification
echo "--- Installation Complete. Verifying DoD Trust ---"
trust list | grep -i "DoD" | sort | uniq

echo "Done."
```

## Waybar

### Clock

```bash
sed 's#//.*##g' $HOME/.config/waybar/config.jsonc | jq '
  .["modules-center"] |= map(if . == "clock" then "custom/clock" else . end) |
  . + {
    "custom/clock": {
      "format": "{}",
      "exec": "date +\"%-l:%M %p %Z %A %B %-d\"",
      "interval": 30
    }
  }
' > $HOME/.config/waybar/config.jsonc.tmp && mv $HOME/.config/waybar/config.jsonc.tmp $HOME/.config/waybar/config.jsonc
```

### Height

```bash
sed 's#//.*##g' $HOME/.config/waybar/config.jsonc | jq '.height = 26' > $HOME/.config/waybar/config.jsonc.tmp && mv $HOME/.config/waybar/config.jsonc.tmp $HOME/.config/waybar/config.jsonc
```