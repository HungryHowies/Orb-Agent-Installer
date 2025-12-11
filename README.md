# Orb-Agent-Installer
This installs all components needed and configurations.
NOTE: This script works off of the Devcie Role and Orb Agent Credentials.
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
# Produces /opt/orb_agent/orb-agent-config.yaml and runs the container
# SNMP targets are listed under 'targets' and a single shared
# 'authentication' block is written after the targets (shared auth).
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
echo "[*] NOTE: If the agent is on Localhost use the same network Diode cluster is on"
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
# Prompt for Diode connection & credentials
# ------------------------------
echo
echo "You MUST set the Diode Target and Token URL to point to the NetBox/Diode cluster."
echo "Example Orb-Agent Remote install:"
echo "  grpc://192.168.0.204:8080/diode"
echo "  http://192.168.0.204:8080/token"
echo "Example Orb-Agent Local Install:"
echo "  grpc://diode-ingress-nginx-1:80/diode"
echo "  http://diode-ingress-nginx-1:80/token"
echo

read -rp "Enter Diode Target URL: " DIODE_TARGET
read -rp "Enter Diode Token URL: " DIODE_TOKEN_URL
if [[ -z "$DIODE_TARGET" || -z "$DIODE_TOKEN_URL" ]]; then
    echo "[ERROR] Diode Target and Token URL must be provided."
    exit 1
fi

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
read -rp "Enter tags for SNMP discovery (comma separated, e.g., snmp,orb): " SNMP_TAGS_INPUT
IFS=',' read -ra SNMP_TAGS_ARRAY <<< "${SNMP_TAGS_INPUT:-snmp}"
# produce YAML list: ["tag1","tag2","orb-agent"]
SNMP_TAGS_YAML="$(printf '\"%s\",' "${SNMP_TAGS_ARRAY[@]}" | sed 's/,$//')"
SNMP_TAGS_YAML="[$SNMP_TAGS_YAML,\"$ORB_AGENT_NAME\"]"

read -rp "Enter tags for Network discovery (comma separated, or leave blank): " NET_TAGS_INPUT
IFS=',' read -ra NET_TAGS_ARRAY <<< "${NET_TAGS_INPUT:-network}"
NET_TAGS_YAML="$(printf '\"%s\",' "${NET_TAGS_ARRAY[@]}" | sed 's/,$//')"
NET_TAGS_YAML="[$NET_TAGS_YAML,\"$ORB_AGENT_NAME\"]"

read -rp "Enter tags for Device discovery (comma separated, or leave blank): " DEV_TAGS_INPUT
IFS=',' read -ra DEV_TAGS_ARRAY <<< "${DEV_TAGS_INPUT:-device}"
DEV_TAGS_YAML="$(printf '\"%s\",' "${DEV_TAGS_ARRAY[@]}" | sed 's/,$//')"
DEV_TAGS_YAML="[$DEV_TAGS_YAML,\"$ORB_AGENT_NAME\"]"

# ------------------------------
# SNMP Discovery targets (collect only host + port entries)
# We'll write a single shared 'authentication' block after targets
# ------------------------------
SNMP_TARGETS_YAML=""
echo
echo "Now add SNMP targets. Repeat until done."
while true; do
    read -rp "Add SNMP target? (y/n): " ADD_SNMP
    [[ "$ADD_SNMP" != "y" ]] && break

    read -rp "SNMP Host/IP: " SNMP_HOST
    read -rp "SNMP Port (default 161): " SNMP_PORT
    SNMP_PORT=${SNMP_PORT:-161}

    # Append an entry with host and port only
    SNMP_TARGETS_YAML+="            - host: \"$SNMP_HOST\""$'\n'
    SNMP_TARGETS_YAML+="              port: $SNMP_PORT"$'\n'
done

# If no targets provided, warn
if [[ -z "${SNMP_TARGETS_YAML//[[:space:]]/}" ]]; then
    echo "[WARN] No SNMP targets were added. You can edit $CONFIG_FILE after install to add targets."
fi

# Shared SNMP authentication (single block)
read -rp "SNMP Protocol Version (default SNMPv2c): " SNMP_VERSION
SNMP_VERSION=${SNMP_VERSION:-SNMPv2c}
read -rp "SNMP Community (default public): " SNMP_COMMUNITY
SNMP_COMMUNITY=${SNMP_COMMUNITY:-public}

# ------------------------------
# SNMP Defaults
# ------------------------------
read -rp "SNMP Site (default: enseva-labs): " SNMP_SITE
SNMP_SITE=${SNMP_SITE:-enseva-labs}
read -rp "SNMP Role (default: sysadmin): " SNMP_ROLE
SNMP_ROLE=${SNMP_ROLE:-sysadmin}
read -rp "SNMP Location (default: suite-aa): " SNMP_LOCATION
SNMP_LOCATION=${SNMP_LOCATION:-suite-aa}
read -rp "Device Manufacturer (default: linux): " SNMP_MANUFACTURER
SNMP_MANUFACTURER=${SNMP_MANUFACTURER:-linux}
read -rp "Device Model (e.g., .1.3.6.1.4.1.8072.3.2.10) (default provided): " SNMP_MODEL
SNMP_MODEL=${SNMP_MODEL:-.1.3.6.1.4.1.8072.3.2.10}

# ------------------------------
# Network Discovery targets
# ------------------------------
NETWORK_TARGETS_YAML=""
read -rp "Enter Network Discovery Targets (comma separated, e.g., 192.168.0.1,10.0.0.0/24) or leave blank: " NETWORK_TARGETS_INPUT
if [[ -n "${NETWORK_TARGETS_INPUT// }" ]]; then
    IFS=',' read -ra NETWORK_TARGETS_ARRAY <<< "$NETWORK_TARGETS_INPUT"
    for target in "${NETWORK_TARGETS_ARRAY[@]}"; do
        NETWORK_TARGETS_YAML+="            - $target"$'\n'
    done
else
    NETWORK_TARGETS_YAML+="            - 192.168.0.0/24"$'\n'
fi

# ------------------------------
# Device Discovery (optional)
# ------------------------------
DEVICE_DISCOVERY_YAML=""
while true; do
    read -rp "Add a device for device_discovery? (y/n): " ADD_DEVICE
    [[ "$ADD_DEVICE" != "y" ]] && break

    read -rp "Device hostname/IP: " DEV_HOST
    read -rp "Device username: " DEV_USER
    read -rp "Device password: " DEV_PASS

    DEVICE_DISCOVERY_YAML+=$(cat <<EOF
          - hostname: $DEV_HOST
            driver: ios
            prompt: '[#$]'
            username: $DEV_USER
            password: "$DEV_PASS"
            read_timeout: 30
            fast_mode: true
            max_retries: 0
            session_log: /tmp/netmiko.log
            tags: $DEV_TAGS_YAML
EOF
)
    DEVICE_DISCOVERY_YAML+=$'\n'
done

# ------------------------------
# Create ORB agent config YAML
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
$SNMP_TARGETS_YAML
          authentication:
            protocol_version: "$SNMP_VERSION"
            community: "$SNMP_COMMUNITY"

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

# ------------------------------
# Validate YAML
# ------------------------------
echo "[*] Validating YAML..."
python3 - <<PY
import sys, yaml
try:
    with open("$CONFIG_FILE", "r") as f:
        yaml.safe_load(f)
except Exception as e:
    print("[ERROR] YAML validation failed:", e)
    sys.exit(1)
print("[+] YAML valid")
PY

echo "[+] ORB agent config created and validated at $CONFIG_FILE"

# ------------------------------
# Pull and run ORB Docker image
# ------------------------------
echo "--- Pulling ORB agent Docker image (if needed) ---"
if ! docker image inspect netboxlabs/orb-agent:latest >/dev/null 2>&1; then
    docker pull --disable-content-trust netboxlabs/orb-agent:latest
else
    echo "[*] ORB agent image already exists."
fi

echo "[*] Starting ORB agent Docker container..."
# Use user-chosen docker network; if user selected "host" network that will be used
if [[ "$DOCKER_NETWORK" == "host" ]]; then
    DOCKER_NET_ARG="--network host"
else
    DOCKER_NET_ARG="--network $DOCKER_NETWORK"
fi

sudo docker run -d \
  --name "$ORB_AGENT_NAME" \
  --restart unless-stopped \
  $DOCKER_NET_ARG \
  -v "$CONFIG_FILE":/etc/orb-agent/config.yaml:ro \
  netboxlabs/orb-agent:latest \
  run --config /etc/orb-agent/config.yaml

echo "[+] ORB agent $ORB_AGENT_NAME is running!"
``
