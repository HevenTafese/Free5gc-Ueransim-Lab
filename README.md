# 5G Core Network Simulation on Local VMs

free5GC v4.2.2 + UERANSIM v3.2.6 · Ubuntu 22.04 · VirtualBox · End-to-End GTP Tunnel Verified

---

## Why This Exists

Most existing guides for free5GC and UERANSIM begin at the software installation step. They assume you already have a configured Linux environment, correctly networked virtual machines, and a working understanding of how the components relate to each other. In practice, that assumption breaks most attempts before they start.

This repository documents a complete working implementation built from scratch: from hypervisor configuration and VM provisioning through to a verified GTP-U tunnel with confirmed end-to-end internet connectivity through the simulated UE. Every configuration error encountered during the process is recorded with the root cause and resolution explained. This is not a polished ideal-path tutorial. It is an honest account of how the system was built, what broke, and why.

The goal is that someone with no prior free5GC experience can follow this from a blank machine and reach a working state without cross-referencing six different sources.

---

## End Result

These are the actual outputs from this implementation.

**gNB connected to free5GC core:**

![gNB Connected](img/gnb-connected.png)

**UE registered and GTP tunnel up:**

![UE Registered](img/ue-registered.png)

**End-to-end ping through the simulated 5G network:**

![Ping Success](img/ping-success.png)

---

## Architecture

The simulation runs across two virtual machines, mirroring the logical separation between the 5G Core Network and the Radio Access Network in a real deployment.

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

| Interface | VM | Purpose |
|---|---|---|
| `enp0s3` NAT | Both | Internet access for package installation |
| `enp0s8` Host-Only | Both | Inter-VM communication over N2 and N3 interfaces |

Each 5G Network Function has a specific role. The table below maps what each one does in the context of this simulation.

| Component | Role | In Plain Terms |
|---|---|---|
| NRF | Network Repository Function | Every other NF registers here on startup. Acts as a directory. |
| AMF | Access and Mobility Management | Handles UE registration and authentication. The front door. |
| SMF | Session Management Function | Establishes and manages PDU sessions. Coordinates data paths. |
| UPF | User Plane Function | Routes actual user traffic. The component that touches the internet. |
| UDM | Unified Data Management | Subscriber database. Stores credentials and profiles. |
| AUSF | Authentication Server Function | Handles cryptographic authentication challenges. |
| gNB | Simulated base station (UERANSIM) | The cell tower in this simulation. |
| UE | Simulated device (UERANSIM) | The mobile phone in this simulation. |

---

## Prerequisites

Before starting, the following must be in place on the host machine.

| Requirement | Version |
|---|---|
| VirtualBox | 6.1 or later |
| Ubuntu 22.04 Desktop ISO | Downloaded before setup |
| RAM available for VMs | 8 GB minimum (4 GB per VM) |
| Free disk space | 40 GB minimum |

Software versions used in this implementation:

| Software | Version |
|---|---|
| free5GC | v4.2.2 |
| UERANSIM | v3.2.6 (commit `85a0fbf`) |
| Go | 1.21.8 |
| gtp5g kernel module | v0.9.14 |
| MongoDB | 8.0 community edition |
| Node.js | 20.x |

---

## Part 1: VM Provisioning

Both VMs are created with the same base configuration. The critical requirement during VM creation is ticking **Skip Unattended Installation**. Unattended mode creates a locked-down user account without sudo access, which blocks every subsequent step.

**VM 1 settings in VirtualBox:**

| Setting | Value |
|---|---|
| Name | `free5gc` |
| ISO | Ubuntu 22.04 Desktop |
| Skip Unattended Installation | Yes |
| RAM | 4096 MB |
| CPUs | 2 |
| Disk | 20 GB |

Network adapters: Adapter 1 as NAT (leave default), Adapter 2 as Host-only Adapter pointing to VirtualBox Host-Only Ethernet Adapter.

During Ubuntu installation set the hostname to `free5gc`, choose Minimal Installation, and set a username and password you will remember.

**VM 2** follows the same settings with name `ueransim` and hostname `ueransim`.

### Static IP Configuration

After Ubuntu installation both VMs need static IPs on the host-only adapter. The `enp0s8` interface will have no IPv4 address by default.

On VM 1 (free5GC):

```bash
sudo nano /etc/netplan/01-network-manager-all.yaml
```

Replace the contents with:

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

On VM 2 (UERANSIM) follow the same process with address `192.168.56.102/24`.

Verify both VMs can reach each other before proceeding:

```bash
# From UERANSIM VM
ping 192.168.56.101

# From free5GC VM
ping 192.168.56.102
```

Both must respond before moving on.

---

## Part 2: free5GC Installation

All commands in this section run on VM 1 (free5GC).

### System Update

```bash
sudo apt -y update
sudo apt -y install wget git
```

### Verify AVX Support

MongoDB 8.0 requires AVX instruction support. Run this before installing it:

```bash
lscpu | grep avx
```

The output must contain `avx`. If nothing is returned, install MongoDB 4.4 instead.

### MongoDB

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

Expected output contains `Active: active (running)`.

### Build Dependencies

```bash
sudo apt -y install git gcc g++ cmake autoconf libtool pkg-config libmnl-dev libyaml-dev
```

### Go

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

The GTP-U kernel module is what allows the UPF to handle 5G data plane traffic. Without it the tunnel interface will appear to come up but no traffic will pass through it.

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

Build time is approximately 5 to 15 minutes depending on hardware. Do not interrupt it.

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

## Part 3: free5GC Configuration

The default free5GC configuration binds all interfaces to loopback addresses (`127.0.0.x`). Three configuration files must be updated to use `192.168.56.101` for inter-VM communication to work.

### AMF

```bash
nano ~/free5gc/config/amfcfg.yaml
```

```yaml
ngapIpList:
  - 192.168.56.101   # was 127.0.0.1
```

### SMF

```bash
nano ~/free5gc/config/smfcfg.yaml
```

```yaml
interfaces:
  - interfaceType: N3
    endpoints:
      - 192.168.56.101   # was 127.0.0.8
```

### UPF

```bash
nano ~/free5gc/config/upfcfg.yaml
```

Three fields require updating in this file:

```yaml
pfcp:
  addr: 192.168.56.101     # was 127.0.0.8
  nodeID: 192.168.56.101   # was 127.0.0.8

gtpu:
  ifList:
    - addr: 192.168.56.101  # was 127.0.0.8
      type: N3
```

### Network Rules

The following script enables IP forwarding and sets up NAT so the UPF can forward UE traffic out to the internet via `enp0s3`. The `ufw` disable is permanent across reboots. The iptables rules are not and must be reapplied each session.

```bash
sudo systemctl stop ufw
sudo systemctl disable ufw
```

Run this before starting free5GC in every session:

```bash
sudo ~/free5gc/reload_host_config.sh enp0s3
```

---

## Part 4: Subscriber Provisioning

Start the WebConsole:

```bash
cd ~/free5gc/webconsole
./bin/webconsole
```

Navigate to `http://192.168.56.101:5000` from the host machine and log in with `admin` / `free5gc`.

Go to Subscribers and create a new subscriber with these exact values:

| Field | Value |
|---|---|
| SUPI | `imsi-208930000000003` |
| MCC | `208` |
| MNC | `93` |
| Authentication Method | `5G_AKA` |
| Operator Code Type | `OP` (not OPc) |
| Operator Code | `8e27b6af0e692e750f32667a3b14605d` |
| Key | `8baf473f2f8fd09487cccbd7097c6862` |
| SQN | `000000000000` |

The Operator Code Type field defaults to OPc. It must be changed to OP before submitting. free5GC will derive and store the OPc value internally. If this is left as OPc and the pre-computed value does not match what free5GC derives from the OP, authentication will fail with MAC_FAILURE. See the troubleshooting section for how to recover from this.

Stop the WebConsole after saving with `Ctrl+C`.

After creating the subscriber, retrieve the internally derived OPc value from MongoDB. You will need this for the UERANSIM configuration:

```bash
mongosh free5gc --eval "db['subscriptionData.authenticationData.authenticationSubscription'].find().pretty()"
```

Note the `encOpcKey` value from the output.

---

## Part 5: UERANSIM Installation

All commands in this section run on VM 2 (UERANSIM).

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

```yaml
ngapIp: 192.168.56.102
gtpIp: 192.168.56.102

amfConfigs:
  - address: 192.168.56.101
```

### UE Configuration

```bash
nano ~/UERANSIM/config/free5gc-ue.yaml
```

```yaml
supi: 'imsi-208930000000003'
mcc: '208'
mnc: '93'
key: '8baf473f2f8fd09487cccbd7097c6862'
op: '<encOpcKey value from MongoDB>'
opType: 'OPC'
```

Use the `encOpcKey` value retrieved from MongoDB in Part 4.

---

## Part 6: Running the Simulation

Four terminals are needed in total. One on the free5GC VM and three on the UERANSIM VM.

**free5GC VM:**

```bash
sudo ~/free5gc/reload_host_config.sh enp0s3
cd ~/free5gc
./run.sh
```

Wait for the output to stabilise. The UPF heartbeat lines appearing every 10 seconds indicate everything is up.

**UERANSIM VM, Terminal 1 (gNB):**

```bash
cd ~/UERANSIM
build/nr-gnb -c config/free5gc-gnb.yaml
```

Wait for `NG Setup procedure is successful` before starting the UE.

**UERANSIM VM, Terminal 2 (UE):**

```bash
cd ~/UERANSIM
sudo build/nr-ue -c config/free5gc-ue.yaml
```

Wait for `uesimtun0 is up`.

**UERANSIM VM, Terminal 3 (route and test):**

```bash
sudo ip route add default dev uesimtun0 metric 1
ping google.com
```

Successful replies confirm end-to-end GTP-U tunnel operation from simulated UE through the 5G Core to the internet.

---

## Session Startup Reference

After rebooting the free5GC VM, run this before anything else:

```bash
sudo ~/free5gc/reload_host_config.sh enp0s3
```

After the UE registers and `uesimtun0` is up, run this on the UERANSIM VM before testing:

```bash
sudo ip route add default dev uesimtun0 metric 1
```

These two steps are required every session. Everything else persists across reboots.

---

## Troubleshooting

All errors below were encountered during this implementation and are documented with their actual root causes.

### SCTP Connection Refused on gNB Start

```
[sctp] [error] Connecting to 192.168.56.101:38412 failed. SCTP could not connect: Connection refused
```

The AMF is not listening on the host-only IP. Either `amfcfg.yaml` still contains `127.0.0.1` in `ngapIpList`, or free5GC is not running. If free5GC stopped unexpectedly, clear stale processes and restart:

```bash
sudo killall -9 amf smf upf pcf udm udr ausf nssf nef chf nrf
sudo ~/free5gc/reload_host_config.sh enp0s3
cd ~/free5gc && ./run.sh
```

### AMF Fails to Start

```
[ERRO][AMF][Ngap] Failed to listen: cannot assign requested address
```

The `ngapIpList` in `amfcfg.yaml` contains an IP that does not exist on this machine. This happens when the UERANSIM IP `192.168.56.102` is mistakenly entered instead of `192.168.56.101`.

### SMF Cannot Reach UPF

```
[WARN][SMF][Main] Failed to setup an association with UPF[[127.0.0.8]]
```

The `smfcfg.yaml` file still has the default `127.0.0.8` loopback address. Open the file and use `Ctrl+W` in nano to search for all occurrences and replace them with `192.168.56.101`.

### Authentication Failure: MAC Mismatch

```
[nas] [error] AUTN validation MAC mismatch
[nas] [error] Authentication Reject received
```

The most common cause is that Operator Code Type was left as OPc in the WebConsole but UERANSIM is configured with the raw OP value. Retrieve the derived OPc from MongoDB and use that in `free5gc-ue.yaml` with `opType: 'OPC'`:

```bash
mongosh free5gc --eval "db['subscriptionData.authenticationData.authenticationSubscription'].find().pretty()"
```

A secondary cause is SQN desynchronisation from previous failed attempts. Open the subscriber in the WebConsole and reset the SQN field to `000000000000`.

### Port Already in Use on Restart

```
[ERRO][CHF][SBI] SBI server error: listen tcp 127.0.0.113:8000: bind: address already in use
```

Previous free5GC processes did not terminate cleanly. Kill them and free the port:

```bash
sudo killall -9 amf smf upf pcf udm udr ausf nssf nef chf nrf webconsole
sudo fuser -k 8000/tcp
```

### Ping Through uesimtun0 Returns 100% Packet Loss

The tunnel interface is up but traffic is not flowing end-to-end. This is almost always caused by the gtp5g kernel module being in a bad state. The fix is to reload it cleanly, then restart free5GC:

```bash
sudo killall -9 amf smf upf pcf udm udr ausf nssf nef chf nrf
sudo rmmod gtp5g
cd ~/gtp5g && sudo make install && sudo modprobe gtp5g
sudo ~/free5gc/reload_host_config.sh enp0s3
cd ~/free5gc && ./run.sh
```

After restarting, add the route on the UERANSIM VM once the UE registers:

```bash
sudo ip route add default dev uesimtun0 metric 1
```

---

## Repository Structure

```
.
├── README.md
├── img/
│   ├── gnb-connected.png
│   ├── ue-registered.png
│   └── ping-success.png
└── config/
    ├── amfcfg.yaml
    ├── smfcfg.yaml
    ├── upfcfg.yaml
    ├── free5gc-gnb.yaml
    └── free5gc-ue.yaml
```

The `config` folder contains only the modified versions of each file with inline comments on every changed line. They are provided as reference. Do not copy them directly without reading through the configuration sections above, as IP addresses may differ depending on your setup.

---

## References

- [free5GC Official Documentation](https://free5gc.org/guide/)
- [free5GC GitHub Repository](https://github.com/free5gc/free5gc)
- [UERANSIM GitHub Repository](https://github.com/aligungr/UERANSIM)
- [gtp5g Kernel Module](https://github.com/free5gc/gtp5g)
- 3GPP TS 23.501: System architecture for the 5G System
- 3GPP TS 33.501: Security architecture and procedures for 5G System

---

Built against free5GC v4.2.2 and UERANSIM v3.2.6 on Ubuntu 22.04 LTS. Verified on VirtualBox 7.x running on a Windows host. The troubleshooting section documents real failures from this implementation rather than theoretical edge cases.
