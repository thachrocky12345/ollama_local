# Ollama 0.4.0 Deployment Guide for Ubuntu Server

This guide provides step-by-step instructions for deploying Ollama version 0.4.0 on Ubuntu Server.

## Prerequisites

- Ubuntu Server 18.04 or later (20.04 LTS, 22.04 LTS recommended)
- Minimum 8GB RAM (16GB+ recommended for larger models)
- Internet connection for downloading models
- Root or sudo access

## Quick Installation

### Option 1: Automated Installation Script (Recommended)

```bash
# Install Ollama 0.4.0 using the official installation script
curl -fsSL https://ollama.com/install.sh | OLLAMA_VERSION=0.4.0 sh
```

### Option 2: Manual Installation

```bash
# Download the specific version
curl -L https://github.com/ollama/ollama/releases/download/v0.4.0/ollama-linux-amd64.tgz -o ollama-linux-amd64.tgz

# Extract to system directory
sudo tar -C /usr -xzf ollama-linux-amd64.tgz

# Make it executable
sudo chmod +x /usr/bin/ollama

# Verify installation
ollama -v
```

## Setting Up Ollama as a System Service

### 1. Create System User

```bash
# Create dedicated user for Ollama service
sudo useradd -r -s /bin/false -U -m -d /usr/share/ollama ollama

# Add your user to ollama group
sudo usermod -a -G ollama $(whoami)
```

### 2. Create Systemd Service

Create the service file:

```bash
sudo tee /etc/systemd/system/ollama.service > /dev/null <<EOF
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
ExecStart=/usr/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=3
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="OLLAMA_HOST=0.0.0.0"

[Install]
WantedBy=default.target
EOF
```

### 3. Start and Enable Service

```bash
# Reload systemd and enable the service
sudo systemctl daemon-reload
sudo systemctl enable ollama
sudo systemctl start ollama

# Check service status
sudo systemctl status ollama
```

## GPU Support (Optional)

### NVIDIA GPU Support

If you have an NVIDIA GPU:

```bash
# Install NVIDIA drivers and CUDA (if not already installed)
sudo apt update
sudo apt install -y nvidia-driver-535 nvidia-cuda-toolkit

# Verify GPU detection
nvidia-smi

# Restart ollama service to detect GPU
sudo systemctl restart ollama
```

### AMD GPU Support

For AMD GPUs, install ROCm support:

```bash
# Download additional ROCm package for version 0.4.0
curl -L https://github.com/ollama/ollama/releases/download/v0.4.0/ollama-linux-amd64-rocm.tgz -o ollama-linux-amd64-rocm.tgz

# Extract ROCm libraries
sudo tar -C /usr -xzf ollama-linux-amd64-rocm.tgz

# Restart service
sudo systemctl restart ollama
```

## Network Configuration

### Firewall Setup

```bash
# Allow Ollama port through UFW firewall
sudo ufw allow 11434/tcp

# Or for specific IP ranges
sudo ufw allow from 192.168.1.0/24 to any port 11434
```

### Reverse Proxy (Optional)

For production deployments, consider using nginx:

```bash
# Install nginx
sudo apt install -y nginx

# Create nginx configuration
sudo tee /etc/nginx/sites-available/ollama > /dev/null <<EOF
server {
    listen 80;
    server_name your-server-domain.com;
    
    location / {
        proxy_pass http://127.0.0.1:11434;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF

# Enable site
sudo ln -s /etc/nginx/sites-available/ollama /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

## Usage

### Basic Usage

```bash
# Download and run a model
ollama pull llama2

# Start interactive chat
ollama run llama2

# Run with custom prompt
ollama run llama2 "Explain quantum computing"
```

### API Usage

```bash
# Test API endpoint
curl http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "Why is the sky blue?",
  "stream": false
}'
```

## Management Commands

### Service Management

```bash
# View service logs
sudo journalctl -u ollama -f

# Stop service
sudo systemctl stop ollama

# Start service
sudo systemctl start ollama

# Restart service
sudo systemctl restart ollama
```

### Model Management

```bash
# List installed models
ollama list

# Remove a model
ollama rm model-name

# Update Ollama (preserves models)
curl -fsSL https://ollama.com/install.sh | OLLAMA_VERSION=0.4.0 sh
```

## Troubleshooting

### Common Issues

1. **Service won't start**
   ```bash
   # Check logs
   sudo journalctl -u ollama --no-pager -l
   
   # Check permissions
   sudo chown -R ollama:ollama /usr/share/ollama
   ```

2. **Out of memory errors**
   ```bash
   # Increase swap space
   sudo fallocate -l 4G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   ```

3. **GPU not detected**
   ```bash
   # Check GPU status
   nvidia-smi  # For NVIDIA
   rocm-smi    # For AMD
   
   # Restart service
   sudo systemctl restart ollama
   ```

### Performance Tuning

```bash
# Set environment variables in service file
sudo systemctl edit ollama

# Add these lines in the override file:
[Service]
Environment="OLLAMA_NUM_PARALLEL=2"
Environment="OLLAMA_MAX_LOADED_MODELS=2"
Environment="OLLAMA_FLASH_ATTENTION=1"
```

## Security Considerations

- Ollama runs on port 11434 by default
- Consider using a reverse proxy with SSL/TLS for production
- Restrict access using firewall rules
- Run with dedicated system user (not root)
- Regularly update to latest security patches

## Uninstallation

```bash
# Stop and disable service
sudo systemctl stop ollama
sudo systemctl disable ollama

# Remove service file
sudo rm /etc/systemd/system/ollama.service

# Remove binary
sudo rm /usr/bin/ollama

# Remove user data and user account
sudo rm -rf /usr/share/ollama
sudo userdel ollama
sudo groupdel ollama

# Reload systemd
sudo systemctl daemon-reload
```

## Version Information

- **Ollama Version**: 0.4.0
- **Supported Ubuntu**: 18.04+, 20.04 LTS, 22.04 LTS
- **Architecture**: x86_64 (amd64), ARM64
- **Default Port**: 11434
- **Service User**: ollama

## Resources

- [Official Ollama Documentation](https://github.com/ollama/ollama)
- [Model Library](https://ollama.ai/library)
- [API Documentation](https://github.com/ollama/ollama/blob/main/docs/api.md)
- [Troubleshooting Guide](https://github.com/ollama/ollama/blob/main/docs/troubleshooting.md)

---

*This deployment guide is specific to Ollama version 0.4.0. For other versions, please refer to the official documentation.*