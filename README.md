## Sepolia Geth + Lighthouse Sequencer Setup Guide

This guide walks you through manually setting up an Ethereum Sepolia testnet node with Geth (execution client) and Lighthouse (consensus client).

## System Requirements

- Ubuntu Linux (recommended)
- At least 1TB SSD storage
- Minimum 16GB RAM
- Open ports: 8545, 30303, 8551, 5052

## Step 1: Install Dependencies

First, we need to ensure all required software is installed on your system.

### Install Docker and Docker Compose

```bash
# Remove any existing Docker installations
sudo apt-get remove -y docker.io docker-doc docker-compose podman-docker containerd runc

# Set up the Docker repository
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package lists and install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify Docker installation
sudo docker run hello-world

# Enable and start Docker service
sudo systemctl enable docker
sudo systemctl restart docker
```

### Install Other Dependencies

```bash
sudo apt update
sudo apt install -y curl openssl
```

## Step 2: Prepare Directories

Create directories to store blockchain data:

```bash
# Create main data directory
DATA_DIR="$HOME/sepolia-node"
mkdir -p "$DATA_DIR"

# Create directories for Geth and Lighthouse
mkdir -p "$DATA_DIR/geth"
mkdir -p "$DATA_DIR/lighthouse"
```

## Step 3: Generate JWT Secret

Geth and Lighthouse need a shared JWT secret to communicate securely:

```bash
# Generate a random 32-byte hexadecimal string
openssl rand -hex 32 > "$DATA_DIR/jwt.hex"
```

## Step 4: Create Docker Compose File

Create a `docker-compose.yml` file to orchestrate the Geth and Lighthouse containers:

```bash
cat > "$DATA_DIR/docker-compose.yml" << 'EOF'
version: '3.8'
services:
  geth:
    image: ethereum/client-go:stable
    container_name: geth
    restart: unless-stopped
    volumes:
      - ./geth:/root/.ethereum
      - ./jwt.hex:/root/jwt.hex
    ports:
      - "8545:8545"
      - "30303:30303"
      - "8551:8551"
    command: >
      --sepolia
      --http --http.addr 0.0.0.0 --http.api eth,web3,net,engine
      --authrpc.addr 0.0.0.0 --authrpc.port 8551
      --authrpc.jwtsecret /root/jwt.hex
      --authrpc.vhosts=*
      --http.corsdomain="*"
  lighthouse:
    image: sigp/lighthouse:latest
    container_name: lighthouse
    restart: unless-stopped
    volumes:
      - ./lighthouse:/root/.lighthouse
      - ./jwt.hex:/root/jwt.hex
    ports:
      - "5052:5052"
    depends_on:
      - geth
    command: >
      lighthouse bn
      --network sepolia
      --execution-endpoint http://geth:8551
      --execution-jwt /root/jwt.hex
      --checkpoint-sync-url https://sepolia.checkpoint-sync.ethpandaops.io
      --http
      --http-address 0.0.0.0
EOF
```

## Step 5: Start the Services

Before starting, check if port 8545 is already in use:

```bash
# Check if port 8545 is already in use
if lsof -i :8545 >/dev/null 2>&1; then
  echo "Port 8545 is already in use. Please stop the process using it before continuing."
  exit 1
fi
```

If the port is free, start the containers:

```bash
# Navigate to your data directory
cd "$DATA_DIR"

# Start the containers in detached mode
docker compose up -d
```

## Step 6: Verify the Setup

Check if the containers are running:

```bash
docker ps
```

Monitor the logs of both containers:

```bash
# Geth logs
docker logs -f geth

# Lighthouse logs
docker logs -f lighthouse
```

## Step 7: Check Sync Status

Check if the nodes are syncing properly:

```bash
# Check Geth sync status
curl -s -X POST http://localhost:8545 \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'

# Check Lighthouse health
curl http://localhost:5052/eth/v1/node/health
```

## Maintenance

### Stopping the Services

```bash
cd "$DATA_DIR"
docker compose down
```

### Updating the Services

```bash
cd "$DATA_DIR"
docker compose pull
docker compose down
docker compose up -d
```

### Checking Disk Usage

```bash
du -h "$DATA_DIR"
```

## Troubleshooting

1. If containers fail to start, check logs:
   ```bash
   docker logs geth
   docker logs lighthouse
   ```

2. If the system runs out of disk space:
   ```bash
   # Check available disk space
   df -h
   ```

3. If you encounter network connectivity issues:
   ```bash
   # Check if ports are open
   sudo lsof -i :8545
   sudo lsof -i :30303
   sudo lsof -i :8551
   sudo lsof -i :5052
   ```

## Additional Resources

- [Geth Documentation](https://geth.ethereum.org/docs/)
- [Lighthouse Documentation](https://lighthouse-book.sigmaprime.io/)
- [Sepolia Testnet Info](https://sepolia.dev/)
