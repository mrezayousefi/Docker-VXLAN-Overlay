# Docker Overlay Network with VXLAN Tunneling

This project demonstrates how to connect two containers running on separate virtual machines (VMs) using a Docker Overlay network with VXLAN tunneling, without publishing any ports publicly. The setup ensures secure and isolated communication between containers across VMs.

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [Step 1: Configure VXLAN Tunnel](#step-1-configure-vxlan-tunnel)
  - [Step 2: Initialize Docker Swarm](#step-2-initialize-docker-swarm)
  - [Step 3: Create Overlay Network](#step-3-create-overlay-network)
  - [Step 4: Deploy Containers](#step-4-deploy-containers)
  - [Step 5: Test Connectivity](#step-5-test-connectivity)
- [Network Diagram](#network-diagram)
- [Important Notes](#important-notes)
- [Contributing](#contributing)
- [License](#license)

## Overview
The goal of this project is to establish communication between two containers running on different VMs (VM1: `185.231.59.229`, VM2: `46.249.101.68`) using Docker's Overlay network and VXLAN tunneling. This approach avoids exposing container ports publicly, ensuring secure and scalable communication.

Key components:
- **VXLAN**: Creates a layer 2 tunnel between VMs to encapsulate container traffic.
- **Docker Swarm**: Manages the Overlay network for seamless container communication.
- **Nginx Containers**: Used as example services to demonstrate connectivity.

## Prerequisites
- Two VMs running a Linux-based OS (e.g., Ubuntu) with the following IPs:
  - VM1: `185.231.59.229`
  - VM2: `46.249.101.68`
- Docker installed on both VMs (`sudo apt install docker.io`).
- Root or sudo access on both VMs.
- Network connectivity between VMs (test with `ping`).
- `iproute2` package for VXLAN configuration (`sudo apt install iproute2`).
- Firewall rules allowing ports:
  - 4789/UDP (VXLAN)
  - 2377/TCP (Swarm management)
  - 7946/TCP and 7946/UDP (Swarm node discovery)

Example firewall setup:
```bash
sudo ufw allow 4789/udp
sudo ufw allow 2377/tcp
sudo ufw allow 7946/tcp
sudo ufw allow 7946/udp
```

## Setup Instructions

### Step 1: Configure VXLAN Tunnel
Create a VXLAN tunnel to enable layer 2 connectivity between VMs.

**On VM1 (`185.231.59.229`)**:
```bash
sudo ip link add vxlan10 type vxlan id 10 remote 46.249.101.68 local 185.231.59.229 dstport 4789
sudo ip link set vxlan10 up
sudo ip addr add 192.168.10.1/24 dev vxlan10
```

**On VM2 (`46.249.101.68`)**:
```bash
sudo ip link add vxlan10 type vxlan id 10 remote 185.231.59.229 local 46.249.101.68 dstport 4789
sudo ip link set vxlan10 up
sudo ip addr add 192.168.10.2/24 dev vxlan10
```

**Test the tunnel**:
- From VM1: `ping 192.168.10.2`
- From VM2: `ping 192.168.10.1`

### Step 2: Initialize Docker Swarm
Set up Docker Swarm to manage the Overlay network.

**On VM1 (Swarm Manager)**:
```bash
docker swarm init --advertise-addr 192.168.10.1
```
Copy the generated `docker swarm join` command (including the token).

**On VM2 (Swarm Worker)**:
Run the `docker swarm join` command from VM1, e.g.:
```bash
docker swarm join --token <TOKEN> 192.168.10.1:2377
```

**Verify Swarm**:
On VM1:
```bash
docker node ls
```
Ensure both VMs are listed (one as `Leader`, one as `Worker`).

### Step 3: Create Overlay Network
Create a Docker Overlay network for container communication.

**On VM1**:
```bash
docker network create --driver overlay --subnet 10.0.0.0/24 my-overlay-network
```

**Verify Network**:
On both VMs:
```bash
docker network ls
```
The `my-overlay-network` should appear on both nodes.

### Step 4: Deploy Containers
Run Nginx containers on both VMs, connected to the Overlay network.

**On VM1**:
```bash
docker run -d --name container1 --network my-overlay-network nginx
```

**On VM2**:
```bash
docker run -d --name container2 --network my-overlay-network nginx
```

**Check Container IPs**:
On VM1:
```bash
docker inspect container1 | grep IPAddress
```
On VM2:
```bash
docker inspect container2 | grep IPAddress
```
Example IPs: `10.0.0.2` (container1), `10.0.0.3` (container2).

### Step 5: Test Connectivity
Verify communication between containers.

**From VM1**:
```bash
docker exec container1 ping 10.0.0.3
docker exec container1 curl http://10.0.0.3
```

A successful `ping` and `curl` response (Nginx welcome page) confirms the setup.

## Network Diagram
```
VM1 (185.231.59.229)          VM2 (46.249.101.68)
   |                             |
   |--- container1 (10.0.0.2)    |--- container2 (10.0.0.3)
   |                             |
   |--- vxlan10 (192.168.10.1)   |--- vxlan10 (192.168.10.2)
   |                             |
   +--- Docker Swarm (Overlay Network: my-overlay-network) ---+
```

## Important Notes
- **No Port Publishing**: Containers communicate securely via the Overlay network and VXLAN, without exposing ports publicly.
- **Security**: Consider encrypting VXLAN traffic with IPsec or WireGuard for production environments.
- **Scalability**: This setup can be extended to additional VMs by joining them to the Swarm and connecting to the Overlay network.
- **Firewall**: Ensure the required ports are open to avoid connectivity issues.
- **Troubleshooting**:
  - Verify VXLAN tunnel with `ping`.
  - Check Swarm status with `docker node ls`.
  - Inspect container logs with `docker logs <container>`.

## Contributing
Contributions are welcome! Please submit a pull request or open an issue for suggestions, bug reports, or improvements.

## License
This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
