
# E2E 5G Testbed Setup Guide

This guide explains how to set up a 5G testbed using Open5GS and UERANSIM on a single Ubuntu VM. It includes installation, configuration, running the network, and troubleshooting.

---


## 1. Prerequisites

### ⚙️ System Requirements
### System Requirements
- 🖥 OS: Ubuntu 24.04
- 💾 RAM: 4GB+ recommended
- 🗄 Disk: 20GB free
- 🌐 Network: Ethernet/Wi-Fi


### Software
- Docker & Docker Compose
- Python3
- Open5GS v2.7.2
- UERANSIM v3.2.7

---

## 2. 📁 Folder Structure

```

5G-standard-full-stack/
│
├── core/          # Contains AMF, SMF, UPF YAML configs
├── ran/           # Contains gNB and UE YAML configs
├── docs/          # Documentation & screenshots
├── open5gs/       # Full Open5GS installation folder
├── UERANSIM/      # Full UERANSIM installation folder
└── SETUP_GUIDE.md

```

## 3. 🐳 Docker Setup
The 5G core network (Open5GS) runs inside Docker containers. The steps below help you set up and run the core network.

1. Build and run Open5GS containers:

```bash
docker-compose -f sa-deploy.yaml build
docker-compose -f sa-deploy.yaml up -d
```
2. Verify containers are running:

```bash
docker ps
```
You should see containers for **AMF, SMF, UPF, NRF**, and other core components.

3. Use host-networking for containers:

```bash
 docker run --network host docker_open5gs
```
**⚠️ **Note****: Using host networking ensures that IP addresses in your YAML configuration match those used by the containers.

## 4. TUN interface and NAT
The **TUN interface** is used to simulate the data plane for 5G UE traffic. NAT allows packets from the virtual interface to reach the internet.
### 4.1 Create TUN device `ogstun`:

```bash
sudo ip tuntap add name ogstun mode tun
sudo ip addr add 10.45.0.1/16 dev ogstun
sudo ip link set ogstun up
```
### 4.2 Enabling IP forwarding:
```bash
sudo nano /etc/sysctl.conf
# Uncomment or add the line:
net.ipv4.ip_forward=1
sudo sysctl -p
```
### 4.3 Configure NAT:
```bash
sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
```
💡 **Tip**: Verify interface status using `ip addr show ogstun`. It should be **UP** once UE attaches.


## 5. 📝 YAML Configurations
YAML files define the network behavior, IP addresses, PLMN info, and connectivity between core and RAN. Make sure all IPs are consistent across files.
### 5.1 Core (Open5GS)

- `core/amf.yaml`
```yaml
amf:
  sbi:
    server:
      address: 192.168.0.111
      port: 7777

  guami:
    - plmn_id:
        mcc: 001
        mnc: 01
      amf_id:
        region: 2
        set: 1

  ngap:
    server:
      - address: 192.168.0.111

  plmn_support:
    - plmn_id:
        mcc: 001
        mnc: 01

  tai:
    - plmn_id:
        mcc: 001
        mnc: 01
      tac: 1
```
- `core/smf.yaml`
```yaml
smf:
  pfcp:
    server:
      - address: 192.168.0.111

  gtpc:
    server:
      - address: 192.168.0.112

  gtpu:
    server:
      - address: 192.168.0.111

  session:
    - subnet: 10.45.0.1/16
      dnn: internet

  dns:
    - 8.8.8.8
    - 8.8.4.4

```
    
- `core/upf.yaml`
```yaml
upf:
  pfcp:
    server:
      - address: 192.168.0.112

  gtpu:
    server:
      - address: 192.168.0.112

  session:
    - subnet: 10.45.0.1/16
      dnn: internet
      dev: ogstun
```
    


### 5.2 RAN (gNB)

-  `ueransim/config/open5gs-gnb.yaml`
```yaml
mcc: '001'
mnc: '01'

nci: '0x000000010'
idLength: 32
tac: 1

cellBarred: false

linkIp: 192.168.0.131      # gNB binds here
ngapIp: 192.168.0.131    
gtpIp: 192.168.0.131       

# AMF
amfConfigs:
  - address: 192.168.0.111  # AMF IP 
    port: 38412
```
    

### 5.3 UE

-  `ueransim/config/open5gs-gnb.yaml`
```yaml

mcc: '001'
mnc: '01'

gnbSearchList:
  - 192.168.0.131   # matches open5gs-gnb.yaml IMP

```
**⚠️Note**: Always double-check IP addresses and PLMN values between core, gNB, and UE configurations to avoid connection issues.
## 6. Running the Network

### 6.1 Launch gNB
Go to your UERANSIM folder and run `nr-gnb` with your gNB config

`
./build/nr-gnb -c ./config/open5gs-gnb.yaml` 

-   `-c`  specifies the YAML configuration for gNB.
    
-   Keep this terminal open because the gNB needs to run continuously.
    

You should see logs like  `Serving NGAP on ...`  or  `New signal detected for cell`.

### 6.2 Launch UE
Open a **new terminal** and run the UE:

`
sudo ./build/nr-ue -c ./config/open5gs-ue.yaml` 
-   Use  `sudo`  because UE may require access to network interfaces.
    
-   📡  Logs will show UE switching states (if successful) You should see:
```rust
MM-DEREGISTERED -> PLMN-SEARCH -> REGISTERED
```
❌ If it fails, check for **cell selection errors** or **IP mismatches** in your YAML configs (Which is the current issue we are facing)



## ✅ Verify Connectivity

### 6.3 Ping the AMF from your host
    

`ping 192.168.0.111` 

### 6.4 Check if SCTP port is listening (used for NGAP between gNB and AMF):
   
`ss -lntp | grep 38412` 

### 6.5 Check ogstun interafec:
    

`ip addr show ogstun` 

-   It should be  `UP`  once the UE attaches and data plane is active, if the interface is DOWN then it means there is no traffic between the two.




## 7. Troubleshooting Checklist

| Problem                  | Solution                                                                 |
|--------------------------|--------------------------------------------------------------------------|
| UE cannot connect to gNB | Check IP addresses in YAML files, ensure gNB and AMF IP match host network |
| SCTP connection timeout  | Verify Docker containers are using host networking                        |
| TUN interface down       | Ensure gNB/UE are running and `ogstun` interface is UP                   |
| NAT issues               | Check `iptables -t nat` rules and IP forwarding                           |


## 8. 📚 References

- [Open5GS Official Documentation](https://open5gs.org/)
- [UERANSIM GitHub Repository](https://github.com/aligungr/UERANSIM)
- [Docker Documentation](https://docs.docker.com/)
