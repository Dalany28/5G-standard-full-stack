@ -1 +1,363 @@
# e2e-bc-5g
# e2e-bc-5g

# 🧪 Full Implementation Installation Guide

This document provides a full walkthrough for installing and configuring the 5G testbed environment. It includes hardware and software requirements, step-by-step instructions, and test validation.

---

## 📦 Table of Contents

1. [Overview](#-overview)
2. [System Requirements](#-system-requirements)
3. [Hardware Setup](#-hardware-setup)
4. [Software Installation](#-software-installation)
   - [1. Operating System](#1-operating-system)
   - [2. Dependencies](#2-dependencies)
   - [3. Testbed Software](#3-testbed-software)
5. [Configuration](#-configuration)
6. [Running the Testbed](#-running-the-testbed)
7. [Validation & Testing](#-validation--testing)
8. [Troubleshooting](#-troubleshooting)
9. [References](#-references)

---

## 🧾 Overview

This testbed simulates a distributed IoT environment for E2E 5G trust management.
It includes the following minimum configuration: C-Plane have one U-Planes, U-Plane have one DN. and one UE connects to the DN.
---

## 🖥️ System Requirements

### Tested Specs

- OS: Ubuntu 24.04
- RAM: 1GB
- Disk: 20GB free
- Network: Ethernet/Wi-Fi

---

## 🔌 Hardware Setup

### Components

- Raspberry Pi Zero W
- VMWare Host

### Main Diagram

*(To add the diagram here)*

---

## 🧰 Software Installation

### 1. Operating System

Install Ubuntu Server on all four nodes:

```bash
sudo apt update && sudo apt upgrade -y
```

Flash image to SD card using `Raspberry Pi Imager`.

### 2. Dependencies

Install required packages:

```bash
sudo apt install -y python3 python3-pip docker docker-compose
```

### 3. Network Stack Components

The 5GC, UE, and RAN components used in this testbed are:

- **5GC:** [Open5GS v2.7.2 (2024.08.04)](https://github.com/open5gs/open5gs)
- **UE / RAN:** [UERANSIM v3.2.7 (2025.02.11)](https://github.com/aligungr/UERANSIM)

## 🛠️ Build Instructions

Please refer to the following official guides for building **Open5GS** and **UERANSIM** from source:

- **Open5GS v2.7.2:** [Building Open5GS from Sources](https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/)
- **UERANSIM v3.2.7:** [Installation Guide](https://github.com/aligungr/UERANSIM/wiki/Installation)

---
## 🖥️ Machines Overview

Each VM used in the testbed is described below:

| VM # | Software & Role           | IP Address           | OS            | Memory | HDD  |
|------|----------------------------|-----------------------|----------------|--------|------|
| VM1  | Open5GS 5GC C-Plane       | 10.42.0.100/24     | Ubuntu 24.04   | 4GB    | 30GB |
| VM2  | Open5GS 5GC U-Plane       | 10.42.0.110/24     | Ubuntu 24.04   | 2GB    | 30GB |
| VM3  | UERANSIM RAN (gNodeB)     | 10.42.0.111/24     | Ubuntu 24.04   | 2GB    | 30GB |
| VM4  | UERANSIM UE               | 10.42.0.112/24     | Ubuntu 24.04   | 512MB    | 20GB |

### 📱 UE Information

| UE ID | IMSI              | APN      | Auth Type |
|-------|-------------------|----------|-----------|
| UE0   | 001010000000000   | internet | OPc       |

---

### 🌐 DN (Data Network) Information

| DN Subnet     | Tunnel Interface | DN Interface |
|---------------|------------------|--------------|
| 10.45.0.0/16  | ogstun           | internet     |


# ⚙️ Configuration

## Changes in configuration files of Open5GS 5GC C-Plane

### 🧵 Setting Up TUN Device (Non-Persistent After Reboot)

Use the following commands to set up the `ogstun` TUN interface manually:

```bash
sudo ip tuntap add name ogstun mode tun
sudo ip addr add 10.45.0.1/16 dev ogstun
sudo ip addr add 2001:db8:cafe::1/48 dev ogstun
sudo ip link set ogstun up
```

⚠️ Note: These settings are **not persistent** and will be lost after a reboot.

### ⚙️ Configuration Change (YAML)

The following changes are made in `etc/open5gs/amf.yaml`. **x** denotes a removal, and **+** denotes an addition.

```diff
# etc/open5gs/amf.yaml:

- x      - address: 127.0.0.5
+ +      - address: 192.168.0.111

- plmn_id:
- x        mcc: 999
- x        mnc: 70
+ +        mcc: 001
+ +        mnc: 01
-       amf_id:

tai:
  - plmn_id:
- x        mcc: 999
- x        mnc: 70
+ +        mcc: 001
+ +        mnc: 01
-       tac: 1

plmn_support:
  - plmn_id:
- x        mcc: 999
- x        mnc: 70
+ +        mcc: 001
+ +        mnc: 01

```

### ⚙️ Configuration Change (YAML)

The following changes are made in `etc/open5gs/nrf.yaml`. **x** denotes a removal, and **+** denotes an addition.

```diff
# etc/open5gs/nrf.yaml:

 plmn_id:
- x        mcc: 999
- x        mnc: 70
+ +        mcc: 001
+ +        mnc: 01
  sbi:

```


### ⚙️ Configuration Change (YAML)

The following changes are made in `etc/open5gs/smf.yaml`. **x** denotes a removal, and **+** denotes an addition.

```diff
# etc/open5gs/smf.yaml:

pfcp:
  server:
- x      - address: 127.0.0.4
+ +      - address: 192.168.0.111

- x        - address: 127.0.0.7
- x  gtpc:
- x    server:
- x      - address: 127.0.0.4
+ +      - address: 192.168.0.112
+ +        dnn: internet

gtpu:
  server:
- x      - address: 127.0.0.4
+ +      - address: 192.168.0.111

session:
  - subnet: 10.45.0.1/16
- x    - subnet: 2001:db8:cafe::1/48
+ +      dnn: internet

dns:
  - 8.8.8.8
  - 8.8.4.4
- x    - 2001:4860:4860::8888
- x    - 2001:4860:4860::8844

#  freeDiameter: /root/open5gs/install/etc/freeDiameter/smf.conf
```

## Changes in configuration files of Open5GS 5GC U-Plane
### ⚙️ Configuration Change (YAML)

The following changes are made in `etc/open5gs/upf.yaml`. **x** denotes a removal, and **+** denotes an addition.

```diff
# etc/open5gs/upf.yaml:

server:
- x      - address: 127.0.0.7
+ +      - address: 192.168.0.112

gtpu:
  server:
- x      - address: 127.0.0.7
+ +      - address: 192.168.0.112

session:
  - subnet: 10.45.0.1/16
- x    - subnet: 2001:db8:cafe::1/48
+ +      dnn: internet
+ +      dev: ogstun
```


### 🔧 Enabling IP Forwarding

To enable IP forwarding, uncomment the following line in `/etc/sysctl.conf`:

```bash
net.ipv4.ip_forward=1
```

Then, apply the changes by running:

```bash
sudo sysctl -p
```

### 🔧 TUN Interface Setup and NAT Configuration

To set up the `ogstun` TUN interface and configure NAT, run the following commands:

```bash
# Create the TUN device
sudo ip tuntap add name ogstun mode tun

# Assign IP address to the TUN interface
sudo ip addr add 10.45.0.1/16 dev ogstun

# Bring the interface up
sudo ip link set ogstun up

# Configure NAT for the subnet
sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
```
## Changes in configuration files of RAN
### ⚙️ Configuration Change (YAML)

The following changes are made in `UERANSIM/config/open5gs-gnb.yaml`. **x** denotes a removal, and **+** denotes an addition.

```diff
# UERANSIM/config/open5gs-gnb.yaml:

 mcc: '999'          # Mobile Country Code value
 mnc: '70'           # Mobile Network Code value (2 or 3 digits)
+ +   mcc: '001'      # Mobile Country Code value
+ +   mnc: '01'       # Mobile Network Code value (2 or 3 digits)

 linkIp: 127.0.0.1   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
 ngapIp: 127.0.0.1   # gNB's local IP address for N2 Interface (Usually same with local IP)
 gtpIp: 127.0.0.1    # gNB's local IP address for N3 Interface (Usually same with local IP)
+ +   linkIp: 192.168.0.131   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
+ +   ngapIp: 192.168.0.131   # gNB's local IP address for N2 Interface (Usually same with local IP)
+ +   gtpIp: 192.168.0.131    # gNB's local IP address for N3 Interface (Usually same with local IP)

# List of AMF address information
amfConfigs:
- x  - address: 127.0.0.5
+ +  - address: 192.168.0.111
```
## Changes in configuration files of UE
### ⚙️ Configuration Change (YAML)

The following changes are made in `UERANSIM/config/open5gs-ue.yaml`. **x** denotes a removal, and **+** denotes an addition.

```diff
# UERANSIM/config/open5gs-ue.yaml:

# IMSI number of the UE. IMSI = [MCC|MNC|MSISDN] (In total 15 digits)
- x   supi: 'imsi-999700000000001'
- +   supi: 'imsi-001010000000000'

# Mobile Country Code value of HPLMN
- x   mcc: '999'
- +   mcc: '001'

# Mobile Network Code value of HPLMN (2 or 3 digits)
- x   mnc: '70'
- +   mnc: '01'

# List of gNB IP addresses for Radio Link Simulation
gnbSearchList:
- x  - 127.0.0.1
- +  - 192.168.0.131
```

# ✅ Validation & Testing

## 🚀 Running the RAN

To start the RAN (gNodeB), run the following command:

```bash
./nr-gnb -c ../config/open5gs-gnb.yaml
```

## 🚀 Running the UE

To start the UE (User Equipment), run the following command with **sudo**:

```bash
sudo ./nr-ue -c ../config/open5gs-ue.yaml
```

# 🛠️ Troubleshooting

| Problem             | Solution                              |
|---------------------|----------------------------------------|
| Node not reachable  | Check IP, power         |

---

## 📚 References

- [Open5GS](https://??)
- [Pi Imager](https://??)
- [UERANSIM](https://??)

---
```