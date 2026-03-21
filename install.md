# Omarchy Install

## Update

```bash
omarchy-update
```

## ls

```
echo "\nalias ls='LC_COLLATE=C /usr/bin/ls -la --color --group-directories-first'" >> $HOME/.local/share/omarchy/default/bash/aliases
```

## Packages

```bash
sudo pacman -S --noconfirm \
    yq \
    jq \
    powershell \
    aws-cli-v2 \
    file \
    visual-studio-code-bin \
    opencode
```

## PowerShell

```bash
yay -S powershell-bin
```

## SSH

```bash
sudo pacman -S --noconfirm openssh
sudo systemctl enable sshd
sudo systemctl start sshd
sudo ufw allow ssh
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

## Smartcard
```bash
sudo pacman -S --noconfirm \
    pcsclite \
    ccid \
    opensc
sudo systemctl enable pcscd
sudo systemctl start pcscd
modutil -dbdir sql:$HOME/.pki/nssdb/ -add "CAC Module" -libfile /usr/lib/opensc-pkcs11.so
```

## Ghostty

SUPER + ALT + SPACE -> Install -> Terminal -Ghostty

```bash
update_config() {
    local key="$1"
    local value="$2"
    local file="$3"

    if [[ ! -f "$file" ]]; then
        echo "Error: File $file not found."
        return 1
    fi

    # Check if key exists (at start of line)
    if grep -q "^$key[[:blank:]]*=" "$file"; then
        # Update existing key, preserving original spacing
        sed -i "s|^\($key\)\([[:blank:]]*=[[:blank:]]*\).*|\1\2$value|" "$file"
    else
        # Append new key-value pair to the end of the file
        echo "$key = $value" >> "$file"
    fi
}

mkdir -p $HOME/.config/ghostty/shaders
cp ./ghostty/shaders/cursor_blaze.glsl $HOME/.config/ghostty/shaders/

update_config "custom-shader" "$HOME/.config/ghostty/shaders/cursor_blaze.glsl" "$HOME/.config/ghostty/config"
update_config "font-size" "12" "$HOME/.config/ghostty/config"
update_config "font-family" "CascadiaCode Nerd Font" "$HOME/.config/ghostty/config"
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

## Oh My Posh

```bash
curl -s https://ohmyposh.dev/install.sh | bash -s
```

```bash
sed -i 's/^\s*eval "$(starship init bash)"/# eval "$(starship init bash)"/' $HOME/.local/share/omarchy/default/bash/init
```

```bash
cat <<'EOF' >> $HOME/.local/share/omarchy/default/bash/init
if command -v oh-my-posh &> /dev/null; then
  eval "$(oh-my-posh init bash --config $HOME/.posh_themes/thezim.omp.yaml)"
fi
EOF
```

```powershell
cp -r .posh_themes $HOME/
mkdir -p $HOME/.config/powershell
'oh-my-posh init pwsh --config $HOME/.posh_themes/thezim.omp.yaml | Invoke-Expression' >> $PROFILE
@'
# Native ls utility function
function Invoke-ListDirectory {
    param (
        [Parameter(Mandatory = $false, Position = 0, ValueFromRemainingArguments = $true)]$Remaining
    )
    $defaultargs = @("-la","--color","--group-directories-first")
    $env:LC_COLLATE = "C"
    ls $defaultargs @Remaining
    Remove-Item Env:\LC_COLLATE
}
Set-Alias -Name ls -Value Invoke-ListDirectory
'@ >> $PROFILE
```