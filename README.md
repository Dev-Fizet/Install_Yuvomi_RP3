# YUVOMI SELF-HOSTED SERVER
### Raspberry Pi 3 · Docker · Tailscale · Mealie

![Platform](https://img.shields.io/badge/Platform-Raspberry%20Pi-red)
![OS](https://img.shields.io/badge/OS-Raspberry%20Pi%20OS%2064--bit-c51a4a)
![Docker](https://img.shields.io/badge/Container-Docker-2496ED)
![Remote Access](https://img.shields.io/badge/Remote%20Access-Tailscale-ffffff)

A complete installation guide to deploy **Yuvomi** on a Raspberry Pi 3 using Docker Compose, with secure remote access through Tailscale and an additional Mealie recipe management service.

The objective is to transform a Raspberry Pi into a lightweight, always-on personal server that can be accessed from anywhere.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Hardware & Requirements](#2-hardware--requirements)
3. [Install Raspberry Pi OS](#3-install-raspberry-pi-os)
4. [Enable SSH Access](#4-enable-ssh-access)
5. [Prepare the Raspberry Pi](#5-prepare-the-raspberry-pi)
6. [Install Docker](#6-install-docker)
7. [Deploy Yuvomi](#7-deploy-yuvomi)
8. [Configure Remote Access with Tailscale](#8-configure-remote-access-with-tailscale)
9. [Deploy Mealie](#9-deploy-mealie)
10. [Service Management](#10-service-management)
11. [Troubleshooting](#11-troubleshooting)
12. [Security & Maintenance](#12-security--maintenance)

---

# 1. Project Overview

This project deploys two self-hosted applications on a Raspberry Pi 3.

| Service | Description | Local Port |
|---|---|---|
| Yuvomi | Main application | `3000` |
| Mealie | Recipe and meal management | `9925` |
| Tailscale | Secure remote connectivity | — |

### Architecture

```text
                 INTERNET
                    |
                    |
              TAILSCALE VPN
                    |
                    |
             RASPBERRY PI 3
             Raspberry Pi OS
                    |
                 DOCKER
                    |
          +---------+---------+
          |                   |
          |                   |
       YUVOMI               MEALIE
      Port 3000            Port 9925
          |                   |
       ./data              ./data
```

Each application runs in its own Docker container and maintains its own persistent data directory.

**Why this architecture?**

- Independent application management.
- Persistent storage.
- Automatic container restart.
- Easy upgrades and maintenance.
- Secure remote access.
- No need to expose application ports directly to the internet.

---

# 2. Hardware & Requirements

### Hardware

- Raspberry Pi 3 (preferably Model B or B+).
- MicroSD card (16 GB minimum; 32 GB recommended).
- Stable power supply.
- Internet connection.
- Ethernet connection recommended for server stability.

### Software

- Raspberry Pi OS Lite (64-bit).
- SSH client.
- Docker Engine.
- Docker Compose plugin.
- Tailscale account.

> **Important:** Raspberry Pi 3 has limited RAM and processing power. Running multiple applications simultaneously may affect performance. A 64-bit operating system is required for this installation path, and ARM64 compatibility of the selected container images should be verified before deployment.

---

# 3. Install Raspberry Pi OS

The first step is to install the operating system on the Raspberry Pi.

### Recommended operating system

**Raspberry Pi OS Lite (64-bit)**

Lite is recommended because it does not include a graphical desktop environment, reducing resource consumption.

### Installation

1. Download and install Raspberry Pi Imager.
2. Insert the microSD card into your computer.
3. Select Raspberry Pi 3 as the device.
4. Select Raspberry Pi OS Lite (64-bit).
5. Select the microSD card.
6. Configure the operating system settings.

Official download:

https://www.raspberrypi.com/software/

### Recommended initial configuration

Before writing the image, configure:

- Hostname: `yuvomi-server`
- Username: Choose your preferred username.
- Password: Set a strong password.
- Wi-Fi: Configure if Ethernet is not available.
- SSH: Enable SSH authentication.

Write the image and insert the microSD card into your Raspberry Pi.

Connect the power supply and wait for the system to boot.

---

# 4. Enable SSH Access

SSH allows us to manage the Raspberry Pi remotely from another computer without connecting a monitor or keyboard.

If SSH was not enabled during the OS installation, connect a monitor and keyboard and execute:

```bash
sudo raspi-config
```

Navigate to:

```text
Interface Options
    └── SSH
          └── Enable
```

Select **Yes** to enable the SSH server.

Exit the configuration utility.

### Verify SSH

Execute:

```bash
sudo systemctl status ssh
```

The expected status is:

```text
Active: active (running)
```

### Connect remotely

From another computer on the same network:

```bash
ssh YOUR_USERNAME@RASPBERRY_PI_IP
```

Replace `YOUR_USERNAME` and `RASPBERRY_PI_IP` with your actual credentials and Raspberry Pi address.

You can find the IP address by running:

```bash
hostname -I
```

**Checkpoint:** You should now have terminal access to the Raspberry Pi through SSH.

---

# 5. Prepare the Raspberry Pi

Before installing Docker, update the operating system.

Execute:

```bash
sudo apt update
sudo apt full-upgrade -y
```

Install the required utilities:

```bash
sudo apt install -y curl ca-certificates nano
```

Check the system architecture:

```bash
uname -m
```

For a 64-bit installation, the expected result is:

```text
aarch64
```

Check available disk space:

```bash
df -h
```

Finally, reboot the Raspberry Pi:

```bash
sudo reboot
```

Reconnect through SSH after the system restarts.

---

# 6. Install Docker

Docker will be responsible for running Yuvomi and Mealie independently.

### 6.1 Install Docker Engine

Execute the official convenience installation script:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
```

Run the installation:

```bash
sudo sh get-docker.sh
```

Wait until the installation finishes.

### 6.2 Enable Docker at startup

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Verify the service:

```bash
sudo systemctl status docker
```

Expected status:

```text
Active: active (running)
```

### 6.3 Add your user to the Docker group

This allows you to execute Docker commands without typing `sudo` every time.

```bash
sudo usermod -aG docker $USER
```

Apply the new group membership:

```bash
newgrp docker
```

Alternatively, log out and reconnect through SSH.

> **Security note:** Membership in the Docker group effectively grants root-level control over the host. Only add trusted users.

### 6.4 Verify Docker installation

Check Docker Engine:

```bash
docker --version
```

Check Docker Compose:

```bash
docker compose version
```

Run a test container:

```bash
docker run hello-world
```

If the test completes successfully, Docker is ready.

---

# 7. Deploy Yuvomi

Now that Docker is installed, we can deploy Yuvomi.

We will create an independent project directory and use Docker Compose to manage the container.

### 7.1 Create the project directory

```bash
mkdir -p ~/yuvomi
cd ~/yuvomi
```

### 7.2 Create the Docker Compose configuration

Create the configuration file:

```bash
nano docker-compose.yml
```

Paste the following configuration:

```yaml
services:
  yuvomi:
    image: ghcr.io/ulsklyc/yuvomi:latest
    container_name: yuvomi
    restart: unless-stopped

    ports:
      - "3000:3000"

    environment:
      TZ: America/Mexico_City

    volumes:
      - ./data:/app/data
```

Save the file:

- `CTRL + O` → Save.
- `ENTER` → Confirm.
- `CTRL + X` → Exit.

### 7.3 Validate the configuration

```bash
docker compose config
```

If Docker Compose displays the configuration without errors, continue.

### 7.4 Start Yuvomi

Download the image and start the container:

```bash
docker compose up -d
```

Check the running containers:

```bash
docker ps
```

You should see a container named:

```text
yuvomi
```

Check its logs:

```bash
docker compose logs -f
```

Press `CTRL + C` to exit the log viewer without stopping the container.

### 7.5 Access Yuvomi

From a device connected to the same local network, open:

```text
http://RASPBERRY_PI_IP:3000
```

Replace `RASPBERRY_PI_IP` with the actual IP address of your Raspberry Pi.

**Checkpoint:** Yuvomi should now be running inside Docker.

---

# 8. Configure Remote Access with Tailscale

We want to access Yuvomi even when we are away from home.

Instead of opening ports on the router or exposing the Raspberry Pi directly to the internet, we will use Tailscale.

Tailscale creates a secure private network between your devices.

### 8.1 Install Tailscale

Execute:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Wait for the installation to complete.

### 8.2 Connect the Raspberry Pi to your Tailscale network

Run:

```bash
sudo tailscale up
```

The terminal will display an authentication URL.

1. Copy the URL.
2. Open it in your browser.
3. Sign in to your Tailscale account.
4. Authorize the Raspberry Pi.

Once authenticated, your Raspberry Pi will be connected to your private Tailscale network.

### 8.3 Find the Tailscale IP address

Execute:

```bash
tailscale ip -4
```

You should receive an IP address similar to:

```text
100.x.x.x
```

This is your Raspberry Pi's private Tailscale IP.

### 8.4 Access Yuvomi from anywhere

From another device connected to the same Tailscale network, open:

```text
http://TAILSCALE_IP:3000
```

Replace `TAILSCALE_IP` with the address returned by the previous command.

You can now access Yuvomi remotely without opening ports on your home router.

Official documentation:

https://tailscale.com/kb/

---

# 9. Deploy Mealie

Mealie is a self-hosted recipe manager that allows you to organize recipes, import recipes from websites, and plan weekly meals.

We will deploy it as an independent Docker Compose project.

### 9.1 Create the project directory

```bash
mkdir -p ~/mealie
cd ~/mealie
```

### 9.2 Create the Docker Compose configuration

```bash
nano docker-compose.yml
```

Paste the following configuration:

```yaml
services:
  mealie:
    image: ghcr.io/mealie-recipes/mealie:latest
    container_name: mealie
    restart: unless-stopped

    ports:
      - "9925:9000"

    environment:
      TZ: America/Mexico_City
      BASE_URL: http://localhost:9925

    volumes:
      - ./data:/app/data
```

Save and exit:

- `CTRL + O` → Save.
- `ENTER` → Confirm.
- `CTRL + X` → Exit.

> **Configuration note:** The `BASE_URL` value above is a local default. If you configure Mealie behind a reverse proxy or a custom domain, update it to the appropriate externally accessible URL. Do not use `localhost` as the external URL in that scenario.

### 9.3 Validate the configuration

```bash
docker compose config
```

### 9.4 Start Mealie

```bash
docker compose up -d
```

Verify the container:

```bash
docker ps
```

You should see:

```text
mealie
```

Check the logs:

```bash
docker compose logs -f
```

### 9.5 Access Mealie

From your local network:

```text
http://RASPBERRY_PI_IP:9925
```

From outside your home using Tailscale:

```text
http://TAILSCALE_IP:9925
```

Complete the initial Mealie setup through its web interface.

Official documentation:

https://docs.mealie.io/

---

# 10. Service Management

Both applications run independently through Docker Compose.

This makes it easy to start, stop, restart, update, and inspect each application.

## Yuvomi

Navigate to the project directory:

```bash
cd ~/yuvomi
```

Start:

```bash
docker compose up -d
```

Stop:

```bash
docker compose down
```

Restart:

```bash
docker compose restart
```

View logs:

```bash
docker compose logs -f
```

Update the image:

```bash
docker compose pull
docker compose up -d
```

## Mealie

Navigate to the project directory:

```bash
cd ~/mealie
```

Start:

```bash
docker compose up -d
```

Stop:

```bash
docker compose down
```

Restart:

```bash
docker compose restart
```

View logs:

```bash
docker compose logs -f
```

Update the image:

```bash
docker compose pull
docker compose up -d
```

> **Important:** Updating a Docker image does not replace the need for backups. Always back up persistent application data before major upgrades.

---

# 11. Troubleshooting

### Docker permission denied

If you receive a permission error when executing Docker commands:

```bash
sudo usermod -aG docker $USER
```

Log out and reconnect through SSH.

### Container is not running

Check the container status:

```bash
docker ps -a
```

Inspect the logs:

```bash
docker logs yuvomi
```

For Mealie:

```bash
docker logs mealie
```

### Port already in use

Check which process is using a port:

```bash
sudo ss -tulpn
```

Make sure ports `3000` and `9925` are available.

### Cannot access Yuvomi or Mealie

Check the following:

1. The Raspberry Pi is powered on.
2. The container is running.
3. The correct IP address is being used.
4. The correct port is being used.
5. The device is connected to the same local network or Tailscale network.

Check Docker:

```bash
docker ps
```

### Tailscale is not connected

Check its status:

```bash
tailscale status
```

Reconnect if necessary:

```bash
sudo tailscale up
```

### Check application logs

Yuvomi:

```bash
cd ~/yuvomi
docker compose logs --tail=100
```

Mealie:

```bash
cd ~/mealie
docker compose logs --tail=100
```

---

# 12. Security & Maintenance

A self-hosted server should be maintained regularly.

### Recommended security practices

- Use a strong password for the Raspberry Pi account.
- Keep Raspberry Pi OS updated.
- Keep Docker updated.
- Use Tailscale for remote access.
- Avoid exposing application ports directly to the public internet.
- Do not share your Tailscale authentication links or credentials.
- Back up application data regularly.

### Persistent data

All application data is stored locally:

```text
/home/YOUR_USERNAME/
│
├── yuvomi/
│   ├── docker-compose.yml
│   └── data/
│
└── mealie/
    ├── docker-compose.yml
    └── data/
```

These directories should be included in your backup strategy.

### Update the operating system

Run periodically:

```bash
sudo apt update
sudo apt full-upgrade -y
```

Reboot when necessary:

```bash
sudo reboot
```

---

# Final Result

Congratulations! Your Raspberry Pi 3 is now configured as a self-hosted server.

### Installed services

| Application | Access |
|---|---|
| Yuvomi | `http://RASPBERRY_PI_IP:3000` |
| Mealie | `http://RASPBERRY_PI_IP:9925` |
| Tailscale | Private remote network |
| Docker | Container management |

### Final checklist

- [ ] Raspberry Pi OS Lite (64-bit) installed.
- [ ] SSH enabled and verified.
- [ ] System updated.
- [ ] Docker Engine installed.
- [ ] Docker Compose verified.
- [ ] Yuvomi deployed successfully.
- [ ] Tailscale connected.
- [ ] Remote access tested.
- [ ] Mealie deployed successfully.
- [ ] Persistent data directories verified.
- [ ] Backups configured.

---

**Project:** Yuvomi Self-Hosted Server  
**Hardware:** Raspberry Pi 3  
**Operating System:** Raspberry Pi OS Lite (64-bit)  
**Container Runtime:** Docker Engine + Docker Compose  
**Remote Access:** Tailscale  
**Additional Application:** Mealie

*Build your own private server. Keep your applications, your data, and your infrastructure under your control.*