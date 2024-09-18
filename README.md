# free5GC 5GC & UERANSIM UE / RAN Sample Configuration - ULCL with one I-UPF and two PSA-UPFs
This describes a very simple configuration that uses free5GC and UERANSIM for ULCL with one I-UPF and two PSA-UPFs.  

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Overview of free5GC 5GC Simulation Mobile Network](#overview)
- [Changes in configuration files of free5GC 5GC and UERANSIM UE / RAN](#changes)
  - [Changes in configuration files of free5GC 5GC C-Plane](#changes_cp)
  - [Changes in configuration files of free5GC 5GC U-Plane (I-UPF)](#changes_up1)
  - [Changes in configuration files of free5GC 5GC U-Plane (PSA-UPF1)](#changes_up2)
  - [Changes in configuration files of free5GC 5GC U-Plane (PSA-UPF2)](#changes_up3)
  - [Changes in configuration files of UERANSIM UE / RAN](#changes_ueransim)
    - [Changes in configuration files of RAN (gNodeB)](#changes_ran)
    - [Changes in configuration files of UE](#changes_ue)
- [Network settings of free5GC 5GC and UERANSIM UE / RAN](#network_settings)
  - [Network settings of free5GC 5GC U-Plane (I-UPF)](#network_settings_up1)
  - [Network settings of free5GC 5GC U-Plane (PSA-UPF1)](#network_settings_up2)
  - [Network settings of free5GC 5GC U-Plane (PSA-UPF2)](#network_settings_up3)
- [Build free5GC and UERANSIM](#build)
  - [Build go-upf](#build_upf)
- [Run free5GC 5GC and UERANSIM UE / RAN](#run)
  - [Run free5GC 5GC U-Plane (I-UPF & PSA-UPFs)](#run_up)
  - [Run free5GC 5GC C-Plane](#run_cp)
  - [Run UERANSIM (gNodeB)](#run_ran)
  - [Run UERANSIM (UE)](#run_ue)
    - [Start UE connected to gNodeB](#con_ue)
    - [Ping google.com going through PSA-UPF1](#ping_google)
    - [Ping 8.8.8.8 going through I-UPF](#ping_8)
    - [Ping 172.17.0.1 going through PSA-UPF1](#ping_docker1)
    - [Ping 172.18.0.1 going through PSA-UPF2](#ping_docker2)
- [Changelog (summary)](#changelog)

---
<a id="overview"></a>

## Overview of free5GC 5GC Simulation Mobile Network

The following minimum configuration was set as a condition.
- I-UPF selects the communication paths according to the destination host and network.
- I-UPF connects with multiple PSA-UPFs and each PSA-UPF has the routing information to a specific network.

The built simulation environment is as follows.
**In this configuration, each PSA-UPF must be reachable to Docker network on local.**

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The 5GC / UE / RAN used are as follows.
- 5GC - free5GC v3.4.3 (2024.09.12) - https://github.com/free5gc/free5gc
- UPF - go-upf v1.2.3 (2024.05.11) - https://github.com/free5gc/go-upf
- (UPF) - gtp5g v0.8.10 (2024.06.03) - https://github.com/free5gc/gtp5g
- UE / RAN - UERANSIM v3.2.6 (2024.08.27) - https://github.com/aligungr/UERANSIM

Each VMs are as follows.  
| VM # | SW & Role | IP address | OS | Mem (Min) | HDD (Min) |
| --- | --- | --- | --- | --- | --- |
| VM1 | free5GC  5GC C-Plane | 192.168.0.141/24 | Ubuntu 24.04 | 2GB | 20GB |
| VM2 | free5GC  5GC U-Plane (I-UPF) | 192.168.0.142/24 | Ubuntu 24.04 | 1GB | 10GB |
| VM3 | free5GC  5GC U-Plane (PSA-UPF1) | 192.168.0.143/24 | Ubuntu 24.04 | 1GB | 10GB |
| VM4 | free5GC  5GC U-Plane (PSA-UPF2) | 192.168.0.144/24 | Ubuntu 24.04 | 1GB | 10GB |
| VM5 | UERANSIM RAN (gNodeB) | 192.168.0.131/24 | Ubuntu 24.04 | 1GB | 10GB |
| VM6 | UERANSIM UE | 192.168.0.132/24 | Ubuntu 24.04 | 1GB | 10GB |

Subscriber Information (other information is default) is as follows.  
**Note. Please select OP or OPc according to the setting of UERANSIM UE configuration files.**
| UE | IMSI | DNN | OP/OPc |
| --- | --- | --- | --- |
| UE | 001010000000000 | internet | OPc |

I registered these information with the free5GC WebUI.
In addition, [3GPP TS 35.208](https://www.3gpp.org/DynaReport/35208.htm) "4.3 Test Sets" is published by 3GPP as test data for the 3GPP authentication and key generation functions (MILENAGE).

DN is as follows.
| DN | TUNnel interface of DN | DNN | TUNnel interface of UE |
| --- | --- | --- | --- |
| 10.60.0.0/16 | upfgtp | internet | uesimtun0 |

The UE routing topology is as follows.
```
                    --- PSA-UPF1
                    |
UE --- gNodeB --- I-UPF
                    |
                    --- PSA-UPF2
```
The communication paths to be confirmed for each destination IP address are as follows.
| Destination IP address | Communication path |
| --- | --- |
| google.com | I-UPF --> PSA-UPF1 --> Internet |
| 8.8.8.8 | I-UPF --> Internet |
| 172.17.0.1 | I-UPF --> PSA-UPF1 --> 172.17.0.0/16 (Docker network on local) |
| 172.18.0.1 | I-UPF --> PSA-UPF2 --> 172.18.0.0/16 (Docker network on local) |

<a id="changes"></a>

## Changes in configuration files of free5GC 5GC and UERANSIM UE / RAN

Please refer to the following for building free5GC and UERANSIM respectively.
- free5GC v3.4.3 (2024.09.12) - https://free5gc.org/guide/
- go-upf v1.2.3 (2024.05.11) - https://free5gc.org/guide/
- gtp5g v0.8.10 (2024.06.03) - https://free5gc.org/guide/
- UERANSIM v3.2.6 (2024.08.27) - https://github.com/aligungr/UERANSIM/wiki/Installation

<a id="changes_cp"></a>

### Changes in configuration files of free5GC 5GC C-Plane

- `free5gc/config/amfcfg.yaml`
```diff
--- amfcfg.yaml.orig    2024-08-31 18:40:42.425497926 +0900
+++ amfcfg.yaml 2024-08-31 18:57:16.692270705 +0900
@@ -5,7 +5,7 @@
 configuration:
   amfName: AMF # the name of this AMF
   ngapIpList:  # the IP list of N2 interfaces on this AMF
-    - 127.0.0.18
+    - 192.168.0.141
   ngapPort: 38412 # the SCTP port listened by NGAP
   sbi: # Service-based interface information
     scheme: http # the protocol for sbi (http or https)
@@ -24,18 +24,18 @@
   servedGuamiList: # Guami (Globally Unique AMF ID) list supported by this AMF
     # <GUAMI> = <MCC><MNC><AMF ID>
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       amfId: cafe00 # AMF identifier (3 bytes hex string, range: 000000~FFFFFF)
   supportTaiList:  # the TAI (Tracking Area Identifier) list supported by this AMF
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       tac: 000001 # Tracking Area Code (3 bytes hex string, range: 000000~FFFFFF)
   plmnSupportList: # the PLMNs (Public land mobile network) list supported by this AMF
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       snssaiList: # the S-NSSAI (Single Network Slice Selection Assistance Information) list supported by this AMF
         - sst: 1 # Slice/Service Type (uinteger, range: 0~255)
           sd: 010203 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
```
- `free5gc/config/ausfcfg.yaml`
```diff
--- ausfcfg.yaml.orig   2024-08-31 18:40:42.425497926 +0900
+++ ausfcfg.yaml        2024-08-31 18:52:04.727832529 +0900
@@ -16,10 +16,8 @@
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
   nrfCertPem: cert/nrf.pem # NRF Certificate
   plmnSupportList: # the PLMNs (Public Land Mobile Network) list supported by this AUSF
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
-    - mcc: 123 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 45  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   groupId: ausfGroup001 # ID for the group of the AUSF
   eapAkaSupiImsiPrefix: false # including "imsi-" prefix or not when using the SUPI to do EAP-AKA' authentication
 
```
- `free5gc/config/nrfcfg.yaml`
```diff
--- nrfcfg.yaml.orig    2024-08-31 18:40:42.425497926 +0900
+++ nrfcfg.yaml 2024-08-31 18:53:00.723935459 +0900
@@ -18,8 +18,8 @@
       key: cert/root.key
     oauth: true
   DefaultPlmnId:
-    mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-    mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+    mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   serviceNameList: # the SBI services provided by this NRF, refer to TS 29.510
     - nnrf-nfm # Nnrf_NFManagement service
     - nnrf-disc # Nnrf_NFDiscovery service
```
- `free5gc/config/nssfcfg.yaml`
```diff
--- nssfcfg.yaml.orig   2024-08-31 18:40:42.425497926 +0900
+++ nssfcfg.yaml        2024-08-31 18:58:49.940161382 +0900
@@ -18,12 +18,12 @@
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
   nrfCertPem: cert/nrf.pem # NRF Certificate
   supportedPlmnList: # the PLMNs (Public land mobile network) list supported by this NSSF
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   supportedNssaiInPlmnList: # Supported S-NSSAI List for each PLMN
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       supportedSnssaiList: # Supported S-NSSAIs of the PLMN
         - sst: 1 # Slice/Service Type (uinteger, range: 0~255)
           sd: 010203 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
```
- `free5gc/config/smfcfg.yaml`
```diff
--- smfcfg.yaml.orig    2024-09-15 18:09:13.892166385 +0900
+++ smfcfg.yaml 2024-09-15 18:11:35.656659221 +0900
@@ -34,14 +34,14 @@
             ipv4: 8.8.8.8
             ipv6: 2001:4860:4860::8888
   plmnList: # the list of PLMN IDs that this SMF belongs to (optional, remove this key when unnecessary)
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   locality: area1 # Name of the location where a set of AMF, SMF, PCF and UPFs are located
   pfcp: # the IP address of N4 interface on this SMF (PFCP)
     # addr config is deprecated in smf config v1.0.3, please use the following config
-    nodeID: 127.0.0.1 # the Node ID of this SMF
-    listenAddr: 127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
-    externalAddr: 127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
+    nodeID: 192.168.0.141 # the Node ID of this SMF
+    listenAddr: 192.168.0.141 # the IP/FQDN of N4 interface on this SMF (PFCP)
+    externalAddr: 192.168.0.141 # the IP/FQDN of N4 interface on this SMF (PFCP)
     assocFailAlertInterval: 10s
     assocFailRetryInterval: 30s
     heartbeatInterval: 10s
@@ -49,10 +49,31 @@
     upNodes: # information of userplane node (AN or UPF)
       gNB1: # the name of the node
         type: AN # the type of the node (AN or UPF)
-      UPF: # the name of the node
+      I-UPF: # the name of the node
         type: UPF # the type of the node (AN or UPF)
-        nodeID: 127.0.0.8 # the Node ID of this UPF
-        addr: 127.0.0.8 # the IP/FQDN of N4 interface on this UPF (PFCP)
+        nodeID: 192.168.0.142 # the Node ID of this UPF
+        addr: 192.168.0.142 # the IP/FQDN of N4 interface on this UPF (PFCP)
+        sNssaiUpfInfos: # S-NSSAI information list for this UPF
+          - sNssai: # S-NSSAI (Single Network Slice Selection Assistance Information)
+              sst: 1 # Slice/Service Type (uinteger, range: 0~255)
+              sd: 010203 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
+            dnnUpfInfoList: # DNN information list for this S-NSSAI
+              - dnn: internet
+        interfaces: # Interface list for this UPF
+          - interfaceType: N3 # the type of the interface (N3 or N9)
+            endpoints: # the IP address of this N3/N9 interface on this UPF
+              - 192.168.0.142
+            networkInstances: # Data Network Name (DNN)
+              - internet
+          - interfaceType: N9 # the type of the interface (N3 or N9)
+            endpoints: # the IP address of this N3/N9 interface on this UPF
+              - 192.168.0.142
+            networkInstances: # Data Network Name (DNN)
+              - internet
+      PSA-UPF1: # the name of the node
+        type: UPF # the type of the node (AN or UPF)
+        nodeID: 192.168.0.143 # the Node ID of this UPF
+        addr: 192.168.0.143 # the IP/FQDN of N4 interface on this UPF (PFCP)
         sNssaiUpfInfos: # S-NSSAI information list for this UPF
           - sNssai: # S-NSSAI (Single Network Slice Selection Assistance Information)
               sst: 1 # Slice/Service Type (uinteger, range: 0~255)
@@ -63,24 +84,36 @@
                   - cidr: 10.60.0.0/16
                 staticPools:
                   - cidr: 10.60.100.0/24
+        interfaces: # Interface list for this UPF
+          - interfaceType: N9 # the type of the interface (N3 or N9)
+            endpoints: # the IP address of this N3/N9 interface on this UPF
+              - 192.168.0.143
+            networkInstances: # Data Network Name (DNN)
+              - internet
+      PSA-UPF2: # the name of the node
+        type: UPF # the type of the node (AN or UPF)
+        nodeID: 192.168.0.144 # the Node ID of this UPF
+        addr: 192.168.0.144 # the IP/FQDN of N4 interface on this UPF (PFCP)
+        sNssaiUpfInfos: # S-NSSAI information list for this UPF
           - sNssai: # S-NSSAI (Single Network Slice Selection Assistance Information)
               sst: 1 # Slice/Service Type (uinteger, range: 0~255)
-              sd: 112233 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
+              sd: 010203 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
             dnnUpfInfoList: # DNN information list for this S-NSSAI
               - dnn: internet
-                pools:
-                  - cidr: 10.61.0.0/16
-                staticPools:
-                  - cidr: 10.61.100.0/24
         interfaces: # Interface list for this UPF
-          - interfaceType: N3 # the type of the interface (N3 or N9)
+          - interfaceType: N9 # the type of the interface (N3 or N9)
             endpoints: # the IP address of this N3/N9 interface on this UPF
-              - 127.0.0.8
+              - 192.168.0.144
             networkInstances: # Data Network Name (DNN)
               - internet
     links: # the topology graph of userplane, A and B represent the two nodes of each link
       - A: gNB1
-        B: UPF
+        B: I-UPF
+      - A: I-UPF
+        B: PSA-UPF1
+      - A: I-UPF
+        B: PSA-UPF2
+  ulcl: true
   # retransmission timer for pdu session modification command
   t3591:
     enable: true # true or false
```
- `free5gc/config/uerouting.yaml`
```yaml
info:
  version: 1.0.7
  description: Routing information for UE

ueRoutingInfo: # the list of UE routing information
  UE1: # Group Name
    members:
    - imsi-001010000000000 # Subscription Permanent Identifier of the UE
    topology: # Network topology for this group (Uplink: A->B, Downlink: B->A)
    # default path derived from this topology
    # node name should be consistent with smfcfg.yaml
      - A: gNB1
        B: I-UPF
      - A: I-UPF
        B: PSA-UPF1
    specificPath:
      - dest: 172.17.0.0/16 # the destination IP address on Data Network (DN)
        # the order of UPF nodes in this path. We use the UPF's name to represent each UPF node.
        # The UPF's name should be consistent with smfcfg.yaml
        path: [I-UPF, PSA-UPF1]
      - dest: 172.18.0.0/16
        path: [I-UPF, PSA-UPF2]
      - dest: 8.8.8.8/32
        path: [I-UPF]
```

<a id="changes_up1"></a>

### Changes in configuration files of free5GC 5GC U-Plane (I-UPF)

- `go-upf/upfcfg.yaml`
```diff
--- upfcfg.yaml.orig    2024-08-31 19:23:52.419250932 +0900
+++ upfcfg.yaml 2024-09-15 18:28:12.128539749 +0900
@@ -3,8 +3,8 @@
 
 # The listen IP and nodeID of the N4 interface on this UPF (Can't set to 0.0.0.0)
 pfcp:
-  addr: 127.0.0.8   # IP addr for listening
-  nodeID: 127.0.0.8 # External IP or FQDN can be reached
+  addr: 192.168.0.142   # IP addr for listening
+  nodeID: 192.168.0.142 # External IP or FQDN can be reached
   retransTimeout: 1s # retransmission timeout
   maxRetrans: 3 # the max number of retransmission
 
@@ -13,18 +13,18 @@
   # The IP list of the N3/N9 interfaces on this UPF
   # If there are multiple connection, set addr to 0.0.0.0 or list all the addresses
   ifList:
-    - addr: 127.0.0.8
+    - addr: 192.168.0.142
       type: N3
       # name: upf.5gc.nctu.me
       # ifname: gtpif
       # mtu: 1400
+    - addr: 192.168.0.142
+      type: N9
 
 # The DNN list supported by UPF
 dnnList:
   - dnn: internet # Data Network Name
     cidr: 10.60.0.0/16 # Classless Inter-Domain Routing for assigned IPv4 pool of UE
-  - dnn: internet # Data Network Name
-    cidr: 10.61.0.0/16 # Classless Inter-Domain Routing for assigned IPv4 pool of UE
     # natifname: eth0
 
 logger: # log output setting
```

<a id="changes_up2"></a>

### Changes in configuration files of free5GC 5GC U-Plane (PSA-UPF1)

- `go-upf/upfcfg.yaml`
```diff
--- upfcfg.yaml.orig    2024-08-31 19:23:52.419250932 +0900
+++ upfcfg.yaml 2024-09-15 18:32:10.552002281 +0900
@@ -3,8 +3,8 @@
 
 # The listen IP and nodeID of the N4 interface on this UPF (Can't set to 0.0.0.0)
 pfcp:
-  addr: 127.0.0.8   # IP addr for listening
-  nodeID: 127.0.0.8 # External IP or FQDN can be reached
+  addr: 192.168.0.143   # IP addr for listening
+  nodeID: 192.168.0.143 # External IP or FQDN can be reached
   retransTimeout: 1s # retransmission timeout
   maxRetrans: 3 # the max number of retransmission
 
@@ -13,8 +13,8 @@
   # The IP list of the N3/N9 interfaces on this UPF
   # If there are multiple connection, set addr to 0.0.0.0 or list all the addresses
   ifList:
-    - addr: 127.0.0.8
-      type: N3
+    - addr: 192.168.0.143
+      type: N9
       # name: upf.5gc.nctu.me
       # ifname: gtpif
       # mtu: 1400
@@ -23,8 +23,6 @@
 dnnList:
   - dnn: internet # Data Network Name
     cidr: 10.60.0.0/16 # Classless Inter-Domain Routing for assigned IPv4 pool of UE
-  - dnn: internet # Data Network Name
-    cidr: 10.61.0.0/16 # Classless Inter-Domain Routing for assigned IPv4 pool of UE
     # natifname: eth0
 
 logger: # log output setting
```

<a id="changes_up3"></a>

### Changes in configuration files of free5GC 5GC U-Plane (PSA-UPF2)

- `go-upf/upfcfg.yaml`
```diff
--- upfcfg.yaml.orig    2024-08-31 19:23:52.419250932 +0900
+++ upfcfg.yaml 2024-09-15 18:35:11.789220274 +0900
@@ -3,8 +3,8 @@
 
 # The listen IP and nodeID of the N4 interface on this UPF (Can't set to 0.0.0.0)
 pfcp:
-  addr: 127.0.0.8   # IP addr for listening
-  nodeID: 127.0.0.8 # External IP or FQDN can be reached
+  addr: 192.168.0.144   # IP addr for listening
+  nodeID: 192.168.0.144 # External IP or FQDN can be reached
   retransTimeout: 1s # retransmission timeout
   maxRetrans: 3 # the max number of retransmission
 
@@ -13,8 +13,8 @@
   # The IP list of the N3/N9 interfaces on this UPF
   # If there are multiple connection, set addr to 0.0.0.0 or list all the addresses
   ifList:
-    - addr: 127.0.0.8
-      type: N3
+    - addr: 192.168.0.144
+      type: N9
       # name: upf.5gc.nctu.me
       # ifname: gtpif
       # mtu: 1400
@@ -23,8 +23,6 @@
 dnnList:
   - dnn: internet # Data Network Name
     cidr: 10.60.0.0/16 # Classless Inter-Domain Routing for assigned IPv4 pool of UE
-  - dnn: internet # Data Network Name
-    cidr: 10.61.0.0/16 # Classless Inter-Domain Routing for assigned IPv4 pool of UE
     # natifname: eth0
 
 logger: # log output setting
```

<a id="changes_ueransim"></a>

### Changes in configuration files of UERANSIM UE / RAN

<a id="changes_ran"></a>

#### Changes in configuration files of RAN (gNodeB)

- `UERANSIM/config/free5gc-gnb.yaml`
```diff
--- free5gc-gnb.yaml.orig       2024-05-12 01:59:00.000000000 +0900
+++ free5gc-gnb.yaml    2024-09-15 18:42:02.193131540 +0900
@@ -1,17 +1,17 @@
-mcc: '208'          # Mobile Country Code value
-mnc: '93'           # Mobile Network Code value (2 or 3 digits)
+mcc: '001'          # Mobile Country Code value
+mnc: '01'           # Mobile Network Code value (2 or 3 digits)
 
 nci: '0x000000010'  # NR Cell Identity (36-bit)
 idLength: 32        # NR gNB ID length in bits [22...32]
 tac: 1              # Tracking Area Code
 
-linkIp: 127.0.0.1   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
-ngapIp: 127.0.0.1   # gNB's local IP address for N2 Interface (Usually same with local IP)
-gtpIp: 127.0.0.1    # gNB's local IP address for N3 Interface (Usually same with local IP)
+linkIp: 192.168.0.131   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
+ngapIp: 192.168.0.131   # gNB's local IP address for N2 Interface (Usually same with local IP)
+gtpIp: 192.168.0.131    # gNB's local IP address for N3 Interface (Usually same with local IP)
 
 # List of AMF address information
 amfConfigs:
-  - address: 127.0.0.1
+  - address: 192.168.0.141
     port: 38412
 
 # List of supported S-NSSAIs by this gNB
```

<a id="changes_ue"></a>

#### Changes in configuration files of UE

- `UERANSIM/config/free5gc-ue.yaml`
```diff
--- free5gc-ue.yaml.orig        2024-05-12 01:59:00.000000000 +0900
+++ free5gc-ue.yaml     2024-08-31 20:59:23.069355772 +0900
@@ -1,9 +1,9 @@
 # IMSI number of the UE. IMSI = [MCC|MNC|MSISDN] (In total 15 digits)
-supi: 'imsi-208930000000001'
+supi: 'imsi-001010000000000'
 # Mobile Country Code value of HPLMN
-mcc: '208'
+mcc: '001'
 # Mobile Network Code value of HPLMN (2 or 3 digits)
-mnc: '93'
+mnc: '01'
 # SUCI Protection Scheme : 0 for Null-scheme, 1 for Profile A and 2 for Profile B
 protectionScheme: 0
 # Home Network Public Key for protecting with SUCI Profile A
@@ -28,7 +28,7 @@
 
 # List of gNB IP addresses for Radio Link Simulation
 gnbSearchList:
-  - 127.0.0.1
+  - 192.168.0.131
 
 # UAC Access Identities Configuration
 uacAic:
```

<a id="network_settings"></a>

## Network settings of free5GC 5GC and UERANSIM UE / RAN

<a id="network_settings_up1"></a>

### Network settings of free5GC 5GC U-Plane (I-UPF)

First, uncomment the next line in the `/etc/sysctl.conf` file and reflect it in the OS.
```
net.ipv4.ip_forward=1
```
```
# sysctl -p
```
Next, configure NAPT.
```
# iptables -t nat -A POSTROUTING -s 10.60.0.0/16 ! -o upfgtp -j MASQUERADE
```

<a id="network_settings_up2"></a>

### Network settings of free5GC 5GC U-Plane (PSA-UPF1)

First, uncomment the next line in the `/etc/sysctl.conf` file and reflect it in the OS.
```
net.ipv4.ip_forward=1
```
```
# sysctl -p
```
Next, configure NAPT.
```
# iptables -t nat -A POSTROUTING -s 10.60.0.0/16 ! -o upfgtp -j MASQUERADE
```

<a id="network_settings_up3"></a>

### Network settings of free5GC 5GC U-Plane (PSA-UPF2)

First, uncomment the next line in the `/etc/sysctl.conf` file and reflect it in the OS.
```
net.ipv4.ip_forward=1
```
```
# sysctl -p
```
Next, configure NAPT.
```
# iptables -t nat -A POSTROUTING -s 10.60.0.0/16 ! -o upfgtp -j MASQUERADE
```

<a id="build"></a>

## Build free5GC and UERANSIM

Please refer to the following for building free5GC and UERANSIM respectively.
- free5GC v3.4.3 (2024.09.12) - https://free5gc.org/guide/
- go-upf v1.2.3 (2024.05.11) - https://free5gc.org/guide/
- gtp5g v0.8.10 (2024.06.03) - https://free5gc.org/guide/
- UERANSIM v3.2.6 (2024.08.27) - https://github.com/aligungr/UERANSIM/wiki/Installation

Install MongoDB on free5GC 5GC C-Plane machine.
It is not necessary to install MongoDB on free5GC 5GC U-Plane machines.
[MongoDB Compass](https://www.mongodb.com/products/compass) is a convenient tool to look at the MongoDB database.

**Note. The installation guide also includes instructions on building the latest committed version.
If it doesn't work properly and you suspect some bug, try updating all NFs by following this update procedure.**

<a id="build_upf"></a>

### Build go-upf

For UPF, select the go-upf version that enables ULCL with the free5GC version selected this time.
Please refer to the free5GC guide at the above URL for building free5GC UPF. Below show only the differences in the procedure.
```
# git clone -b v1.2.3 https://github.com/free5gc/go-upf.git
# cd go-upf
# CGO_ENABLED=0 go build -o upf cmd/main.go
# ls upf
upf
# wget https://raw.githubusercontent.com/free5gc/free5gc/main/config/upfcfg.yaml
```

<a id="run"></a>

## Run free5GC 5GC and UERANSIM UE / RAN

First run the 5GC, then UERANSIM (UE & RAN implementation).

<a id="run_up"></a>

### Run free5GC 5GC U-Plane (I-UPF & PSA-UPFs)

- free5GC 5GC U-Plane (I-UPF)
```
# cd go-upf
# ./upf -c upfcfg.yaml
```
- free5GC 5GC U-Plane (PSA-UPF1)
```
# cd go-upf
# ./upf -c upfcfg.yaml
```
- free5GC 5GC U-Plane (PSA-UPF2)
```
# cd go-upf
# ./upf -c upfcfg.yaml
```
Then run `tcpdump` on one more terminals for U-Plane.
- Run `tcpdump` on VM2 (U-Plane (I-UPF))
```
# tcpdump -i upfgtp -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on upfgtp, link-type RAW (Raw IP), snapshot length 262144 bytes
```
- Run `tcpdump` on VM3 (U-Plane (PSA-UPF1))
```
# tcpdump -i upfgtp -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on upfgtp, link-type RAW (Raw IP), snapshot length 262144 bytes
```
- Run `tcpdump` on VM4 (U-Plane (PSA-UPF2))
```
# tcpdump -i upfgtp -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on upfgtp, link-type RAW (Raw IP), snapshot length 262144 bytes
```

<a id="run_cp"></a>

### Run free5GC 5GC C-Plane

Next, run free5GC 5GC C-Plane.

- free5GC 5GC C-Plane

Create the following shell script and run it.
```bash
#!/usr/bin/env bash

PID_LIST=()

NF_LIST="amf udr pcf udm nssf ausf chf"

export GIN_MODE=release

./bin/nrf &
PID_LIST+=($!)
sleep 1

./bin/smf -c config/smfcfg.yaml -u config/uerouting.yaml &
PID_LIST+=($!)
sleep 1

for NF in ${NF_LIST}; do
    ./bin/${NF} &
    PID_LIST+=($!)
    sleep 1
done

function terminate()
{
    sudo kill -SIGTERM ${PID_LIST[${#PID_LIST[@]}-2]} ${PID_LIST[${#PID_LIST[@]}-1]}
    sleep 2
}

trap terminate SIGINT
wait ${PID_LIST}
```

<a id="run_ran"></a>

### Run UERANSIM (gNodeB)

Please refer to the following for usage of UERANSIM.

https://github.com/aligungr/UERANSIM/wiki/Usage

```
# ./nr-gnb -c ../config/free5gc-gnb.yaml
UERANSIM v3.2.6
[2024-09-18 20:55:41.327] [sctp] [info] Trying to establish SCTP connection... (192.168.0.141:38412)
[2024-09-18 20:55:41.339] [sctp] [info] SCTP connection established (192.168.0.141:38412)
[2024-09-18 20:55:41.340] [sctp] [debug] SCTP association setup ascId[5]
[2024-09-18 20:55:41.341] [ngap] [debug] Sending NG Setup Request
[2024-09-18 20:55:41.355] [ngap] [debug] NG Setup Response received
[2024-09-18 20:55:41.355] [ngap] [info] NG Setup procedure is successful
```
The free5GC C-Plane log when executed is as follows.
```
2024-09-18T20:55:41.355630264+09:00 [INFO][AMF][Ngap] [AMF] SCTP Accept from: 192.168.0.131:44965
2024-09-18T20:55:41.359400796+09:00 [INFO][AMF][Ngap] Create a new NG connection for: 192.168.0.131:44965
2024-09-18T20:55:41.363609909+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44965] Handle NGSetupRequest
2024-09-18T20:55:41.366138656+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44965] Send NG-Setup response
```

<a id="run_ue"></a>

### Run UERANSIM (UE)

Ping the following four destination IP addresses and confirm that they are routed through different UPFs.

- google.com - routing from PSA-UPF1 to Internet
- 8.8.8.8 - routing from I-UPF to Internet
- 172.17.0.1 - routing from PSA-UPF1 to 172.17.0.0/16 (Docker network on local)
- 172.18.0.1 - routing from PSA-UPF2 to 172.18.0.0/16 (Docker network on local)

**Note. For example, 172.17.0.0/16 is docker's default network.**

<a id="con_ue"></a>

#### Start UE connected to gNodeB

```
# ./nr-ue -c ../config/free5gc-ue.yaml
UERANSIM v3.2.6
[2024-09-18 20:56:38.261] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2024-09-18 20:56:38.263] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2024-09-18 20:56:38.265] [nas] [info] Selected plmn[001/01]
[2024-09-18 20:56:38.265] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2024-09-18 20:56:38.266] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2024-09-18 20:56:38.266] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2024-09-18 20:56:38.266] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2024-09-18 20:56:38.267] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2024-09-18 20:56:38.268] [nas] [debug] Sending Initial Registration
[2024-09-18 20:56:38.270] [rrc] [debug] Sending RRC Setup Request
[2024-09-18 20:56:38.270] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2024-09-18 20:56:38.271] [rrc] [info] RRC connection established
[2024-09-18 20:56:38.272] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2024-09-18 20:56:38.272] [nas] [info] UE switches to state [CM-CONNECTED]
[2024-09-18 20:56:38.396] [nas] [debug] Authentication Request received
[2024-09-18 20:56:38.396] [nas] [debug] Received SQN [000000000043]
[2024-09-18 20:56:38.396] [nas] [debug] SQN-MS [000000000000]
[2024-09-18 20:56:38.424] [nas] [debug] Security Mode Command received
[2024-09-18 20:56:38.424] [nas] [debug] Selected integrity[2] ciphering[0]
[2024-09-18 20:56:38.578] [nas] [debug] Registration accept received
[2024-09-18 20:56:38.578] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2024-09-18 20:56:38.578] [nas] [debug] Sending Registration Complete
[2024-09-18 20:56:38.578] [nas] [info] Initial Registration is successful
[2024-09-18 20:56:38.578] [nas] [debug] Sending PDU Session Establishment Request
[2024-09-18 20:56:38.578] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2024-09-18 20:56:38.784] [nas] [debug] Configuration Update Command received
[2024-09-18 20:56:38.931] [nas] [debug] PDU Session Establishment Accept received
[2024-09-18 20:56:38.931] [nas] [info] PDU Session establishment is successful PSI[1]
[2024-09-18 20:56:38.953] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
The free5GC C-Plane log when executed is as follows.
```
2024-09-18T20:56:38.318232042+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44965] Handle InitialUEMessage
2024-09-18T20:56:38.318509857+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44965] New RanUe [RanUeNgapID:1][AmfUeNgapID:1]
2024-09-18T20:56:38.318646562+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44965] 5GSMobileIdentity ["SUCI":"suci-0-001-01-0000-0-0-0000000000", err: <nil>]
2024-09-18T20:56:38.322030904+09:00 [INFO][AMF][CTX] New AmfUe [supi:][guti:00101cafe0000000001]
2024-09-18T20:56:38.323385875+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Deregistered] to [Deregistered]
2024-09-18T20:56:38.324392528+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Registration Request
2024-09-18T20:56:38.325006704+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] RegistrationType: Initial Registration
2024-09-18T20:56:38.325705667+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] MobileIdentity5GS: SUCI[suci-0-001-01-0000-0-0-0000000000]
2024-09-18T20:56:38.325783276+09:00 [INFO][AMF][Gmm] Handle event[Start Authentication], transition from [Deregistered] to [Authentication]
2024-09-18T20:56:38.325831104+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Authentication procedure
2024-09-18T20:56:38.328862719+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.334454182+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.345443042+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.348910655+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.353126580+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=AUSF |
2024-09-18T20:56:38.357573029+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.363218792+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.372037227+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.378623109+09:00 [INFO][AUSF][UeAuth] HandleUeAuthPostRequest
2024-09-18T20:56:38.378732882+09:00 [INFO][AUSF][UeAuth] Serving network authorized
2024-09-18T20:56:38.381513610+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.384839895+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.394295738+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.396470027+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.398985280+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AUSF&service-names=nudm-ueau&target-nf-type=UDM |
2024-09-18T20:56:38.399922844+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.401687260+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.406555782+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.408400922+09:00 [INFO][UDM][UEAU] Handle GenerateAuthDataRequest
2024-09-18T20:56:38.409794471+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.412066117+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.417184930+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.419234438+09:00 [INFO][UDM][Suci] suciPart: [suci 0 001 01 0000 0 0 0000000000]
2024-09-18T20:56:38.419311595+09:00 [INFO][UDM][Suci] scheme 0
2024-09-18T20:56:38.419519750+09:00 [INFO][UDM][Suci] SUPI type is IMSI
http://127.0.0.10:8000
2024-09-18T20:56:38.419907459+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.421487525+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.424305540+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.426938636+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.427869300+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=UDM&target-nf-type=UDR |
2024-09-18T20:56:38.431946738+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2024-09-18T20:56:38.433035794+09:00 [INFO][UDM][UEAU] Nil Op
2024-09-18T20:56:38.435442627+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PATCH   | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2024-09-18T20:56:38.436268163+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | POST    | /nudm-ueau/v1/suci-0-001-01-0000-0-0-0000000000/security-information/generate-auth-data |
2024-09-18T20:56:38.436845995+09:00 [INFO][AUSF][UeAuth] Add SuciSupiPair (suci-0-001-01-0000-0-0-0000000000, imsi-001010000000000) to map.
2024-09-18T20:56:38.436876734+09:00 [INFO][AUSF][UeAuth] Use 5G AKA auth method
2024-09-18T20:56:38.436893415+09:00 [INFO][AUSF][5gAka] XresStar = 6138303965316566383562353437373238336436393061633261303634613235
2024-09-18T20:56:38.436995065+09:00 [INFO][AUSF][GIN] | 201 |       127.0.0.1 | POST    | /nausf-auth/v1/ue-authentications |
2024-09-18T20:56:38.437716609+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Send Authentication Request
2024-09-18T20:56:38.437987576+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44965] Send Downlink Nas Transport
2024-09-18T20:56:38.438791158+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Start T3560 timer
2024-09-18T20:56:38.440108874+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44965] Handle UplinkNASTransport
2024-09-18T20:56:38.440426180+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44965] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2024-09-18T20:56:38.440718604+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Authentication] to [Authentication]
2024-09-18T20:56:38.440904537+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Authentication Response
2024-09-18T20:56:38.440921438+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Stop T3560 timer
2024-09-18T20:56:38.441717817+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.443315511+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.446529778+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.447793394+09:00 [INFO][AUSF][5gAka] Auth5gAkaComfirmRequest
2024-09-18T20:56:38.448114998+09:00 [INFO][AUSF][5gAka] res*: 6138303965316566383562353437373238336436393061633261303634613235
Xres*: 6138303965316566383562353437373238336436393061633261303634613235
2024-09-18T20:56:38.448833588+09:00 [INFO][AUSF][5gAka] 5G AKA confirmation succeeded
2024-09-18T20:56:38.449626450+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.450634192+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.454029482+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.455675924+09:00 [INFO][UDM][UEAU] Handle ConfirmAuthDataRequest
2024-09-18T20:56:38.456477591+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.457980478+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.461100291+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.463322517+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-status |
2024-09-18T20:56:38.463822248+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-ueau/v1/imsi-001010000000000/auth-events |
2024-09-18T20:56:38.464178499+09:00 [INFO][AUSF][GIN] | 200 |       127.0.0.1 | PUT     | /nausf-auth/v1/ue-authentications/suci-0-001-01-0000-0-0-0000000000/5g-aka-confirmation |
2024-09-18T20:56:38.464837410+09:00 [INFO][AMF][Gmm] Handle event[Authentication Success], transition from [Authentication] to [SecurityMode]
2024-09-18T20:56:38.465267627+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Security Mode Command
2024-09-18T20:56:38.465515845+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44965] Send Downlink Nas Transport
2024-09-18T20:56:38.466365770+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3560 timer
2024-09-18T20:56:38.467717202+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44965] Handle UplinkNASTransport
2024-09-18T20:56:38.467992334+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44965] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2024-09-18T20:56:38.468371339+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [SecurityMode] to [SecurityMode]
2024-09-18T20:56:38.468505001+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Security Mode Complete
2024-09-18T20:56:38.468527771+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3560 timer
2024-09-18T20:56:38.468564164+09:00 [INFO][AMF][Gmm] Handle event[SecurityMode Success], transition from [SecurityMode] to [ContextSetup]
2024-09-18T20:56:38.468806434+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle InitialRegistration
2024-09-18T20:56:38.469612951+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.471046326+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.473773663+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.474611176+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.475789697+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2024-09-18T20:56:38.478250694+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.479661305+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.483269524+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.484687279+09:00 [INFO][UDM][SDM] Handle GetNssai
2024-09-18T20:56:38.485487351+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.486894675+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.489990419+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.491297062+09:00 [INFO][UDR][DataRepo] QueryAmDataProcedure: ueId: imsi-001010000000000, servingPlmnId: 00101
2024-09-18T20:56:38.492186161+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data |
2024-09-18T20:56:38.492697348+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/nssai?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2024-09-18T20:56:38.493268007+09:00 [INFO][AMF][Gmm] RequestedNssai: &{Iei:47 Len:5 Buffer:[4 1 1 2 3]}
2024-09-18T20:56:38.493299391+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] RequestedNssai - ServingSnssai: &{Sst:1 Sd:010203}, HomeSnssai: <nil>
2024-09-18T20:56:38.494228275+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.496052119+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.500944637+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.501974793+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.503237441+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2024-09-18T20:56:38.503934950+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.505170997+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.508455432+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.509837580+09:00 [INFO][UDM][UECM] Handle RegistrationAmf3gppAccess
2024-09-18T20:56:38.509882059+09:00 [INFO][UDM][UECM] UEID: imsi-001010000000000
2024-09-18T20:56:38.510514775+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.511580999+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.514487510+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.516481518+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/amf-3gpp-access |
2024-09-18T20:56:38.517101534+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | PUT     | /nudm-uecm/v1/imsi-001010000000000/registrations/amf-3gpp-access |
2024-09-18T20:56:38.518100861+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.519491622+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.522539443+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.523718683+09:00 [INFO][UDM][SDM] Handle GetAmData
2024-09-18T20:56:38.524622051+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.525772107+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.528868529+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.529882044+09:00 [INFO][UDR][DataRepo] QueryAmDataProcedure: ueId: imsi-001010000000000, servingPlmnId: 00101
2024-09-18T20:56:38.530243173+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2024-09-18T20:56:38.530745910+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/am-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2024-09-18T20:56:38.531818815+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.533087359+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.536278598+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.537315753+09:00 [INFO][UDM][SDM] Handle GetSmfSelectData
2024-09-18T20:56:38.537743171+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.539339752+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.542283627+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.543809011+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/smf-selection-subscription-data |
2024-09-18T20:56:38.544303464+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/smf-select-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2024-09-18T20:56:38.545392216+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.547052012+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.550533763+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.553502638+09:00 [INFO][UDM][SDM] Handle GetUeContextInSmfData
2024-09-18T20:56:38.554406721+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.555445806+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.558297637+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.560185395+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/smf-registrations |
2024-09-18T20:56:38.560928385+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/ue-context-in-smf-data |
2024-09-18T20:56:38.561754642+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.563484823+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.566693577+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.567976419+09:00 [INFO][UDM][SDM] Handle Subscribe
2024-09-18T20:56:38.568864662+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.569971647+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.572906045+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.574066788+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |
2024-09-18T20:56:38.574598421+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v1/imsi-001010000000000/sdm-subscriptions |
2024-09-18T20:56:38.575544042+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.577063763+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.579653171+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.580462020+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.581663143+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=PCF |
2024-09-18T20:56:38.582189539+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.583319814+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.586733299+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.588627974+09:00 [INFO][PCF][AmPol] Handle AM Policy Create Request
2024-09-18T20:56:38.589486219+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.590738802+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.593193473+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.593976423+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.594802439+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=UDR |
2024-09-18T20:56:38.595493085+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.596598097+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.599443405+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.601834969+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/policy-data/ues/imsi-001010000000000/am-data |
2024-09-18T20:56:38.603075599+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.604980221+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.609342095+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.610329137+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.611576391+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?guami=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22amfId%22%3A%22cafe00%22%7D&requester-nf-type=PCF&target-nf-type=AMF |
2024-09-18T20:56:38.612182610+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.613439025+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.616530921+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.617763425+09:00 [INFO][AMF][Comm] Handle AMF Status Change Subscribe Request
2024-09-18T20:56:38.617813138+09:00 [INFO][AMF][Comm] new AMF Status Subscription[1]
2024-09-18T20:56:38.618180195+09:00 [INFO][AMF][GIN] | 201 |       127.0.0.1 | POST    | /namf-comm/v1/subscriptions |
2024-09-18T20:56:38.618651850+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-am-policy-control/v1/policies |
2024-09-18T20:56:38.619029072+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Registration Accept
2024-09-18T20:56:38.619196399+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44965] Send Initial Context Setup Request
2024-09-18T20:56:38.620537574+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3550 timer
2024-09-18T20:56:38.621305962+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44965] Handle InitialContextSetupResponse
2024-09-18T20:56:38.621348052+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44965] Handle InitialContextSetupResponse (RAN UE NGAP ID: 1)
2024-09-18T20:56:38.825994148+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44965] Handle UplinkNASTransport
2024-09-18T20:56:38.826021510+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44965] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2024-09-18T20:56:38.826059931+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [ContextSetup] to [ContextSetup]
2024-09-18T20:56:38.826066997+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Registration Complete
2024-09-18T20:56:38.826072766+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3550 timer
2024-09-18T20:56:38.826108425+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Configuration Update Command
2024-09-18T20:56:38.826117099+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44965] Send Downlink Nas Transport
2024-09-18T20:56:38.826792711+09:00 [INFO][AMF][Gmm] Handle event[ContextSetup Success], transition from [ContextSetup] to [Registered]
2024-09-18T20:56:38.828071996+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44965] Handle UplinkNASTransport
2024-09-18T20:56:38.828313055+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44965] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2024-09-18T20:56:38.828641216+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Registered] to [Registered]
2024-09-18T20:56:38.828807753+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle UL NAS Transport
2024-09-18T20:56:38.828991076+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Transport 5GSM Message to SMF
2024-09-18T20:56:38.829017438+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Select SMF [snssai: {Sst:1 Sd:010203}, dnn: internet]
2024-09-18T20:56:38.831368754+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.832583374+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.835581904+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.836369905+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.837337090+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=NSSF |
2024-09-18T20:56:38.837949721+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.839313614+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.842195098+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.843288252+09:00 [INFO][NSSF][NsSel] Handle NSSelectionGet
2024-09-18T20:56:38.843641552+09:00 [WARN][NSSF][Util] No TA {"plmnId":{"mcc":"001","mnc":"01"},"tac":"000001"} in NSSF configuration
2024-09-18T20:56:38.844181210+09:00 [INFO][NSSF][GIN] | 200 |       127.0.0.1 | GET     | /nnssf-nsselection/v1/network-slice-information?nf-id=674ab488-184d-488a-9c3e-6d6a25abf14f&nf-type=AMF&slice-info-request-for-pdu-session=%7B%22sNssai%22%3A%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%2C%22roamingIndication%22%3A%22NON_ROAMING%22%7D&tai=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22tac%22%3A%22000001%22%7D |
2024-09-18T20:56:38.844622968+09:00 [WARN][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] nsiInformation is still nil, use default NRF[http://127.0.0.10:8000]
2024-09-18T20:56:38.845338333+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.846795665+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.849237180+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.849838236+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.850849856+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?dnn=internet&preferred-locality=area1&requester-nf-type=AMF&service-names=nsmf-pdusession&snssais=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D&target-nf-type=SMF&target-plmn-list=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2024-09-18T20:56:38.851632811+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.852892424+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.855975125+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.857406010+09:00 [INFO][SMF][PduSess] Receive Create SM Context Request
2024-09-18T20:56:38.858417223+09:00 [INFO][SMF][PduSess] In HandlePDUSessionSMContextCreate
2024-09-18T20:56:38.858786839+09:00 [INFO][SMF][CTX] UrrPeriod: 30s
2024-09-18T20:56:38.859095311+09:00 [INFO][SMF][CTX] UrrThreshold: 500000
2024-09-18T20:56:38.859826999+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.861087987+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.863429646+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.864269300+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.865356518+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=UDM |
2024-09-18T20:56:38.865861068+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Send NF Discovery Serving UDM Successfully
2024-09-18T20:56:38.866436267+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.867414704+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.870579119+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.871529222+09:00 [INFO][UDM][SDM] Handle GetSmData
2024-09-18T20:56:38.872671598+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.874080375+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.877082217+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.877644539+09:00 [INFO][UDM][SDM] getSmDataProcedure: SUPI[imsi-001010000000000] PLMNID[00101] DNN[internet] SNssai[{"sst":1,"sd":"010203"}]
2024-09-18T20:56:38.879508982+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/sm-data?single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2024-09-18T20:56:38.880046629+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/sm-data?dnn=internet&plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D&single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2024-09-18T20:56:38.880647483+09:00 [INFO][SMF][GSM] In HandlePDUSessionEstablishmentRequest
2024-09-18T20:56:38+09:00 [INFO][NAS][Convert] ProtocolOrContainerList:  [0xc0002df880 0xc0002df8a0]
2024-09-18T20:56:38.880905045+09:00 [INFO][SMF][GSM] Protocol Configuration Options
2024-09-18T20:56:38.880951274+09:00 [INFO][SMF][GSM] &{[0xc0002df880 0xc0002df8a0]}
2024-09-18T20:56:38.880992567+09:00 [INFO][SMF][GSM] Didn't Implement container type IPAddressAllocationViaNASSignallingUL
2024-09-18T20:56:38.881921631+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.883136439+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.885623565+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.886415700+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.887658081+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-instance-id=674ab488-184d-488a-9c3e-6d6a25abf14f&target-nf-type=AMF |
2024-09-18T20:56:38.888592233+09:00 [INFO][SMF][Consumer] SendNFDiscoveryServingAMF ok
2024-09-18T20:56:38.889018071+09:00 [INFO][SMF][CTX] Allocated UE IP address: 10.60.0.1
2024-09-18T20:56:38.889385945+09:00 [INFO][SMF][CTX] Selected UPF: PSA-UPF1
2024-09-18T20:56:38.891355067+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.893398175+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.895908233+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.896846973+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.898024745+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=SMF&target-nf-type=PCF |
2024-09-18T20:56:38.899080076+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.901704926+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.905808308+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.907721143+09:00 [INFO][PCF][SMpolicy] Handle CreateSmPolicy
2024-09-18T20:56:38.908746155+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.910276261+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.913205581+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.915399716+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/policy-data/ues/imsi-001010000000000/sm-data?dnn=internet&snssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2024-09-18T20:56:38.920085905+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.921391523+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.924370626+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.925715995+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/application-data/influenceData?dnns=internet&internal-Group-Ids=&snssais=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D&supis=imsi-001010000000000 |
2024-09-18T20:56:38.925890379+09:00 [INFO][PCF][SMpolicy] Matched [0] trafficInfluDatas from UDR
2024-09-18T20:56:38.926356838+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.927604920+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.930654283+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.931942957+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v1/application-data/influenceData/subs-to-notify |
2024-09-18T20:56:38.932958071+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.934039185+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.936814842+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.937700058+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.938358119+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=BSF |
2024-09-18T20:56:38.939122213+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-smpolicycontrol/v1/sm-policies |
2024-09-18T20:56:38.939828070+09:00 [INFO][SMF][PduSess] CHF Selection for SMContext SUPI[imsi-001010000000000] PDUSessionID[1]
2024-09-18T20:56:38.940862809+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.942148610+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.944624462+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.945418815+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-09-18T20:56:38.946288802+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=CHF |
2024-09-18T20:56:38.946601453+09:00 [INFO][SMF][Charging] Handle SendConvergedChargingRequest
2024-09-18T20:56:38.947080125+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.948097704+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.951030355+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.954648893+09:00 [INFO][CHF][ChargingPost] HandleChargingdataInitial
2024-09-18T20:56:38.954721794+09:00 [INFO][CHF][ChargingPost] SMF charging event
2024-09-18T20:56:38.954950086+09:00 [ERRO][CHF][ChargingPost] Charging gateway fail to send CDR to billing domain dial tcp 127.0.0.1:2121: connect: connection refused
2024-09-18T20:56:38.955220647+09:00 [INFO][CHF][ChargingPost] Open CDR for UE imsi-001010000000000
2024-09-18T20:56:38.955514480+09:00 [INFO][CHF][ChargingPost] NewChfUe imsi-001010000000000
2024-09-18T20:56:38.956029018+09:00 [INFO][CHF][GIN] | 201 |       127.0.0.1 | POST    | /nchf-convergedcharging/v3/chargingdata |
2024-09-18T20:56:38.956847230+09:00 [INFO][SMF][Charging] Send Charging Data Request[Init] successfully
2024-09-18T20:56:38.957224867+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Install PCCRule[PccRuleId-1]
2024-09-18T20:56:38.957250017+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] No srcTcData and tgtTcData. Nothing to do
2024-09-18T20:56:38.957543449+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Install PCCRule[PccRuleId-2]
2024-09-18T20:56:38.957557525+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] No srcTcData and tgtTcData. Nothing to do
2024-09-18T20:56:38.957600725+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Has pre-config route
2024-09-18T20:56:38.957708158+09:00 [INFO][SMF][GIN] | 201 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts |
2024-09-18T20:56:38.958147665+09:00 [INFO][SMF][PduSess] Sending PFCP Session Establishment Request
2024-09-18T20:56:38.958698417+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] create smContext[pduSessionID: 1] Success
2024-09-18T20:56:38.958469433+09:00 [INFO][SMF][PduSess] Sending PFCP Session Establishment Request
2024-09-18T20:56:38.961546206+09:00 [INFO][SMF][PduSess] Received PFCP Session Establishment Accepted Response
2024-09-18T20:56:38.962363069+09:00 [INFO][SMF][PduSess] Received PFCP Session Establishment Accepted Response
2024-09-18T20:56:38.963949239+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.965182915+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.968555882+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.970020678+09:00 [INFO][AMF][Producer] Handle N1N2 Message Transfer Request
2024-09-18T20:56:38.970599251+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44965] Send PDU Session Resource Setup Request
2024-09-18T20:56:38.971631215+09:00 [INFO][AMF][GIN] | 200 |       127.0.0.1 | POST    | /namf-comm/v1/ue-contexts/imsi-001010000000000/n1-n2-messages |
2024-09-18T20:56:38.973733668+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44965] Handle PDUSessionResourceSetupResponse
2024-09-18T20:56:38.974061468+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44965] Handle PDUSessionResourceSetupResponse (RAN UE NGAP ID: 1)
2024-09-18T20:56:38.975008827+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.976555048+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.979633902+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.981029301+09:00 [INFO][SMF][PduSess] Receive Update SM Context Request
2024-09-18T20:56:38.983327681+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Accepted Response from AN UPF
2024-09-18T20:56:38.983588106+09:00 [INFO][SMF][PFCP] Add PSAAndULCL
2024-09-18T20:56:38.983716526+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] In AddPDUSessionAnchorAndULCL
2024-09-18T20:56:38.983939936+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Establish PSA2
2024-09-18T20:56:38.984186196+09:00 [INFO][SMF][PduSess] In EstablishULCL
2024-09-18T20:56:38.984332723+09:00 [INFO][SMF][PFCP] [SMF] Establish ULCL msg has been send
2024-09-18T20:56:38.984448431+09:00 [INFO][SMF][PduSess] Sending PFCP Session Modification Request
2024-09-18T20:56:38.988239891+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Response
2024-09-18T20:56:38.988583612+09:00 [INFO][SMF][PduSess] Sending PFCP Session Modification Request
2024-09-18T20:56:38.992369042+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Response
2024-09-18T20:56:38.992418982+09:00 [INFO][SMF][CTX] [SMF] Add PSA success
2024-09-18T20:56:38.992605943+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Establish PSA2
2024-09-18T20:56:38.992670171+09:00 [INFO][SMF][Charging] Handle SendConvergedChargingRequest
2024-09-18T20:56:38.993483275+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-09-18T20:56:38.994880634+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-09-18T20:56:38.997887591+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-09-18T20:56:38.999151335+09:00 [INFO][CHF][ChargingPost] HandleChargingdataUpdate
2024-09-18T20:56:38.999598064+09:00 [INFO][CHF][ChargingPost] In BuildConvergedChargingDataUpdateResopone
2024-09-18T20:56:39.000044186+09:00 [ERRO][CHF][ChargingPost] Charging gateway fail to send CDR to billing domain dial tcp 127.0.0.1:2121: connect: connection refused
2024-09-18T20:56:39.001075853+09:00 [INFO][CHF][GIN] | 200 |       127.0.0.1 | POST    | /nchf-convergedcharging/v3/chargingdata/imsi-001010000000000SMF0/update |
2024-09-18T20:56:39.001242544+09:00 [INFO][SMF][Charging] Send Charging Data Request[Init] successfully
2024-09-18T20:56:39.001266989+09:00 [INFO][SMF][PduSess] Sending PFCP Session Establishment Request
2024-09-18T20:56:39.003377306+09:00 [INFO][SMF][PduSess] Received PFCP Session Establishment Accepted Response
2024-09-18T20:56:39.003640596+09:00 [INFO][SMF][PduSess] In EstablishULCL
2024-09-18T20:56:39.003963676+09:00 [INFO][SMF][PFCP] [SMF] Establish ULCL msg has been send
2024-09-18T20:56:39.003985265+09:00 [INFO][SMF][PduSess] Sending PFCP Session Modification Request
2024-09-18T20:56:39.007167106+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Response
2024-09-18T20:56:39.007347676+09:00 [INFO][SMF][PFCP] [SMF] Update PSA2 downlink msg has been send
2024-09-18T20:56:39.007364552+09:00 [INFO][SMF][PduSess] Sending PFCP Session Modification Request
2024-09-18T20:56:39.008555828+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Response
2024-09-18T20:56:39.008618647+09:00 [INFO][SMF][CTX] [SMF] Add PSA success
2024-09-18T20:56:39.008788055+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Establish PSA2
2024-09-18T20:56:39.008888932+09:00 [INFO][SMF][PduSess] In EstablishULCL
2024-09-18T20:56:39.008963322+09:00 [INFO][SMF][PFCP] [SMF] Establish ULCL msg has been send
2024-09-18T20:56:39.009170407+09:00 [INFO][SMF][PduSess] Sending PFCP Session Modification Request
2024-09-18T20:56:39.011179023+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Response
2024-09-18T20:56:39.011452827+09:00 [INFO][SMF][CTX] [SMF] Add PSA success
2024-09-18T20:56:39.011810043+09:00 [INFO][SMF][GIN] | 200 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts/urn:uuid:b891a72c-0d5e-4ad4-90fc-f32b12703a3a/modify |
```
The free5GC U-Plane (I-UPF) log when executed is as follows.
```
2024-09-18T20:56:38.944139737+09:00 [INFO][UPF][PFCP][LAddr:192.168.0.142:8805] handleSessionEstablishmentRequest
2024-09-18T20:56:38.944170154+09:00 [INFO][UPF][PFCP][LAddr:192.168.0.142:8805][CPNodeID:192.168.0.141][CPSEID:0x1][UPSEID:0x1] New session
2024-09-18T20:56:38.945213106+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:5 period:30000000000}]
2024-09-18T20:56:38.945275692+09:00 [INFO][UPF][Perio] new ticker [30s]
2024-09-18T20:56:38.945453725+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:6 period:30000000000}]
2024-09-18T20:56:38.966445213+09:00 [INFO][UPF][PFCP][LAddr:192.168.0.142:8805] handleSessionModificationRequest
2024-09-18T20:56:38.973046796+09:00 [INFO][UPF][PFCP][LAddr:192.168.0.142:8805] handleSessionModificationRequest
2024-09-18T20:56:38.975163360+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:5 period:30000000000}]
2024-09-18T20:56:38.975481262+09:00 [ERRO][UPF][PFCP][LAddr:192.168.0.142:8805][CPNodeID:192.168.0.141][CPSEID:0x1][UPSEID:0x1] Mod CreateURR error: file exists
2024-09-18T20:56:38.975537225+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:6 period:30000000000}]
2024-09-18T20:56:38.975774836+09:00 [ERRO][UPF][PFCP][LAddr:192.168.0.142:8805][CPNodeID:192.168.0.141][CPSEID:0x1][UPSEID:0x1] Mod CreateURR error: file exists
2024-09-18T20:56:38.988742240+09:00 [INFO][UPF][PFCP][LAddr:192.168.0.142:8805] handleSessionModificationRequest
2024-09-18T20:56:38.989660911+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:5 period:30000000000}]
2024-09-18T20:56:38.989968564+09:00 [ERRO][UPF][PFCP][LAddr:192.168.0.142:8805][CPNodeID:192.168.0.141][CPSEID:0x1][UPSEID:0x1] Mod CreateURR error: file exists
2024-09-18T20:56:38.990177919+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:6 period:30000000000}]
2024-09-18T20:56:38.990524868+09:00 [ERRO][UPF][PFCP][LAddr:192.168.0.142:8805][CPNodeID:192.168.0.141][CPSEID:0x1][UPSEID:0x1] Mod CreateURR error: file exists
2024-09-18T20:56:38.993584491+09:00 [INFO][UPF][PFCP][LAddr:192.168.0.142:8805] handleSessionModificationRequest
2024-09-18T20:56:38.994409573+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:1 period:30000000000}]
2024-09-18T20:56:38.994506707+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:2 period:30000000000}]
```
The free5GC U-Plane (PSA-UPF1) log when executed is as follows.
```
2024-09-18T20:56:38.913806408+09:00 [INFO][UPF][PFCP][LAddr:192.168.0.143:8805] handleSessionEstablishmentRequest
2024-09-18T20:56:38.913833948+09:00 [INFO][UPF][PFCP][LAddr:192.168.0.143:8805][CPNodeID:192.168.0.141][CPSEID:0x2][UPSEID:0x1] New session
2024-09-18T20:56:38.914895934+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:3 period:30000000000}]
2024-09-18T20:56:38.915082763+09:00 [INFO][UPF][Perio] new ticker [30s]
2024-09-18T20:56:38.915152708+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:4 period:30000000000}]
2024-09-18T20:56:38.939335902+09:00 [INFO][UPF][PFCP][LAddr:192.168.0.143:8805] handleSessionModificationRequest
2024-09-18T20:56:38.940009087+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:3 period:30000000000}]
2024-09-18T20:56:38.940259975+09:00 [ERRO][UPF][PFCP][LAddr:192.168.0.143:8805][CPNodeID:192.168.0.141][CPSEID:0x2][UPSEID:0x1] Mod CreateURR error: file exists
2024-09-18T20:56:38.940300165+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:4 period:30000000000}]
2024-09-18T20:56:38.940667397+09:00 [ERRO][UPF][PFCP][LAddr:192.168.0.143:8805][CPNodeID:192.168.0.141][CPSEID:0x2][UPSEID:0x1] Mod CreateURR error: file exists
```
The free5GC U-Plane (PSA-UPF2) log when executed is as follows.
```
2024-09-18T20:56:38.963748273+09:00 [INFO][UPF][PFCP][LAddr:192.168.0.144:8805] handleSessionEstablishmentRequest
2024-09-18T20:56:38.963777348+09:00 [INFO][UPF][PFCP][LAddr:192.168.0.144:8805][CPNodeID:192.168.0.141][CPSEID:0x3][UPSEID:0x1] New session
2024-09-18T20:56:38.964409137+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:3 period:30000000000}]
2024-09-18T20:56:38.964727499+09:00 [INFO][UPF][Perio] new ticker [30s]
2024-09-18T20:56:38.964882357+09:00 [INFO][UPF][Perio] recv event[TYPE_PERIO_ADD][{eType:1 lSeid:1 urrid:4 period:30000000000}]
2024-09-18T20:56:38.969859911+09:00 [INFO][UPF][PFCP][LAddr:192.168.0.144:8805] handleSessionModificationRequest
```
The TUNnel interface `uesimtun0` is created as follows.
```
# ip addr show
...
6: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none 
    inet 10.60.0.1/32 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::dc97:68c:8d2f:99aa/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
...
```

<a id="ping_google"></a>

#### Ping google.com going through PSA-UPF1

Confirm by using `tcpdump` that the packet goes through `if=upfgtp` on U-Plane (PSA-UPF1).
```
# ping google.com -I uesimtun0 -n
PING google.com (142.250.207.14) from 10.60.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 142.250.207.14: icmp_seq=1 ttl=61 time=19.9 ms
64 bytes from 142.250.207.14: icmp_seq=2 ttl=61 time=20.1 ms
64 bytes from 142.250.207.14: icmp_seq=3 ttl=61 time=18.8 ms
```
The `tcpdump` log on U-Plane (PSA-UPF1) is as follows.
```
21:04:21.473820 IP 10.60.0.1 > 142.250.207.14: ICMP echo request, id 1470, seq 1, length 64
21:04:21.490411 IP 142.250.207.14 > 10.60.0.1: ICMP echo reply, id 1470, seq 1, length 64
21:04:22.475624 IP 10.60.0.1 > 142.250.207.14: ICMP echo request, id 1470, seq 2, length 64
21:04:22.491507 IP 142.250.207.14 > 10.60.0.1: ICMP echo reply, id 1470, seq 2, length 64
21:04:23.476562 IP 10.60.0.1 > 142.250.207.14: ICMP echo request, id 1470, seq 3, length 64
21:04:23.492152 IP 142.250.207.14 > 10.60.0.1: ICMP echo reply, id 1470, seq 3, length 64
```
**Note. Make sure that the packets are not routed from I-UPF & PSA-UPF2 to the Internet.**

<a id="ping_8"></a>

#### Ping 8.8.8.8 going through I-UPF

Confirm by using `tcpdump` that the packet goes through `if=upfgtp` on U-Plane (I-UPF).
```
# ping 8.8.8.8 -I uesimtun0 -n
PING 8.8.8.8 (8.8.8.8) from 10.60.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=61 time=15.2 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=61 time=12.1 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=61 time=11.3 ms
```
The `tcpdump` log on U-Plane (I-UPF) is as follows.
```
21:05:42.029294 IP 10.60.0.1 > 8.8.8.8: ICMP echo request, id 1475, seq 1, length 64
21:05:42.041588 IP 8.8.8.8 > 10.60.0.1: ICMP echo reply, id 1475, seq 1, length 64
21:05:43.031135 IP 10.60.0.1 > 8.8.8.8: ICMP echo request, id 1475, seq 2, length 64
21:05:43.040134 IP 8.8.8.8 > 10.60.0.1: ICMP echo reply, id 1475, seq 2, length 64
21:05:44.031321 IP 10.60.0.1 > 8.8.8.8: ICMP echo request, id 1475, seq 3, length 64
21:05:44.040366 IP 8.8.8.8 > 10.60.0.1: ICMP echo reply, id 1475, seq 3, length 64
```
**Note. Make sure that the packets are not routed from PSA-UPFs to the Internet.**

<a id="ping_docker1"></a>

#### Ping 172.17.0.1 going through PSA-UPF1

Confirm by using `tcpdump` that the packet goes through `if=upfgtp` on U-Plane (PSA-UPF1).
```
# ping 172.17.0.1 -I uesimtun0 -n
PING 172.17.0.1 (172.17.0.1) from 10.60.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 172.17.0.1: icmp_seq=1 ttl=64 time=3.06 ms
64 bytes from 172.17.0.1: icmp_seq=2 ttl=64 time=3.17 ms
64 bytes from 172.17.0.1: icmp_seq=3 ttl=64 time=3.13 ms
```
The `tcpdump` log on U-Plane (PSA-UPF1) is as follows.
```
21:06:26.335859 IP 10.60.0.1 > 172.17.0.1: ICMP echo request, id 1476, seq 1, length 64
21:06:26.335990 IP 172.17.0.1 > 10.60.0.1: ICMP echo reply, id 1476, seq 1, length 64
21:06:27.337463 IP 10.60.0.1 > 172.17.0.1: ICMP echo request, id 1476, seq 2, length 64
21:06:27.337586 IP 172.17.0.1 > 10.60.0.1: ICMP echo reply, id 1476, seq 2, length 64
21:06:28.338503 IP 10.60.0.1 > 172.17.0.1: ICMP echo request, id 1476, seq 3, length 64
21:06:28.338603 IP 172.17.0.1 > 10.60.0.1: ICMP echo reply, id 1476, seq 3, length 64
```
**Note. Make sure that the packets are not routed from I-UPF & PSA-UPF2 to anywhere.**

<a id="ping_docker2"></a>

#### Ping 172.18.0.1 going through PSA-UPF2

Confirm by using `tcpdump` that the packet goes through `if=upfgtp` on U-Plane (PSA-UPF2).
```
# ping 172.18.0.1 -I uesimtun0 -n
PING 172.18.0.1 (172.18.0.1) from 10.60.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 172.18.0.1: icmp_seq=1 ttl=64 time=4.10 ms
64 bytes from 172.18.0.1: icmp_seq=2 ttl=64 time=3.25 ms
64 bytes from 172.18.0.1: icmp_seq=3 ttl=64 time=2.86 ms
```
The `tcpdump` log on U-Plane (PSA-UPF2) is as follows.
```
21:07:23.172869 IP 10.60.0.1 > 172.18.0.1: ICMP echo request, id 1477, seq 1, length 64
21:07:23.172992 IP 172.18.0.1 > 10.60.0.1: ICMP echo reply, id 1477, seq 1, length 64
21:07:24.173407 IP 10.60.0.1 > 172.18.0.1: ICMP echo request, id 1477, seq 2, length 64
21:07:24.173534 IP 172.18.0.1 > 10.60.0.1: ICMP echo reply, id 1477, seq 2, length 64
21:07:25.174260 IP 10.60.0.1 > 172.18.0.1: ICMP echo request, id 1477, seq 3, length 64
21:07:25.174363 IP 172.18.0.1 > 10.60.0.1: ICMP echo reply, id 1477, seq 3, length 64
```
**Note. Make sure that the packets are not routed from I-UPF & PSA-UPF1 to anywhere.**

---
Using ULCL, I was able to confirm the very simple configuration for controlling the communication path to specific destination IP addresses.
I would like to thank the excellent developers and all the contributors of free5GC and UERANSIM.

<a id="changelog"></a>

## Changelog (summary)

- [2024.09.18] According to [this](https://github.com/free5gc/free5gc/issues/599#issuecomment-2357448799), the bug in the ULCL function of SMF has been fixed, so updated to go-upf v1.2.3.
- [2024.09.16] Added a note at the beginning that ULCL can be confirmed using [NextMN-UPF](https://github.com/nextmn/upf) as UPF.
- [2024.09.15] Updated to free5GC v3.4.3 (2024.09.12) and go-upf v1.2.1 (2023.12.19).
- [2022.10.09] Initial release.
