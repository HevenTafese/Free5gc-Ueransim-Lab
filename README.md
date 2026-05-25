# 5G Core Network Simulation on Local VMs
### free5GC + UERANSIM · Ubuntu 22.04 · VirtualBox · End-to-End GTP Tunnel Verified

---

## Why This Exists

Most existing guides for free5GC and UERANSIM begin at the software installation step — they assume you already have a configured Linux environment, correctly networked VMs, and a working understanding of how the components relate to each other. In practice, that assumption breaks most attempts before they start.

This repository documents a complete, working implementation of a 5G Core Network simulation built from scratch: from hypervisor configuration and VM provisioning through to a verified GTP-U tunnel with confirmed end-to-end internet connectivity through the simulated UE. Every configuration error encountered during the process is recorded, with the root cause and resolution explained. This is not a polished ideal-path tutorial — it is an honest account of how the system was built, what broke, and why.

The goal is that someone with no prior free5GC experience can follow this from a blank machine and reach a working state without having to cross-reference six different sources.

---

## Architecture Overview

This simulation implements a split deployment across two virtual machines, mirroring the logical separation between the **5G Core Network (5GC)** and the **Radio Access Network (RAN)** in a real deployment.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        HOST MACHINE (VirtualBox)                     │
│                                                                       │
│   ┌────────────────────────────┐   ┌──────────────────────────────┐  │
│   │        VM 1: free5gc        │   │       VM 2: ueransim          │  │
│   │     Ubuntu 22.04 Desktop    │   │     Ubuntu 22.04 Desktop     │  │
│   │                             │   │                              │  │
│   │  ┌──────────────────────┐   │   │  ┌────────────────────────┐  │  │
│   │  │     free5GC v4.2.2   │   │   │  │    UERANSIM v3.2.6     │  │  │
│   │  │                      │   │   │  │                        │  │  │
│   │  │  NRF  AMF  SMF  UPF  │   │   │  │  nr-gnb  (gNB sim)     │  │  │
│   │  │  UDM  AUSF PCF  CHF  │   │   │  │  nr-ue   (UE sim)      │  │  │
│   │  │  NEF  NSSF           │   │   │  │                        │  │  │
│   │  │                      │   │   │  │  uesimtun0             │  │  │
│   │  │  MongoDB             │   │   │  │  (GTP-U tunnel iface)  │  │  │
│   │  │  WebConsole :5000    │   │   │  └────────────────────────┘  │  │
│   │  └──────────────────────┘   │   │                              │  │
│   │                             │   │                              │  │
│   │  enp0s3 (NAT)  10.0.2.15    │   │  enp0s3 (NAT)  10.0.2.15    │  │
│   │  enp0s8 (HO)  192.168.56.101│   │  enp0s8 (HO) 192.168.56.102 │  │
│   └────────────────────────────┘   └──────────────────────────────┘  │
│                                                                       │
│              Host-Only Network: 192.168.56.0/24                       │
└─────────────────────────────────────────────────────────────────────┘
```

**Network Interfaces:**
| Interface | VM | Purpose |
|---|---|---|
| `enp0s3` (NAT) | Both | Internet access for package installation |
| `enp0s8` (Host-Only) | Both | Inter-VM communication (N2/N3 interfaces) |

**Key 5G Component Mapping:**
| Component | Role | Analogy |
|---|---|---|
| NRF | Network Repository — all NFs register here | Directory service |
| AMF | Access and Mobility Management — handles UE registration | Reception / authentication desk |
| SMF | Session Management — establishes PDU sessions | Session coordinator |
| UPF | User Plane Function — routes actual data traffic | The router to the internet |
| UDM | Unified Data Management — subscriber database | HR records |
| AUSF | Authentication Server — cryptographic validation | Security verification |
| gNB (UERANSIM) | Simulated 5G base station | Cell tower |
| UE (UERANSIM) | Simulated 5G device | Mobile phone |

---

## Prerequisites

**Host machine requirements:**
- VirtualBox 6.1 or later
- Ubuntu 22.04 Desktop ISO (downloaded prior to setup)
- Minimum 8 GB RAM available for VMs (4 GB per VM)
- Minimum 40 GB free disk space

**Software versions used in this implementation:**
- free5GC: v4.2.2
- UERANSIM: v3.2.6 (commit `85a0fbf`)
- Go: 1.21.8
- gtp5g kernel module: v0.9.14
- MongoDB: 8.0 (community edition)
- Node.js: 20.x

---

## Part 1 — VM Provisioning

Both VMs are created with the same base configuration. The critical requirement is **Skip Unattended Installation** during VM creation — unattended mode creates a locked-down user without sudo access which prevents all subsequent steps.

### VM 1: free5GC

**VirtualBox settings:**
- Name: `free5gc`
- ISO: Ubuntu 22.04 Desktop
- ✅ Skip Unattended Installation
- RAM: 4096 MB, CPUs: 2
- Disk: 20 GB

**Network adapters:**
- Adapter 1: NAT (leave default)
- Adapter 2: Host-only Adapter → `VirtualBox Host-Only Ethernet Adapter`

**During Ubuntu installation:**
- Installation type: Minimal
- Username: `vboxuser` (or your choice)
- Hostname: `free5gc`

### VM 2: UERANSIM

Same hardware configuration, with:
- Name: `ueransim`
- Hostname: `ueransim`

### Static IP Configuration

After Ubuntu installation, both VMs need static IPs on the host-only adapter. Adapter `enp0s8` will have no IPv4 address by default.

**On VM 1 (free5GC):**

```bash
sudo nano /etc/netplan/01-network-manager-all.yaml
```

Replace contents with:

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      addresses: [192.168.56.101/24]
```

```bash
sudo chmod 600 /etc/netplan/01-network-manager-all.yaml
sudo netplan apply
```

**On VM 2 (UERANSIM):** Same process, with address `192.168.56.102/24`.

**Verify connectivity:**

```bash
# From UERANSIM VM
ping 192.168.56.101

# From free5GC VM
ping 192.168.56.102
```

Both must respond before proceeding.

---

## Part 2 — free5GC Installation

All commands in this section are run on **VM 1 (free5GC)**.

### System Update

```bash
sudo apt -y update
sudo apt -y install wget git
```

### Verify AVX Support (Required for MongoDB)

```bash
lscpu | grep avx
```

Output must contain `avx`. MongoDB 8.0 requires AVX instruction support. If no output is returned, install MongoDB 4.4 instead.

### MongoDB Installation

```bash
sudo apt install -y gnupg curl

curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] \
  https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.2 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.2.list

sudo apt update
sudo apt install -y mongodb-org

sudo systemctl start mongod
sudo systemctl enable mongod
sudo systemctl status mongod
```

Expected: `Active: active (running)`

### Build Dependencies

```bash
sudo apt -y install git gcc g++ cmake autoconf libtool pkg-config libmnl-dev libyaml-dev
```

### Go Installation

```bash
wget https://dl.google.com/go/go1.21.8.linux-amd64.tar.gz
sudo tar -C /usr/local -zxvf go1.21.8.linux-amd64.tar.gz

mkdir -p ~/go/{bin,pkg,src}
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export GOROOT=/usr/local/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin:$GOROOT/bin' >> ~/.bashrc
echo 'export GO111MODULE=auto' >> ~/.bashrc
source ~/.bashrc
```

Verify:

```bash
go version
# Expected: go version go1.21.8 linux/amd64
```

### gtp5g Kernel Module

The GTP-U kernel module is required for the UPF to handle 5G data plane traffic.

```bash
git clone -b v0.9.14 https://github.com/free5gc/gtp5g.git
cd gtp5g
make
sudo make install
cd ~
```

### Clone and Build free5GC

```bash
git clone --recursive -b v4.2.2 -j `nproc` https://github.com/free5gc/free5gc.git
cd free5gc
make
```

Build time is approximately 5–15 minutes depending on hardware.

### WebConsole

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt update
sudo apt install -y nodejs
sudo corepack enable

cd ~/free5gc
make webconsole
```

---

## Part 3 — free5GC Configuration

The default free5GC configuration binds all interfaces to loopback addresses (`127.0.0.x`). For inter-VM communication over the host-only network, three configuration files must be updated to use `192.168.56.101`.

### AMF Configuration

```bash
nano ~/free5gc/config/amfcfg.yaml
```

Locate and update:

```yaml
ngapIpList:
  - 192.168.56.101   # was 127.0.0.1
```

### SMF Configuration

```bash
nano ~/free5gc/config/smfcfg.yaml
```

Locate and update:

```yaml
interfaces:
  - interfaceType: N3
    endpoints:
      - 192.168.56.101   # was 127.0.0.8
```

### UPF Configuration

```bash
nano ~/free5gc/config/upfcfg.yaml
```

Three fields require updating:

```yaml
pfcp:
  addr: 192.168.56.101     # was 127.0.0.8
  nodeID: 192.168.56.101   # was 127.0.0.8

gtpu:
  ifList:
    - addr: 192.168.56.101  # was 127.0.0.8
      type: N3
```

### Network Rules (IP Forwarding and NAT)

These rules enable the UPF to forward UE traffic to the internet via `enp0s3`. The `ufw` disable is permanent; the iptables rules must be reapplied after each reboot using the provided script.

```bash
sudo systemctl stop ufw
sudo systemctl disable ufw
```

For each session (or after reboot):

```bash
sudo ~/free5gc/reload_host_config.sh enp0s3
```

This script applies:
- `net.ipv4.ip_forward=1`
- MASQUERADE rule on `enp0s3`
- TCP MSS clamping
- FORWARD chain accept rule

---

## Part 4 — Subscriber Provisioning (WebConsole)

Start the WebConsole:

```bash
cd ~/free5gc/webconsole
./bin/webconsole
```

Navigate to `http://192.168.56.101:5000` from the host machine. Login: `admin` / `free5gc`.

**Add subscriber with the following values:**

| Field | Value |
|---|---|
| SUPI | `imsi-208930000000003` |
| MCC | `208` |
| MNC | `93` |
| Authentication Method | `5G_AKA` |
| Operator Code Type | **`OP`** (not OPc — this is the most common misconfiguration) |
| Operator Code | `8e27b6af0e692e750f32667a3b14605d` |
| Key | `8baf473f2f8fd09487cccbd7097c6862` |
| SQN | `000000000000` |

> **Important:** The `Operator Code Type` field defaults to `OPc`. It must be changed to `OP`. free5GC will derive and store the OPc internally. If left as OPc and the pre-computed OPc value does not match what free5GC derives from the OP, authentication will fail with `MAC_FAILURE` — see Troubleshooting section.

Stop the WebConsole after saving: `Ctrl+C`.

---

## Part 5 — UERANSIM Installation

All commands in this section are run on **VM 2 (UERANSIM)**.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git make g++ libsctp-dev lksctp-tools iproute2
sudo snap install cmake --classic

cd ~
git clone https://github.com/aligungr/UERANSIM
cd UERANSIM
git checkout 85a0fbf
make
```

### gNB Configuration

```bash
nano ~/UERANSIM/config/free5gc-gnb.yaml
```

Update:

```yaml
ngapIp: 192.168.56.102   # this VM's IP
gtpIp: 192.168.56.102    # this VM's IP

amfConfigs:
  - address: 192.168.56.101   # free5GC VM IP
```

### UE Configuration

```bash
nano ~/UERANSIM/config/free5gc-ue.yaml
```

Verify these values match the WebConsole subscriber exactly:

```yaml
supi: 'imsi-208930000000003'
mcc: '208'
mnc: '93'
key: '8baf473f2f8fd09487cccbd7097c6862'
op: '8e27b6af0e692e750f32667a3b14605d'
opType: 'OPC'   # use the derived OPc value from MongoDB
```

> **Note on opType:** After creating the subscriber via WebConsole, retrieve the derived OPc value from MongoDB and use that in the UERANSIM config with `opType: 'OPC'`:
> ```bash
> mongosh free5gc --eval "db['subscriptionData.authenticationData.authenticationSubscription'].find().pretty()"
> ```
> Use the `encOpcKey` value returned.

---

## Part 6 — Running the Simulation

### Terminal layout required: 1 on free5GC VM, 3 on UERANSIM VM

**free5GC VM — Terminal 1:**

```bash
sudo ~/free5gc/reload_host_config.sh enp0s3
cd ~/free5gc
./run.sh
```

Wait for output to stabilise (all NF registration complete, UPF heartbeat appearing).

**UERANSIM VM — Terminal 1 (gNB):**

```bash
cd ~/UERANSIM
build/nr-gnb -c config/free5gc-gnb.yaml
```

Expected output:
```
[sctp] [info] SCTP connection established (192.168.56.101:38412)
[ngap] [info] NG Setup procedure is successful
```

> Screenshot: `img/gnb-connected.png`

**UERANSIM VM — Terminal 2 (UE):**

```bash
cd ~/UERANSIM
sudo build/nr-ue -c config/free5gc-ue.yaml
```

Expected output:
```
[nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[nas] [info] Initial Registration is successful
[nas] [info] PDU Session establishment is successful PSI[1]
[app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```

> Screenshot: `img/ue-registered.png`

**UERANSIM VM — Terminal 3 (Connectivity Test):**

```bash
ping -I uesimtun0 google.com
```

Successful replies confirm end-to-end GTP-U tunnel operation: UE → gNB → AMF/SMF/UPF → internet.

> Screenshot: `img/ping-success.png`

---

## Troubleshooting

These are real errors encountered during this implementation, documented with their root causes.

### SCTP Connection Refused on gNB Start

```
[sctp] [error] Connecting to 192.168.56.101:38412 failed. SCTP could not connect: Connection refused
```

**Cause:** AMF is not listening on the host-only IP. Either `amfcfg.yaml` still has `127.0.0.1` in `ngapIpList`, or free5GC is not running.

**Resolution:** Verify `amfcfg.yaml` shows `192.168.56.101` under `ngapIpList`. If free5GC stopped, check for stale processes:

```bash
sudo killall -9 amf smf upf pcf udm udr ausf nssf nef chf nrf
sudo ~/free5gc/reload_host_config.sh enp0s3
cd ~/free5gc && ./run.sh
```

---

### AMF Fails to Start: `cannot assign requested address`

```
[ERRO][AMF][Ngap] Failed to listen: cannot assign requested address
```

**Cause:** `ngapIpList` in `amfcfg.yaml` contains an IP that does not exist on this machine — for example the UERANSIM IP `192.168.56.102` instead of `192.168.56.101`.

**Resolution:** Open `amfcfg.yaml` and confirm `ngapIpList` contains the free5GC VM's IP, not the UERANSIM VM's IP.

---

### SMF Cannot Reach UPF

```
[WARN][SMF][Main] Failed to setup an association with UPF[[127.0.0.8]]
```

**Cause:** `smfcfg.yaml` still references the default `127.0.0.8` loopback address for the UPF endpoint.

**Resolution:** Open `smfcfg.yaml` and update all `127.0.0.8` references under `interfaces.N3.endpoints` to `192.168.56.101`. Use `Ctrl+W` in nano to search.

---

### Authentication Failure: MAC Mismatch

```
[nas] [error] AUTN validation MAC mismatch
[nas] [error] Authentication Reject received
```

**Cause (most common):** `Operator Code Type` was left as `OPc` in the WebConsole when creating the subscriber, but UERANSIM's `free5gc-ue.yaml` is configured with the raw `OP` value. The cryptographic derivation does not match.

**Resolution:** Retrieve the derived OPc value that free5GC stored internally:

```bash
mongosh free5gc --eval "db['subscriptionData.authenticationData.authenticationSubscription'].find().pretty()"
```

Use the `encOpcKey` value in `free5gc-ue.yaml`:

```yaml
op: '<encOpcKey value from MongoDB>'
opType: 'OPC'
```

**Cause (secondary):** SQN desynchronisation from previous failed attempts. In the WebConsole, edit the subscriber and reset the SQN field to `000000000000`.

---

### Port Already in Use on Restart

```
[ERRO][CHF][SBI] SBI server error: listen tcp 127.0.0.113:8000: bind: address already in use
```

**Cause:** Previous free5GC processes did not terminate cleanly and are still holding ports.

**Resolution:**

```bash
sudo killall -9 amf smf upf pcf udm udr ausf nssf nef chf nrf webconsole
sudo fuser -k 8000/tcp
```

---

### uesimtun0 Not Appearing After UE Registration

If the UE shows `MM-REGISTERED` but no `uesimtun0` interface appears, check that the UPF `ifList` address in `upfcfg.yaml` is `192.168.56.101` and that the gtp5g kernel module loaded correctly:

```bash
lsmod | grep gtp5g
```

If not loaded:

```bash
cd ~/gtp5g && sudo make install
```

---

## Repository Structure

```
.
├── README.md
├── img/
│   ├── architecture-diagram.png
│   ├── gnb-connected.png        # NG Setup successful
│   ├── ue-registered.png        # PDU session established, uesimtun0 up
│   └── ping-success.png         # ping -I uesimtun0 google.com
└── config/
    ├── amfcfg.yaml              # Modified AMF config
    ├── smfcfg.yaml              # Modified SMF config
    ├── upfcfg.yaml              # Modified UPF config
    ├── free5gc-gnb.yaml         # Modified gNB config
    └── free5gc-ue.yaml          # Modified UE config
```

> The `config/` folder contains only the modified versions of each file with inline comments marking every change from the default. These are provided as reference — do not copy them directly without reading through the configuration sections above.

---

## References

- [free5GC Official Documentation](https://free5gc.org/guide/)
- [free5GC GitHub Repository](https://github.com/free5gc/free5gc)
- [UERANSIM GitHub Repository](https://github.com/aligungr/UERANSIM)
- [gtp5g Kernel Module](https://github.com/free5gc/gtp5g)
- 3GPP TS 23.501 — System architecture for the 5G System
- 3GPP TS 33.501 — Security architecture and procedures for 5G System

---

## Acknowledgements

Built against free5GC v4.2.2 and UERANSIM v3.2.6 on Ubuntu 22.04 LTS. Verified on VirtualBox 7.x running on a Windows host.

This repository was created because the existing documentation, while technically accurate, does not cover the complete path from bare VM to working simulation in a single coherent guide. The troubleshooting section documents real failures from this implementation rather than theoretical edge cases.
