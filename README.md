# Orb-Agent-Installer
This installs all components needed and configurations.
Create file.
```
vi install_orb_agent.sh
```
Copy paste Script in file.
```
#!/usr/bin/env bash
set -euo pipefail

# ------------------------------
# Script: Install ORB Agent (Flexible Docker Install)
# ------------------------------

CONFIG_DIR="/opt/orb_agent"
CONFIG_FILE="$CONFIG_DIR/orb-agent-config.yaml"
ENV_FILE="$CONFIG_DIR/orb-agent-env.sh"

# Ensure /opt/orb_agent exists
sudo mkdir -p "$CONFIG_DIR"

# ------------------------------
# Detect & Install Docker if missing
# ------------------------------
echo "[*] Checking if Docker is installed..."
if ! command -v docker &> /dev/null; then
    echo "[!] Docker not found — installing now..."

    # Determine package manager
    if command -v apt-get &> /dev/null; then
        echo "[*] Detected Debian/Ubuntu system. Installing Docker..."
        sudo apt-get update
        sudo apt-get install -y ca-certificates curl gnupg lsb-release
        sudo mkdir -p /etc/apt/keyrings
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
        echo \
          "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
          https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
          sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
        sudo apt-get update
        sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

    elif command -v yum &> /dev/null || command -v dnf &> /dev/null; then
        PM=$(command -v dnf || echo yum)
        echo "[*] Detected RHEL/CentOS system. Installing Docker..."
        sudo $PM install -y yum-utils
        sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
        sudo $PM install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

    else
        echo "[ERROR] Unsupported Linux distribution — please install Docker manually."
        exit 1
    fi

    echo "[*] Enabling and starting Docker..."
    sudo systemctl enable docker
    sudo systemctl start docker
    if ! sudo systemctl is-active --quiet docker; then
        echo "[ERROR] Docker service failed to start!"
        exit 1
    fi

    echo "[+] Docker installation complete."
else
    echo "[+] Docker already installed."
    sudo systemctl start docker || true
fi

# ------------------------------
# Determine Docker network and default Diode target
# ------------------------------
echo "[*] NOTE: If the agent is on remote host use the HOST network"
echo "[*] Detecting available Docker networks..."
AVAILABLE_NETWORKS=($(docker network ls --format '{{.Name}}'))

if [ ${#AVAILABLE_NETWORKS[@]} -eq 0 ]; then
    echo "[!] No Docker networks found! Defaulting to 'host' network."
    DOCKER_NETWORK="host"
else
    echo "Available Docker networks:"
    i=1
    for net in "${AVAILABLE_NETWORKS[@]}"; do
        echo "  [$i] $net"
        ((i++))
    done

    # Try to auto-detect likely network
    DETECTED_NET=""
    for net in "${AVAILABLE_NETWORKS[@]}"; do
        if [[ "$net" == *"diode"* || "$net" == *"netbox"* ]]; then
            DETECTED_NET="$net"
            break
        fi
    done

    echo
    if [ -n "$DETECTED_NET" ]; then
        read -rp "Enter network number or name [default: $DETECTED_NET]: " NET_INPUT
        if [[ -z "$NET_INPUT" ]]; then
            DOCKER_NETWORK="$DETECTED_NET"
        elif [[ "$NET_INPUT" =~ ^[0-9]+$ ]] && (( NET_INPUT >= 1 && NET_INPUT <= ${#AVAILABLE_NETWORKS[@]} )); then
            DOCKER_NETWORK="${AVAILABLE_NETWORKS[$((NET_INPUT-1))]}"
        else
            DOCKER_NETWORK="$NET_INPUT"
        fi
    else
        read -rp "Enter network number or name [default: host]: " NET_INPUT
        if [[ -z "$NET_INPUT" ]]; then
            DOCKER_NETWORK="host"
        elif [[ "$NET_INPUT" =~ ^[0-9]+$ ]] && (( NET_INPUT >= 1 && NET_INPUT <= ${#AVAILABLE_NETWORKS[@]} )); then
            DOCKER_NETWORK="${AVAILABLE_NETWORKS[$((NET_INPUT-1))]}"
        else
            DOCKER_NETWORK="$NET_INPUT"
        fi
    fi
fi

echo "[*] Using Docker network: $DOCKER_NETWORK"
echo

# ------------------------------
# Prompt for Diode connection
# ------------------------------
echo
echo "    You MUST set the Diode Target and Token URL to point to the NetBox/Diode cluster. Ensure your diode target and token URL are correct"
echo
echo "    Example Orb-Agent Remote install:"
echo "      grpc://192.168.0.204:8080/diode"
echo "      http://192.168.0.204:8080/token"
echo
echo "    Replace 192.168.0.204 with the actual IP/FQDN where your NetBox/Diode cluster is running."
echo "    If Orb Agent is on the same network"
echo
echo "     Example Orb-Agent Local Install:"
echo "      grpc://diode-ingress-nginx-1:80/diode"
echo "      http://diode-ingress-nginx-1:80/token"

read -rp "Enter Diode Target URL ${DIODE_DEFAULT:+[$DIODE_DEFAULT]}: " INPUT
DIODE_TARGET="${INPUT:-$DIODE_DEFAULT}"

read -rp "Enter Diode Token URL ${DIODE_TOKEN_DEFAULT:+[$DIODE_TOKEN_DEFAULT]}: " INPUT
DIODE_TOKEN_URL="${INPUT:-$DIODE_TOKEN_DEFAULT}"

# Validation if remote mode
if [[ -z "$DIODE_TARGET" || -z "$DIODE_TOKEN_URL" ]]; then
    echo "[ERROR] Diode Target and Token URL must be provided for remote deployments."
    exit 1
fi

# ------------------------------
# ORB / Diode Credentials
# ------------------------------
read -rp "Enter ORB Agent Name (e.g., orb-agent): " ORB_AGENT_NAME
read -rp "Enter Client ID from NetBox: " CLIENT_ID
read -rp "Enter Client Secret from NetBox: " CLIENT_SECRET

# Save to environment file
cat <<EOF | sudo tee "$ENV_FILE" >/dev/null
export ORB_AGENT_NAME="$ORB_AGENT_NAME"
export CLIENT_ID="$CLIENT_ID"
export CLIENT_SECRET="$CLIENT_SECRET"
export DIODE_TARGET="$DIODE_TARGET"
export DIODE_TOKEN_URL="$DIODE_TOKEN_URL"
EOF


sudo chmod 600 "$ENV_FILE"

# ------------------------------
# Tags
# ------------------------------
read -rp "Enter tags for SNMP discovery (comma separated, e.g., snmp-discovery,my-tag): " SNMP_TAGS_INPUT
IFS=',' read -ra SNMP_TAGS_ARRAY <<< "$SNMP_TAGS_INPUT"
SNMP_TAGS_YAML="[\"${SNMP_TAGS_ARRAY[*]// /\",\"}\",\"$ORB_AGENT_NAME\"]"

read -rp "Enter tags for Network discovery (comma separated): " NET_TAGS_INPUT
IFS=',' read -ra NET_TAGS_ARRAY <<< "$NET_TAGS_INPUT"
NET_TAGS_YAML="[\"${NET_TAGS_ARRAY[*]// /\",\"}\",\"$ORB_AGENT_NAME\"]"

read -rp "Enter tags for Device discovery (comma separated): " DEV_TAGS_INPUT
IFS=',' read -ra DEV_TAGS_ARRAY <<< "$DEV_TAGS_INPUT"
DEV_TAGS_YAML="[\"${DEV_TAGS_ARRAY[*]// /\",\"}\",\"$ORB_AGENT_NAME\"]"

# ------------------------------
# Network Targets
# ------------------------------
read -rp "Enter Network Discovery Targets (comma separated, e.g., 192.168.0.1,10.0.0.0/24): " NETWORK_TARGETS_INPUT
IFS=',' read -ra NETWORK_TARGETS_ARRAY <<< "$NETWORK_TARGETS_INPUT"
NETWORK_TARGETS_YAML=""
for target in "${NETWORK_TARGETS_ARRAY[@]}"; do
    NETWORK_TARGETS_YAML+="            - $target"$'\n'
done

# ------------------------------
# Device Discovery
# ------------------------------
DEVICE_DISCOVERY_YAML=""
while true; do
    read -rp "Add a device for discovery? (y/n): " ADD_DEVICE
    [[ "$ADD_DEVICE" != "y" ]] && break

    read -rp "Device hostname/IP: " DEV_HOST
    read -rp "Device username: " DEV_USER
    read -rp "Device password: " DEV_PASS

    DEVICE_DISCOVERY_YAML+="          - hostname: $DEV_HOST
            driver: ios
            prompt: '[#$]'
            username: $DEV_USER
            password: \"$DEV_PASS\"
            read_timeout: 30
            fast_mode: true
            max_retries: 0
            session_log: /tmp/netmiko.log
            tags: $DEV_TAGS_YAML
"
done

# ------------------------------
# SNMP Discovery
# ------------------------------
SNMP_TARGETS_YAML=""
SNMP_AUTH_YAML=""
while true; do
    read -rp "Add SNMP target? (y/n): " ADD_SNMP
    [[ "$ADD_SNMP" != "y" ]] && break

    read -rp "SNMP Host/IP: " SNMP_HOST
    read -rp "SNMP Port (default 161): " SNMP_PORT
    SNMP_PORT=${SNMP_PORT:-161}
    read -rp "SNMP Protocol Version (default SNMPv2c): " SNMP_VERSION
    SNMP_VERSION=${SNMP_VERSION:-SNMPv2c}
    read -rp "SNMP Community (default public): " SNMP_COMMUNITY
    SNMP_COMMUNITY=${SNMP_COMMUNITY:-public}

    SNMP_TARGETS_YAML+="            - host: \"$SNMP_HOST\"
              port: $SNMP_PORT
          authentication:
             protocol_version: \"$SNMP_VERSION\"
             community: \"$SNMP_COMMUNITY\"
"$'\n'
done

# ------------------------------
# SNMP Discovery Defaults
# ------------------------------
read -rp "SNMP Site: " SNMP_SITE
read -rp "SNMP Role (eg, sysadmin): " SNMP_ROLE
read -rp "SNMP Location: " SNMP_LOCATION
read -rp "Device Manufacturer: " SNMP_MANUFACTURER
read -rp "Device Model (eg, .1.3.6.1.4.1.8072.3.2.10): " SNMP_MODEL

# ------------------------------
# Create ORB agent config
# ------------------------------
sudo tee "$CONFIG_FILE" >/dev/null <<EOF
orb:
  backends:
    common:
      diode:
        agent_name: "$ORB_AGENT_NAME"
        client_id: "$CLIENT_ID"
        client_secret: "$CLIENT_SECRET"
        scope: "diode:read diode:write"
        target: "$DIODE_TARGET"
        token_url: "$DIODE_TOKEN_URL"
        token_endpoint_auth_method: client_secret_post
        audience: "diode"

    network_discovery:
      host: 0.0.0.0
      port: 8073
      log_level: INFO
      log_format: TEXT

    device_discovery:
      host: 0.0.0.0
      port: 8857
      log_level: DEBUG
      log_format: TEXT

    snmp_discovery:
      host: 0.0.0.0
      port: 8140
      log_level: DEBUG
      log_format: TEXT

  policies:
    snmp_discovery:
      snmp_network_1:
        config:
          schedule: "* * * * *"
          defaults:
            site: "$SNMP_SITE"
            role: "$SNMP_ROLE"
            location: "$SNMP_LOCATION"
            device_type:
              manufacturer: "$SNMP_MANUFACTURER"
              model: "$SNMP_MODEL"
            tags: $SNMP_TAGS_YAML
        scope:
          targets:
$SNMP_TARGETS_YAML$SNMP_AUTH_YAML

    network_discovery:
      test_discovery:
        config:
          schedule: "* * * * *"
          timeout: 5
          defaults:
            description: IP discovered by network discovery
            tags: $NET_TAGS_YAML
        scope:
          targets:
$NETWORK_TARGETS_YAML
          fast_mode: true
          max_retries: 0

    device_discovery:
      test1_discovery:
        config:
          schedule: "* * * * *"
          timeout: 10
          defaults:
            description: IP discovered by device discovery
            tags: $DEV_TAGS_YAML
        scope:
$DEVICE_DISCOVERY_YAML
EOF

echo "ORB agent config created at $CONFIG_FILE"

# ------------------------------
# Docker install if missing
# ------------------------------
PM=""
if command -v apt-get &> /dev/null; then
    PM="apt"
elif command -v yum &> /dev/null || command -v dnf &> /dev/null; then
    PM=$(command -v dnf || echo yum)
else
    echo "Unsupported package manager."
    exit 1
fi

if ! command -v docker &> /dev/null; then
    echo "[*] Docker not found, installing..."
    if [[ $PM == "apt" ]]; then
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/docker-archive-keyring.gpg >/dev/null
        echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
        sudo apt-get update
        sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    else
        sudo $PM install -y yum-utils
        sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
        sudo $PM install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
    fi
else
    echo "[*] Docker already installed."
fi

# Ensure Docker is running
sudo systemctl enable docker
sudo systemctl start docker
if ! sudo systemctl is-active --quiet docker; then
    echo "Docker service failed to start. Exiting."
    exit 1
fi

# ------------------------------
# Pull ORB Agent image
# ------------------------------
echo "--- Pulling ORB agent Docker image ---"
if ! docker image inspect netboxlabs/orb-agent:latest >/dev/null 2>&1; then
    docker pull --disable-content-trust netboxlabs/orb-agent:latest
else
    echo "[*] ORB agent image already exists."
fi

# ------------------------------
# Run ORB Agent
# ------------------------------
echo "Starting ORB agent..."
sudo docker run -d \
  --name "$ORB_AGENT_NAME" \
  --restart unless-stopped \
  --network "$DOCKER_NETWORK" \
  -v "$CONFIG_FILE":/etc/orb-agent/config.yaml:ro \
  netboxlabs/orb-agent:latest \
  run --config /etc/orb-agent/config.yaml

echo "ORB agent $ORB_AGENT_NAME is running!"
```
