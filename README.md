
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
All modidications can also be found here: `open5gs/patches/`

### 5.1 Core (Open5GS)
### Patch: amf.patch
```diff
diff --git a/./amf.yaml.orig b/./amf.yaml
index 286074d..15459e2 100644
--- a/./amf.yaml.orig
+++ b/./amf.yaml
@@ -11,36 +11,36 @@ global:
 amf:
   sbi:
     server:
-      - address: 127.0.0.5
-        port: 7777
+      - address: 192.168.0.111
+        port: 7744
     client:
-#      nrf:
-#        - uri: http://127.0.0.10:7777
+      nrf:
+        - uri: http://192.168.0.111:7777
       scp:
-        - uri: http://127.0.0.200:7777
+        - uri: http://192.168.0.111:7777
   ngap:
     server:
-      - address: 127.0.0.5
+      - address: 192.168.0.111
   metrics:
     server:
       - address: 127.0.0.5
         port: 9090
   guami:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       amf_id:
         region: 2
         set: 1
   tai:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       tac: 1
   plmn_support:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       s_nssai:
         - sst: 1
   security:
```

### Patch: ausf.patch
```diff
diff --git a/./ausf.yaml.orig b/./ausf.yaml
index 3a3fea7..451e713 100644
--- a/./ausf.yaml.orig
+++ b/./ausf.yaml
@@ -11,13 +11,13 @@ global:
 ausf:
   sbi:
     server:
-      - address: 127.0.0.11
+      - address: 192.168.0.111
         port: 7777
     client:
-#      nrf:
-#        - uri: http://127.0.0.10:7777
+      nrf:
+        - uri: http://192.168.0.111:7777
       scp:
-        - uri: http://127.0.0.200:7777
+        - uri: http://192.168.0.111:7777
 
 ################################################################################
 # SBI Server
```

### Patch: nrf.patch
```diff
diff --git a/./nrf.yaml.orig b/./nrf.yaml
index b97e927..995808a 100644
--- a/./nrf.yaml.orig
+++ b/./nrf.yaml
@@ -11,11 +11,11 @@ global:
 nrf:
   serving:  # 5G roaming requires PLMN in NRF
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
   sbi:
     server:
-      - address: 127.0.0.10
+      - address: 192.168.0.111
         port: 7777
 
 ################################################################################
```

### Patch: pcf.patch
```diff
diff --git a/./pcf.yaml.orig b/./pcf.yaml
index dadbdc9..2624ceb 100644
--- a/./pcf.yaml.orig
+++ b/./pcf.yaml
@@ -12,13 +12,13 @@ global:
 pcf:
   sbi:
     server:
-      - address: 127.0.0.13
+      - address: 192.168.0.111
         port: 7777
     client:
-#      nrf:
-#        - uri: http://127.0.0.10:7777
+      nrf:
+        - uri: http://192.168.0.111:7777
       scp:
-        - uri: http://127.0.0.200:7777
+        - uri: http://192.168.0.111:7777
   metrics:
     server:
       - address: 127.0.0.13
```

### Patch: smf.patch
```diff
diff --git a/./smf.yaml.orig b/./smf.yaml
index d6b952a..c78f97e 100644
--- a/./smf.yaml.orig
+++ b/./smf.yaml
@@ -11,25 +11,25 @@ global:
 smf:
   sbi:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.0.111
         port: 7777
     client:
-#      nrf:
-#        - uri: http://127.0.0.10:7777
+      nrf:
+        - uri: http://192.168.0.111:7777
       scp:
-        - uri: http://127.0.0.200:7777
+        - uri: http://192.168.0.111:7777
   pfcp:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.0.111
     client:
       upf:
-        - address: 127.0.0.7
+        - address: 192.168.0.112
   gtpc:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.0.111
   gtpu:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.0.111
   metrics:
     server:
       - address: 127.0.0.4
@@ -39,6 +39,8 @@ smf:
       gateway: 10.45.0.1
     - subnet: 2001:db8:cafe::/48
       gateway: 2001:db8:cafe::1
+      dnn: internet
+      dev: ogstun
   dns:
     - 8.8.8.8
     - 8.8.4.4
```

### Patch: udm.patch
```diff
diff --git a/./udm.yaml.orig b/./udm.yaml
index 6d5fa4f..6656c40 100644
--- a/./udm.yaml.orig
+++ b/./udm.yaml
@@ -30,13 +30,13 @@ udm:
       key: /home/cplane/Downloads/open5gs/install/etc/open5gs/hnet/secp256r1-6.key
   sbi:
     server:
-      - address: 127.0.0.12
+      - address: 192.168.0.111
         port: 7777
     client:
-#      nrf:
-#        - uri: http://127.0.0.10:7777
+      nrf:
+        - uri: http://192.168.0.111:7777
       scp:
-        - uri: http://127.0.0.200:7777
+        - uri: http://192.168.0.111:7777
 
 ################################################################################
 # Home Network Public Key
```

### Patch: udr.patch
```diff
diff --git a/./udr.yaml.orig b/./udr.yaml
index a9e1dcc..cdba85f 100644
--- a/./udr.yaml.orig
+++ b/./udr.yaml
@@ -12,13 +12,13 @@ global:
 udr:
   sbi:
     server:
-      - address: 127.0.0.20
+      - address: 192.168.0.111
         port: 7777
     client:
-#      nrf:
-#        - uri: http://127.0.0.10:7777
+      nrf:
+        - uri: http://192.168.0.111:7777
       scp:
-        - uri: http://127.0.0.200:7777
+        - uri: http://192.168.0.111:7777
 
 ################################################################################
 # SBI Server
```

### Patch: upf.patch
```diff
diff --git a/./upf.yaml.orig b/./upf.yaml
index d02ee0c..b59b801 100644
--- a/./upf.yaml.orig
+++ b/./upf.yaml
@@ -11,18 +11,20 @@ global:
 upf:
   pfcp:
     server:
-      - address: 127.0.0.7
+      - address: 192.168.0.112
     client:
-#      smf:     #  UPF PFCP Client try to associate SMF PFCP Server
-#        - address: 127.0.0.4
+      smf:     #  UPF PFCP Client try to associate SMF PFCP Server
+        - address: 192.168.0.111
   gtpu:
     server:
-      - address: 127.0.0.7
+      - address: 192.168.0.112
   session:
     - subnet: 10.45.0.0/16
       gateway: 10.45.0.1
     - subnet: 2001:db8:cafe::/48
       gateway: 2001:db8:cafe::1
+      dnn: internet
+      dev: ogstun
   metrics:
     server:
       - address: 127.0.0.7
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


