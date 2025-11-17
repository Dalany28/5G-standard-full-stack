# E2E 5G Testbed Setup Guide

This guide explains how to set up a 5G testbed using Open5GS and UERANSIM on a single Ubuntu VM. It includes installation, configuration, running the network, and troubleshooting.

---

## 1. Prerequisites

### System Requirements
- OS: Ubuntu 24.04
- RAM: 4GB+ recommended
- Disk: 20GB free
- Network: Ethernet/Wi-Fi

### Software
- Docker & Docker Compose
- Python3
- Open5GS v2.7.2
- UERANSIM v3.2.7

---

## 2. Folder Structure

5G-standard-full-stack/
│
├── core/ # Contains AMF, SMF, UPF YAML configs
├── ran/ # Contains gNB and UE YAML configs
├── docs/ # Documentation & screenshots
├── open5gs/ # Full Open5GS installation folder
├── UERANSIM/ # Full UERANSIM installation folder
└── SETUP_GUIDE.md # This guide

---

## 3. Docker Setup

1. Build and run Open5GS containers:
```bash
docker-compose build
docker-compose up -d
Verify containers are running:
docker ps
Use host networking for containers:
docker run --network host docker_open5gs
4. TUN Interface and NAT
Create TUN device ogstun:
sudo ip tuntap add name ogstun mode tun
sudo ip addr add 10.45.0.1/16 dev ogstun
sudo ip link set ogstun up
Enable IP forwarding:
sudo nano /etc/sysctl.conf
# Uncomment or add the line:
net.ipv4.ip_forward=1
sudo sysctl -p
Configure NAT:
sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
Screenshot Placeholder: docs/ogstun_setup.png
5. YAML Configurations
5.1 Core (Open5GS)
core/amf.yaml
core/smf.yaml
core/upf.yaml
Screenshot Placeholder: docs/core_yaml.png
5.2 RAN (gNB)
ran/open5gs-gnb.yaml
5.3 UE
ran/open5gs-ue.yaml
Screenshot Placeholder: docs/ran_yaml.png
6. Running the Network
6.1 Launch gNB
cd UERANSIM
./build/nr-gnb -c ./config/open5gs-gnb.yaml
6.2 Launch UE
cd UERANSIM
sudo ./build/nr-ue -c ./config/open5gs-ue.yaml
6.3 Verify Connectivity
Ping between gNB ↔ AMF:
ping 192.168.0.111
Check SCTP port (38412) is listening:
ss -lntp | grep 38412
Screenshot Placeholder: docs/connection_check.png
7. Troubleshooting Checklist
Problem	Solution
UE cannot connect to gNB	Check IP addresses in YAML files, ensure gNB and AMF IP match host network
SCTP connection timeout	Verify Docker containers are using host networking
TUN interface down	Ensure gNB/UE are running and ogstun interface is UP
NAT issues	Check iptables -t nat rules and IP forwarding
Screenshot Placeholder: docs/ue_error_logs.png
8. Notes
Ensure all YAML IPs are consistent across core, gNB, and UE.
For multi-user testing, update UE IMSI and APN in open5gs-ue.yaml.
Document all errors and resolutions in docs/.
9. References
Open5GS Official Documentation
UERANSIM GitHub
Docker Documentation

---

