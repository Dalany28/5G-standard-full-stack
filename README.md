
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

## 2. Install MongoDB

```bash
sudo apt update
sudo apt install -y gnupg
```
Now import the MongoDB public keys
```bash
curl -fsSL https://pgp.mongodb.com/server-8.0.asc \
  | sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-8.0.gpg
```
Add the MongoDB APT Repository (Ubuntu 22.04)
Create the repository file:
```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.0 multiverse" \
  | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
```

Install MongoDB pacakges
```bash
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod 
```

## 3. Setting Up the TUN Interface (ogstun)
Open5gs uses a TUN interface to route UE traffic through UPF

### 3.1 Create the `ogstun` Interface:
```bash
sudo ip tuntap add name ogstun mode tun
sudo ip addr add 10.45.0.1/16 dev ogstun
sudo ip addr add 2001:db8:cafe::1/48 dev ogstun
sudo ip link set ogstun up
```
⚠️ Note: This configuration is _temporary_ — it disappears after a reboot unless scripted.

💡Optional: Use the Auto-Config Script
If you cloned the Open5GS repo, there is a helper script that sets up the TUN device for you:
```bash
sudo ./misc/netconf.sh
```

### 3.2 Enabling IP forwarding:
```bash
sudo nano /etc/sysctl.conf
# Uncomment or add the line:
net.ipv4.ip_forward=1
sudo sysctl -p
```
### 3.3 Configure NAT:
```bash
sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
```
💡 **Tip**: Verify interface status using `ip addr show ogstun`. It should be **UP** once UE attaches.
## 4. Build Open5gs
Before building Open5GS, install the development packages and tools required to compile the project:
1. Install Build Dependencies:

```bash
sudo apt install -y \
  python3-pip python3-setuptools python3-wheel \
  ninja-build build-essential flex bison git cmake \
  libsctp-dev libgnutls28-dev libgcrypt-dev libssl-dev \
  libmongoc-dev libbson-dev libyaml-dev libmicrohttpd-dev \
  libcurl4-gnutls-dev libnghttp2-dev libtins-dev libtalloc-dev \
  meson

```
2. Install libidn (Version Depends on Your System):
 -  Different Ubuntu systems may require either **libidn-dev** or **libidn11-dev**.  
This script checks which one is available and installs it:

```bash
if apt-cache show libidn-dev > /dev/null 2>&1; then
    sudo apt-get install -y --no-install-recommends libidn-dev
else
    sudo apt-get install -y --no-install-recommends libidn11-dev
fi
```

3. git clone:

```bash
git clone https://github.com/open5gs/open5gs
```

- Build Using Meson & Ninja:
```bash
 cd open5gs
 meson build --prefix=`pwd`/install
 ninja -C build
```
4. Run the test program (optional):
Before running them, ensure your system or VM has enough memory (at least **2–4GB RAM** recommended).

```bash
./build/tests/attach/attach ## EPC Only
./build/tests/registration/registration ## 5G Core Only
```
Run all test as below
```bash
cd build
meson test -v
```
Once everything builds correctly, install the compiled binaries:
```bash
cd build
sudo ninja install
cd ../
```


## 5. 📝 YAML Configurations
All configuration files can be found here: `install/etc/open5gs/`.
These files define IP addresses, PLMN IDs, TAC values, and session parameters used by the 5G Core.

### 5.1 Core (Open5GS)
- `core/nrf.yaml`
```yaml
logger:
  file:
    path: /home/cplane/open5gs/install/var/log/open5gs/nrf.log
#  level: info   # fatal|error|warn|info(default)|debug|trace

global:
  max:
    ue: 1024  # The number of UE can be increased depending on memory size.
#    peer: 64

nrf:
  serving:  # 5G roaming requires PLMN in NRF
    - plmn_id:
        mcc: 001
        mnc: 01
  sbi:
    server:
      - address: 192.168.0.111
        port: 7777

```

- `core/udm.yaml`
```yaml
logger:
  file:
    path: /home/cplane/open5gs/install/var/log/open5gs/udm.log
#  level: info   # fatal|error|warn|info(default)|debug|trace

global:
  max:
    ue: 1024  # The number of UE can be increased depending on memory size.
#    peer: 64

udm:
  hnet:
    - id: 1
      scheme: 1
      key: /home/cplane/open5gs/install/etc/open5gs/hnet/curve25519-1.key
    - id: 2
      scheme: 2
      key: /home/cplane/open5gs/install/etc/open5gs/hnet/secp256r1-2.key
    - id: 3
      scheme: 1
      key: /home/cplane/open5gs/install/etc/open5gs/hnet/curve25519-3.key
    - id: 4
      scheme: 2
      key: /home/cplane/open5gs/install/etc/open5gs/hnet/secp256r1-4.key
    - id: 5
      scheme: 1
      key: /home/cplane/open5gs/install/etc/open5gs/hnet/curve25519-5.key
    - id: 6
      scheme: 2
      key: /home/cplane/open5gs/install/etc/open5gs/hnet/secp256r1-6.key
  sbi:
    server:
      - address: 192.168.0.111
        port: 7755
    client:
      nrf:
        - uri: http://192.168.0.111:7777
      scp:
        - uri: http://192.168.0.111:7777

```
- `core/ausf.yaml`
```yaml
logger:
  file:
    path: /home/cplane/open5gs/install/var/log/open5gs/ausf.log
#  level: info   # fatal|error|warn|info(default)|debug|trace

global:
  max:
    ue: 1024  # The number of UE can be increased depending on memory size.
#    peer: 64

ausf:
  sbi:
    server:
      - address: 192.168.0.111
        port: 6666
    client:
      nrf:
        - uri: http://192.168.0.111:7777
      scp:
        - uri: http://192.168.0.111:7777

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
    
    
- `core/pcf.yaml`
```yaml
db_uri: mongodb://localhost/open5gs
logger:
  file:
    path: /home/cplane/open5gs/install/var/log/open5gs/pcf.log
#  level: info   # fatal|error|warn|info(default)|debug|trace

global:
  max:
    ue: 1024  # The number of UE can be increased depending on memory size.
#    peer: 64

pcf:
  sbi:
    server:
      - address: 192.168.0.111
        port: 7722
    client:
      nrf:
        - uri: http://192.168.0.111:7777
      scp:
        - uri: http://192.168.0.111:7777
  metrics:
    server:
      - address: 127.0.0.13
        port: 9090
```
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

- `core/nssf.yaml`
```yaml
  GNU nano 8.4                                                                       nssf.yaml                                                                                
logger:
  file:
    path: /home/cplane/open5gs/install/var/log/open5gs/nssf.log
#  level: info   # fatal|error|warn|info(default)|debug|trace

global:
  max:
    ue: 1024  # The number of UE can be increased depending on memory size.
#    peer: 64

nssf:
  sbi:
    server:
      - address: 192.168.0.111
        port: 6677
    client:
      nrf:
        - uri: http://192.168.0.111:7777
      scp:
        - uri: http://192.168.0.111:7777
      nsi:
        - uri: http://127.0.0.10:7777
          s_nssai:
            sst: 1

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

Building and Running Open5GS WebUI 
Install Node.js (required for WebUI):
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg

NODE_MAJOR=20
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" \
  | sudo tee /etc/apt/sources.list.d/nodesource.list

sudo apt update
sudo apt install -y nodejs

```
Install WebUI Dependencies:
```bash
cd webui
npm ci
```
Run the WebUI:
```bash
npm run dev
```
Access WebUI & Add subscriber:
Open:  
**[http://localhost:9999](http://localhost:9999/)**

**Login:**

-   Username:  **admin**
    
-   Password:  **1423**
    

**Add a subscriber:**

1.  Open  **Subscriber**  menu
    
2.  Click  **+**
    
3.  Enter  **IMSI**,  **K/OPc/AMF**,  **APN**
    
4.  Click  **Save**

### 5.2 Build UERANSIM From Source
Clone the UERANSIM project:
```bash
git clone https://github.com/aligungr/UERANSIM
cd UERANSIM
```
Install Required Dependencies:
```bash
sudo apt install -y make gcc g++ libsctp-dev lksctp-tools iproute2
sudo snap install cmake --classic
```
⚠️ **Do NOT install cmake using apt** — Ubuntu provides an outdated version.

Build UERANSIM:
```bash
cd ~/UERANSIM
make
```
Build Output:
After compiling, binaries appear in:
```bash
~/UERANSIM/build
```
And that's it. After successfully compiling the project, output binaries will be copied to `~/UERANSIM/build` folder. And you should see the following files:
|File|Descriptions  |
|--|--|
| nr-gnb | Main executable for 5G gNB (RAN) |
|nr-ue|5G UE executable|
|nr-cli|CLI tool for UE/gNB control|
|nr-binder|Tool to bind UE Internet to host apps|





**UERANSIM/config/open5gs-gnb.yaml**
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
    


**UERANSIM/config/open5gs-ue.yaml**
```yaml
                                                                            
supi: 'imsi-001010000000000'

mcc: '001'
mnc: '01'

gnbSearchList:
  - 192.168.0.131

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









## 7. Troubleshooting Checklist

| Problem                  | Solution                                                                 |
|--------------------------|--------------------------------------------------------------------------|
| UE cannot connect to gNB | Check IP addresses in YAML files, ensure gNB and AMF IP match host network |




## 8. 📚 References

- [Open5GS Official Documentation](https://open5gs.org/)
- [UERANSIM GitHub Repository](https://github.com/aligungr/UERANSIM)
- [Docker Documentation](https://docs.docker.com/)
