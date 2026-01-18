---
layout: post
title: "wi-fi linux flow"
date: 2026-01-18 9:00:00 +0530
categories: [connman]
tags: [connman,wlan]
description: "wi-fi linux flow in  Depth"

---

# Complete Linux WiFi Connection Flow - End to End

## The Big Picture: All Layers

```
┌─────────────────────────────────────────────────────────┐
│  USER SPACE                                             │
├─────────────────────────────────────────────────────────┤
│  [1] NetworkManager / ConnMan / wpa_supplicant          │
│      "I want to connect to 'HomeWiFi'"                  │
└──────────────────────┬──────────────────────────────────┘
                       │ nl80211 (netlink)
                       │ (User ↔ Kernel communication)
                       ▼
┌─────────────────────────────────────────────────────────┐
│  KERNEL SPACE                                           │
├─────────────────────────────────────────────────────────┤
│  [2] cfg80211                                           │
│      "Central wireless configuration layer"             │
│      - Validates requests                               │
│      - Manages regulatory                               │
│      - Tracks scan results                              │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  [3] mac80211                                           │
│      "Software MAC layer implementation"                │
│      - Handles 802.11 frames                            │
│      - Authentication/Association                       │
│      - Encryption (if in software)                      │
│      - Rate control                                     │
│      - Power management                                 │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  [4] ath12k / ath11k / ath10k Driver                   │
│      "Hardware-specific driver"                         │
│      - Talks to specific WiFi chip                      │
│      - Manages TX/RX rings                              │
│      - Configures hardware registers                    │
│      - DMA operations                                   │
└──────────────────────┬──────────────────────────────────┘
                       │ PCI / USB / SDIO
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  [5] WiFi Chip Hardware                                 │
│      ├─ Firmware (running on chip)                      │
│      │  - Low-level MAC                                 │
│      │  - PHY control                                   │
│      │  - Beamforming                                   │
│      └─ Hardware                                        │
│         - RF frontend                                   │
│         - Antennas                                      │
│         - Actual radio transmission                     │
└─────────────────────────────────────────────────────────┘
                       │ Radio waves
                       ▼
                   [6] Access Point (WiFi Router)
```

---

## Complete Connection Flow: Step by Step

### Scenario: Connect to "HomeWiFi" WPA2 network

Let me trace this through **all layers** with actual code paths.

---

## Phase 1: User Initiates Connection

### Step 1.1: User Command

```bash
# Option A: Using wpa_supplicant directly
wpa_supplicant -i wlan0 -c /etc/wpa_supplicant.conf

# Option B: Using ConnMan
connmanctl connect wifi_xxxxx_HomeWiFi_managed_psk

# Option C: Using NetworkManager
nmcli device wifi connect "HomeWiFi" password "mypassword"
```

**What happens:**
```
Application (wpa_supplicant / ConnMan / NetworkManager)
  │
  ├─ Read configuration
  ├─ Know: SSID = "HomeWiFi", Security = WPA2-PSK, Password = "xxx"
  └─ Need to: Scan → Authenticate → Associate → Get IP
```

---

## Phase 2: Scanning for Networks

### Step 2.1: Application Requests Scan (Userspace)

**wpa_supplicant/ConnMan sends:**
```c
// Userspace: wpa_supplicant or ConnMan
// Uses nl80211 netlink interface

struct nl_msg *msg;
msg = nlmsg_alloc();

// NL80211_CMD_TRIGGER_SCAN command
genlmsg_put(msg, 0, 0, nl80211_id, 0, 0, NL80211_CMD_TRIGGER_SCAN, 0);

// Specify interface
nla_put_u32(msg, NL80211_ATTR_IFINDEX, if_nametoindex("wlan0"));

// Send to kernel
nl_send_auto(socket, msg);
```

**nl80211 message structure:**
```
Netlink Message:
  Command: NL80211_CMD_TRIGGER_SCAN
  Attributes:
    - IFINDEX: 3 (wlan0)
    - SCAN_SSIDS: ["HomeWiFi"] (optional, can be broadcast)
    - SCAN_FREQUENCIES: [2412, 2437, 2462, ...] (optional)
```

---

### Step 2.2: cfg80211 Processes Scan Request (Kernel)

**File:** `net/wireless/scan.c`

```c
// cfg80211 receives nl80211 command
int cfg80211_trigger_scan(struct cfg80211_registered_device *rdev,
                         struct wireless_dev *wdev,
                         struct cfg80211_scan_request *request)
{
    // Validate request
    if (!rdev->ops->scan)
        return -EOPNOTSUPP;
    
    // Check regulatory domain
    if (!cfg80211_is_allowed(request->channels))
        return -EINVAL;
    
    // Call driver's scan function
    return rdev->ops->scan(&rdev->wiphy, request);
}
```

**cfg80211 validates:**
- Is interface UP?
- Is scan allowed in current country?
- Are requested channels legal?
- Is another scan already running?

---

### Step 2.3: mac80211 Prepares Scan (Kernel)

**File:** `net/mac80211/scan.c`

```c
// mac80211 scan implementation
static int __ieee80211_start_scan(struct ieee80211_sub_if_data *sdata,
                                 struct cfg80211_scan_request *req)
{
    struct ieee80211_local *local = sdata->local;
    
    // Switch to scanning state
    local->scanning |= SCAN_SW_SCANNING;
    
    // For each channel to scan
    for (i = 0; i < req->n_channels; i++) {
        struct ieee80211_channel *chan = req->channels[i];
        
        // Switch to this channel
        ieee80211_hw_config(local, IEEE80211_CONF_CHANGE_CHANNEL);
        
        // Send probe request
        ieee80211_send_probe_req(sdata, NULL, req->ssids[0].ssid, 
                                req->ssids[0].ssid_len);
        
        // Wait for probe responses (typically 100-200ms per channel)
        msleep(scan_dwell_time);
    }
    
    // Scan complete, notify cfg80211
    cfg80211_scan_done(req, &info);
}
```

**What mac80211 does:**
1. Switch radio to each channel (2.4 GHz and 5 GHz)
2. On each channel:
   - Send Probe Request frame
   - Listen for Probe Response frames
   - Wait ~100ms
3. Collect all responses
4. Return results to cfg80211

---

### Step 2.4: ath12k Driver Executes Scan (Kernel)

**File:** `drivers/net/wireless/ath/ath12k/mac.c`

```c
// ath12k implements mac80211 scan callback
static int ath12k_mac_op_hw_scan(struct ieee80211_hw *hw,
                                struct ieee80211_vif *vif,
                                struct ieee80211_scan_request *hw_req)
{
    struct ath12k *ar = hw->priv;
    struct scan_req_params arg;
    int ret;
    
    // Build scan parameters for firmware
    arg.scan_id = ar->scan.scan_id++;
    arg.n_channels = hw_req->req.n_channels;
    arg.dwell_time_active = 100;  // ms to listen on each channel
    arg.dwell_time_passive = 200;
    
    // Copy channel list
    for (i = 0; i < arg.n_channels; i++) {
        arg.channels[i] = hw_req->req.channels[i]->center_freq;
    }
    
    // Send scan command to firmware via WMI
    ret = ath12k_wmi_send_scan_start_cmd(ar, &arg);
    if (ret) {
        ath12k_warn(ar->ab, "failed to send scan start: %d\n", ret);
        return ret;
    }
    
    // Firmware will scan and send results back via events
    return 0;
}
```

**What ath12k does:**
1. Translates mac80211 scan request to firmware format
2. Sends WMI (Wireless Management Interface) command to firmware
3. Firmware takes over and performs actual scan
4. Driver waits for scan complete event from firmware

---

### Step 2.5: Firmware Scans (WiFi Chip)

**Inside Qualcomm firmware (black box, but we know what it does):**

```
For each channel in list:
  1. Switch radio to channel frequency
  2. Configure RF frontend
  3. Send Probe Request 802.11 frame:
     ┌──────────────────────────────────────┐
     │ Frame Control: Probe Request         │
     │ Duration: 0                          │
     │ DA: ff:ff:ff:ff:ff:ff (broadcast)    │
     │ SA: 00:11:22:33:44:55 (our MAC)      │
     │ BSSID: ff:ff:ff:ff:ff:ff             │
     │ SSID: "HomeWiFi"                     │
     │ Supported Rates: 6, 12, 24 Mbps...  │
     │ HT Capabilities (WiFi 4)             │
     │ VHT Capabilities (WiFi 5)            │
     │ HE Capabilities (WiFi 6)             │
     │ EHT Capabilities (WiFi 7)            │
     └──────────────────────────────────────┘
  
  4. Listen for Probe Response frames
  5. Collect beacon frames
  6. Store RSSI, channel, capabilities
  7. Move to next channel
  
After all channels scanned:
  8. Send scan complete event to driver
```

**Probe Response from AP looks like:**
```
┌──────────────────────────────────────┐
│ Frame Control: Probe Response        │
│ SA: aa:bb:cc:dd:ee:ff (AP's MAC)     │
│ SSID: "HomeWiFi"                     │
│ Supported Rates: 6-540 Mbps          │
│ Channel: 36 (5 GHz)                  │
│ RSSI: -45 dBm                        │
│ Capabilities:                        │
│   - WPA2-PSK encryption              │
│   - 802.11ax (WiFi 6)                │
│   - 80 MHz bandwidth                 │
│ RSN (Security) Information:          │
│   - Cipher: CCMP (AES)               │
│   - Authentication: PSK              │
└──────────────────────────────────────┘
```

---

### Step 2.6: Scan Results Bubble Up

**Firmware → Driver → mac80211 → cfg80211 → Userspace**

```c
// ath12k receives scan complete event from firmware
static void ath12k_wmi_event_scan_complete(struct ath12k *ar,
                                          struct sk_buff *skb)
{
    ath12k_dbg(ar->ab, ATH12K_DBG_WMI, "scan complete event\n");
    
    // Notify mac80211 that scan is done
    ieee80211_scan_completed(ar->hw, &info);
}

// mac80211 notifies cfg80211
void ieee80211_scan_completed(struct ieee80211_hw *hw,
                             struct cfg80211_scan_info *info)
{
    struct ieee80211_local *local = hw_to_local(hw);
    
    local->scanning = 0;
    
    // Tell cfg80211
    cfg80211_scan_done(local->scan_req, info);
}

// cfg80211 notifies userspace via nl80211
void cfg80211_scan_done(struct cfg80211_scan_request *request,
                       struct cfg80211_scan_info *info)
{
    // Send NL80211_CMD_NEW_SCAN_RESULTS to userspace
    nl80211_send_scan_result(request->wiphy, NL80211_CMD_NEW_SCAN_RESULTS);
}
```

**Userspace receives:**
```
Netlink Event:
  Command: NL80211_CMD_NEW_SCAN_RESULTS
  
  BSS 1:
    SSID: "HomeWiFi"
    BSSID: aa:bb:cc:dd:ee:ff
    Channel: 36 (5180 MHz)
    Signal: -45 dBm
    Capabilities: ESS, Privacy
    RSN: WPA2-PSK
    
  BSS 2:
    SSID: "Neighbor_WiFi"
    ...
```

**wpa_supplicant/ConnMan now has scan results!**

---

## Phase 3: Authentication and Association

### Step 3.1: wpa_supplicant Selects Network

```c
// wpa_supplicant code (userspace)
void wpa_supplicant_select_network(struct wpa_supplicant *wpa_s,
                                  struct wpa_ssid *ssid)
{
    // Found "HomeWiFi" in scan results
    // It's WPA2-PSK
    // We have the password
    
    // Step 1: Authenticate (802.11 authentication)
    wpa_supplicant_authenticate(wpa_s, ssid);
}
```

---

### Step 3.2: 802.11 Authentication (Open System)

**wpa_supplicant sends authentication via nl80211:**

```c
// nl80211 command: NL80211_CMD_AUTHENTICATE
struct nl_msg *msg = nlmsg_alloc();
genlmsg_put(msg, 0, 0, nl80211_id, 0, 0, NL80211_CMD_AUTHENTICATE, 0);

nla_put_u32(msg, NL80211_ATTR_IFINDEX, ifindex);
nla_put(msg, NL80211_ATTR_MAC, ETH_ALEN, ap_bssid);  // AP's MAC
nla_put_u32(msg, NL80211_ATTR_AUTH_TYPE, NL80211_AUTHTYPE_OPEN_SYSTEM);
nla_put(msg, NL80211_ATTR_SSID, ssid_len, ssid);

nl_send_auto(socket, msg);
```

**cfg80211 → mac80211 → ath12k:**

```c
// mac80211: net/mac80211/mlme.c
static int ieee80211_auth(struct ieee80211_sub_if_data *sdata)
{
    struct ieee80211_local *local = sdata->local;
    struct ieee80211_mgmt *mgmt;
    struct sk_buff *skb;
    
    // Build Authentication frame (802.11 management frame)
    skb = alloc_skb(sizeof(*mgmt) + extra, GFP_KERNEL);
    mgmt = (struct ieee80211_mgmt *)skb_put(skb, sizeof(*mgmt));
    
    mgmt->frame_control = cpu_to_le16(IEEE80211_FTYPE_MGMT |
                                      IEEE80211_STYPE_AUTH);
    mgmt->duration = 0;
    memcpy(mgmt->da, ap_addr, ETH_ALEN);       // Destination: AP
    memcpy(mgmt->sa, sdata->vif.addr, ETH_ALEN); // Source: Our MAC
    memcpy(mgmt->bssid, ap_addr, ETH_ALEN);    // BSSID: AP
    
    // Authentication algorithm: Open System
    mgmt->u.auth.auth_alg = cpu_to_le16(WLAN_AUTH_OPEN);
    mgmt->u.auth.auth_transaction = cpu_to_le16(1);  // Transaction 1
    mgmt->u.auth.status_code = 0;
    
    // Send via driver
    ieee80211_tx_skb(sdata, skb);
}
```

**ath12k transmits:**

```c
// ath12k: drivers/net/wireless/ath/ath12k/mac.c
static void ath12k_mac_op_tx(struct ieee80211_hw *hw,
                            struct ieee80211_tx_control *control,
                            struct sk_buff *skb)
{
    struct ath12k *ar = hw->priv;
    
    // Management frames (like Authentication) use special queue
    if (ieee80211_is_mgmt(hdr->frame_control)) {
        // Send via management TX queue
        ath12k_dp_tx_send_mgmt_frame(ar, skb);
    }
}

static int ath12k_dp_tx_send_mgmt_frame(struct ath12k *ar,
                                       struct sk_buff *skb)
{
    // Get TX descriptor
    struct hal_tx_desc *desc = ath12k_hal_tx_desc_get();
    
    // DMA map the frame
    dma_addr_t paddr = dma_map_single(ar->ab->dev, skb->data, skb->len);
    
    // Fill descriptor
    desc->buf_addr = paddr;
    desc->buf_len = skb->len;
    desc->info0 = FIELD_PREP(TX_MGMT_DESC, 1);  // Management frame
    
    // Kick hardware
    ath12k_hal_tx_ring_doorbell(ar->ab, HAL_TCL_MGMT_RING);
    
    // Firmware will transmit this 802.11 management frame
}
```

**Over the air (802.11 Authentication frame):**
```
┌────────────────────────────────────────┐
│ Frame Type: Management                 │
│ Subtype: Authentication                │
│ To: aa:bb:cc:dd:ee:ff (AP)            │
│ From: 00:11:22:33:44:55 (our station) │
│ Algorithm: Open System                 │
│ Transaction Sequence: 1                │
│ Status: Reserved                       │
└────────────────────────────────────────┘
```

**AP responds:**
```
┌────────────────────────────────────────┐
│ Frame Type: Management                 │
│ Subtype: Authentication                │
│ To: 00:11:22:33:44:55 (our station)   │
│ From: aa:bb:cc:dd:ee:ff (AP)          │
│ Transaction Sequence: 2                │
│ Status: Success (0x0000)               │
└────────────────────────────────────────┘
```

**ath12k receives response:**

```c
// ath12k RX path: drivers/net/wireless/ath/ath12k/dp_rx.c
static void ath12k_dp_rx_process_received_packets(struct ath12k *ar)
{
    struct sk_buff *skb;
    struct ieee80211_hdr *hdr;
    
    // Get packet from RX ring
    skb = ath12k_dp_rx_get_skb_from_ring(ar);
    
    hdr = (struct ieee80211_hdr *)skb->data;
    
    if (ieee80211_is_auth(hdr->frame_control)) {
        // This is authentication response
        ath12k_dbg(ar->ab, ATH12K_DBG_MAC, "RX authentication frame\n");
    }
    
    // Pass to mac80211
    ieee80211_rx_napi(ar->hw, NULL, skb, napi);
}
```

**mac80211 processes:**

```c
// mac80211: net/mac80211/mlme.c
static void ieee80211_rx_mgmt_auth(struct ieee80211_sub_if_data *sdata,
                                  struct ieee80211_mgmt *mgmt, size_t len)
{
    u16 status = le16_to_cpu(mgmt->u.auth.status_code);
    
    if (status == WLAN_STATUS_SUCCESS) {
        // Authentication successful!
        sdata->u.mgd.auth_data->status = status;
        
        // Now proceed to Association
        ieee80211_assoc(sdata);
    }
}
```

---

### Step 3.3: 802.11 Association

**Similar flow, but Association Request frame:**

```
mac80211 sends Association Request
  ↓
ath12k transmits over the air
  ↓
┌────────────────────────────────────────┐
│ Frame Type: Management                 │
│ Subtype: Association Request           │
│ To: aa:bb:cc:dd:ee:ff (AP)            │
│ Capabilities: ESS, Privacy             │
│ Listen Interval: 10                    │
│ SSID: "HomeWiFi"                       │
│ Supported Rates: 6, 9, 12, 18...      │
│ HT Capabilities                        │
│ VHT Capabilities                       │
│ HE Capabilities (WiFi 6)               │
│ EHT Capabilities (WiFi 7 if supported) │
│ RSN (Security):                        │
│   - Cipher: CCMP (AES)                 │
│   - AKM: PSK                           │
└────────────────────────────────────────┘
```

**AP responds:**
```
┌────────────────────────────────────────┐
│ Frame Type: Management                 │
│ Subtype: Association Response          │
│ Status: Success (0x0000)               │
│ AID: 1 (Association ID)                │
│ Supported Rates                        │
│ HT Operation                           │
│ VHT Operation                          │
│ HE Operation (WiFi 6)                  │
└────────────────────────────────────────┘
```

**Now we're associated! But not encrypted yet...**

---

## Phase 4: WPA2 4-Way Handshake

### Step 4.1: Key Exchange (EAPOL Frames)

**This happens in wpa_supplicant (userspace):**

```
Station (us)                              AP
    │                                     │
    │◄──── EAPOL Message 1 ───────────────┤
    │      (ANonce from AP)               │
    │                                     │
    ├───── EAPOL Message 2 ──────────────►│
    │      (SNonce from Station, MIC)     │
    │                                     │
    │◄──── EAPOL Message 3 ───────────────┤
    │      (GTK, MIC)                     │
    │                                     │
    ├───── EAPOL Message 4 ──────────────►│
    │      (Acknowledgment)               │
    │                                     │
    └──────── Keys Installed ──────────────┘
```

**EAPOL frames go through normal data path:**
- wpa_supplicant creates EAPOL frame
- Sent via regular socket
- Goes through network stack → mac80211 → ath12k
- Transmitted as data frames (not management)

**After handshake:**
```c
// wpa_supplicant installs keys via nl80211
struct nl_msg *msg = nlmsg_alloc();
genlmsg_put(msg, 0, 0, nl80211_id, 0, 0, NL80211_CMD_NEW_KEY, 0);

nla_put(msg, NL80211_ATTR_KEY_DATA, key_len, pairwise_key);
nla_put_u8(msg, NL80211_ATTR_KEY_CIPHER, WLAN_CIPHER_SUITE_CCMP);
nla_put_u8(msg, NL80211_ATTR_KEY_IDX, 0);

nl_send_auto(socket, msg);
```

**cfg80211 → mac80211 → ath12k:**

```c
// ath12k: drivers/net/wireless/ath/ath12k/mac.c
static int ath12k_mac_op_set_key(struct ieee80211_hw *hw,
                                enum set_key_cmd cmd,
                                struct ieee80211_vif *vif,
                                struct ieee80211_sta *sta,
                                struct ieee80211_key_conf *key)
{
    struct ath12k *ar = hw->priv;
    
    if (cmd == SET_KEY) {
        // Install key in hardware/firmware
        ret = ath12k_wmi_vdev_install_key(ar, key);
        
        // For CCMP (AES), hardware will encrypt/decrypt
        key->flags |= IEEE80211_KEY_FLAG_HW_MGMT_TX;
    }
    
    return ret;
}
```

**Firmware now has encryption keys! All data will be encrypted.**

---

## Phase 5: DHCP (Get IP Address)

### Step 5.1: DHCP Discovery

**Now wpa_supplicant notifies system: "Connected!"**

```c
// wpa_supplicant sends event
wpa_msg(wpa_s, MSG_INFO, WPA_EVENT_CONNECTED);
```

**ConnMan (or NetworkManager) receives this and starts DHCP:**

```c
// ConnMan: src/dhcp.c
int __connman_dhcp_start(struct connman_ipconfig *ipconfig,
                        struct connman_network *network,
                        dhcp_cb callback, gpointer user_data)
{
    // Start DHCP client (gdhcp)
    dhcp_client = g_dhcp_client_new(G_DHCP_IPV4, index, &error);
    
    g_dhcp_client_start(dhcp_client, last_address);
}
```

**DHCP packets go through regular data path:**
```
DHCP DISCOVER
  ↓ Socket
  ↓ Network stack
  ↓ mac80211 (encrypted with CCMP now!)
  ↓ ath12k TX
  ↓ Over the air (encrypted 802.11 data frame)
  ↓ AP decrypts and forwards to DHCP server
  ↓ DHCP OFFER comes back
  ↓ ath12k RX
  ↓ mac80211 (decrypted)
  ↓ Network stack
  ↓ DHCP client receives
```

**After DHCP completes:**
- IP address: 192.168.1.100
- Gateway: 192.168.1.1
- DNS: 192.168.1.1

---

## Phase 6: Data Transfer

### Step 6.1: Normal Data Flow

**Application wants to fetch google.com:**

```
Application: curl https://google.com
  ↓ Socket API (TCP)
  ↓ TCP/IP stack
  ↓ Routing (via 192.168.1.1)
  ↓ ARP (find gateway MAC)
  ↓ Create Ethernet frame
  ↓ Convert to 802.11 frame
  ↓ mac80211
  ↓ Encrypt with CCMP (hardware)
  ↓ ath12k TX path
  ↓ DMA to WiFi chip
  ↓ Firmware adds PHY header
  ↓ Transmit over the air
  ↓ AP receives, decrypts, forwards to internet
```

**ath12k TX for data:**

```c
// ath12k: drivers/net/wireless/ath/ath12k/dp_tx.c
int ath12k_dp_tx(struct ath12k *ar, struct sk_buff *skb)
{
    struct ath12k_base *ab = ar->ab;
    struct hal_tx_desc *desc;
    
    // Get TX descriptor from ring
    desc = ath12k_hal_tx_desc_get(ab->hal.tcl_data_ring);
    
    // Map packet for DMA
    dma_addr_t paddr = dma_map_single(ab->dev, skb->data, skb->len);
    
    // Fill descriptor with:
    desc->buf_addr = paddr;
    desc->buf_len = skb->len;
    
    // Rate information (WiFi 6/7 specific)
    desc->rate_info = calculate_rate_info(ar, skb);
    
    // For WiFi 7 (802.11be):
    if (ar->ab->hw_params.is_wifi7) {
        // Use 320 MHz bandwidth if available
        desc->bw = HAL_BW_320;
        // Use 4096-QAM if conditions good
        desc->mcs = HAL_MCS_14;
    }
    
    // Encryption flag (hardware will encrypt)
    desc->encrypt_type = HAL_ENCRYPT_TYPE_CCMP_128;
    
    // Ring doorbell (tell hardware about new packet)
    ath12k_hal_tx_ring_doorbell(ab);
}
```

**Firmware transmits 802.11 frame:**
```
┌───────────────────────────────────────────┐
│ 802.11 Data Frame (Encrypted)             │
├───────────────────────────────────────────┤
│ Frame Control: Data, Protected            │
│ To: aa:bb:cc:dd:ee:ff (AP)               │
│ From: 00:11:22:33:44:55 (us)             │
│ Sequence: 1234                            │
│ Encrypted Payload:                        │
│   [CCMP Header]                           │
│   [Encrypted IP packet to google.com]     │
│   [MIC for integrity]                     │
└───────────────────────────────────────────┘
```

---

# Complete Linux WiFi Connection Flow - End to End

## The Big Picture: All Layers

```
┌──────────────────────────────────────────────────────────────────┐
│                    PHASE 1: SCANNING                             │
│  wpa_supplicant → nl80211 → cfg80211 → mac80211 → ath12k        │
│  → Firmware scans all channels → Returns scan results           │
│  Duration: ~3-5 seconds                                          │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│              PHASE 2: AUTHENTICATION (802.11)                    │
│  wpa_supplicant → nl80211 → cfg80211 → mac80211                  │
│  → ath12k transmits Authentication frame                         │
│  → AP responds with success                                      │
│  Duration: ~10-50 ms                                             │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│               PHASE 3: ASSOCIATION (802.11)                      │
│  wpa_supplicant → nl80211 → cfg80211 → mac80211                  │
│  → ath12k transmits Association Request                          │
│  → AP assigns AID, responds with success                         │
│  Duration: ~10-50 ms                                             │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│            PHASE 4: WPA2 4-WAY HANDSHAKE                         │
│  wpa_supplicant ↔ AP (EAPOL frames)                             │
│  1. AP sends ANonce                                              │
│  2. Station sends SNonce + MIC                                   │
│  3. AP sends GTK + MIC                                           │
│  4. Station acknowledges                                         │
│  → Keys installed in ath12k hardware                             │
│  Duration: ~50-200 ms                                            │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                  PHASE 5: DHCP (Get IP)                          │
│  ConnMan/NetworkManager starts DHCP client                       │
│  DHCP packets → encrypted by ath12k → sent over WiFi            │
│  Receives IP: 192.168.1.100                                      │
│  Duration: ~500-2000 ms                                          │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                 PHASE 6: ONLINE CHECK                            │
│  ConnMan: HTTP GET to ipv4.connman.net                          │
│  Verifies internet connectivity                                  │
│  Duration: ~100-500 ms                                           │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    CONNECTED & ONLINE!                           │
│  Total time: ~5-10 seconds                                       │
│  Now ready for normal data transfer                              │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

### Layer Responsibilities

**1. wpa_supplicant/ConnMan (Userspace)**
- Decides which network to connect to
- Handles WPA2/WPA3 encryption handshake
- Manages credentials
- Starts DHCP

**2. cfg80211 (Kernel)**
- Validates all requests
- Manages regulatory rules
- Tracks scan results
- Coordinate between userspace and drivers

**3. mac80211 (Kernel)**
- Implements 802.11 protocol
- Builds/parses 802.11 frames
- Rate control
- Power save
- Frame aggregation

**4. ath12k Driver (Kernel)**
- Hardware-specific code
- Communicates with firmware
- Manages DMA rings
- TX/RX data path
- Configures WiFi chip

**5. Firmware (On WiFi Chip)**
- Low-level MAC
- PHY control
- Beamforming
- Channel estimation
- Actual radio control

**6. Hardware (WiFi Chip)**
- RF frontend
- Antennas
- Radio transmission

---

## Critical Paths You Should Understand

### For ath Driver Development:

**1. TX Path:**
```
Application
  → TCP/IP stack
  → mac80211: ieee80211_tx()
  → ath12k: ath12k_mac_op_tx()
  → ath12k: ath12k_dp_tx()
  → DMA descriptor filled
  → Hardware doorbell
  → Firmware transmits
```

**2. RX Path:**
```
Hardware receives
  → DMA to memory
  → Interrupt
  → ath12k: ath12k_dp_rx_process()
  → Parse descriptor
  → Build skb
  → mac80211: ieee80211_rx()
  → Network stack
  → Application
```

**3. Management Path:**
```
Userspace command
  → nl80211 netlink
  → cfg80211 validation
  → mac80211 processing
  → ath12k: ath12k_mac_op_*()
  → WMI command to firmware
  → Firmware executes
  → WMI event back
  → ath12k processes event
  → mac80211 notification
  → cfg80211 notification
  → nl80211 event to userspace
```

---

## Next Steps for You

Now that you understand the complete flow, you can:

1. **Trace specific scenarios** through the code
2. **Add debug prints** at each layer
3. **Understand where ath12k fits** in the big picture
4. **Know what to modify** for new features

**Want me to:**
- [ ] Deep dive into TX path with actual ath12k code
- [ ] Deep dive into RX path with actual ath12k code
- [ ] Explain WMI (firmware communication) in detail
- [ ] Show you how to trace a complete connection with debug logs

**Which would be most helpful?** 🚀─────────────────────────────────────┐
│  USER SPACE                                             │
├─────────────────────────────────────────────────────────┤
│  [1] NetworkManager / ConnMan / wpa_supplicant          │
│      "I want to connect to 'HomeWiFi'"                  │
└──────────────────────┬──────────────────────────────────┘
                       │ nl80211 (netlink)
                       │ (User ↔ Kernel communication)
                       ▼
┌─────────────────────────────────────────────────────────┐
│  KERNEL SPACE                                           │
├─────────────────────────────────────────────────────────┤
│  [2] cfg80211                                           │
│      "Central wireless configuration layer"             │
│      - Validates requests                               │
│      - Manages regulatory                               │
│      - Tracks scan results                              │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  [3] mac80211                                           │
│      "Software MAC layer implementation"                │
│      - Handles 802.11 frames                            │
│      - Authentication/Association                       │
│      - Encryption (if in software)                      │
│      - Rate control                                     │
│      - Power management                                 │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  [4] ath12k / ath11k / ath10k Driver                   │
│      "Hardware-specific driver"                         │
│      - Talks to specific WiFi chip                      │
│      - Manages TX/RX rings                              │
│      - Configures hardware registers                    │
│      - DMA operations                                   │
└──────────────────────┬──────────────────────────────────┘
                       │ PCI / USB / SDIO
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  [5] WiFi Chip Hardware                                 │
│      ├─ Firmware (running on chip)                      │
│      │  - Low-level MAC                                 │
│      │  - PHY control                                   │
│      │  - Beamforming                                   │
│      └─ Hardware                                        │
│         - RF frontend                                   │
│         - Antennas                                      │
│         - Actual radio transmission                     │
└─────────────────────────────────────────────────────────┘
                       │ Radio waves
                       ▼
                   [6] Access Point (WiFi Router)
```

---

## Complete Connection Flow: Step by Step

### Scenario: Connect to "HomeWiFi" WPA2 network

Let me trace this through **all layers** with actual code paths.

---

## Phase 1: User Initiates Connection

### Step 1.1: User Command

```bash
# Option A: Using wpa_supplicant directly
wpa_supplicant -i wlan0 -c /etc/wpa_supplicant.conf

# Option B: Using ConnMan
connmanctl connect wifi_xxxxx_HomeWiFi_managed_psk

# Option C: Using NetworkManager
nmcli device wifi connect "HomeWiFi" password "mypassword"
```

**What happens:**
```
Application (wpa_supplicant / ConnMan / NetworkManager)
  │
  ├─ Read configuration
  ├─ Know: SSID = "HomeWiFi", Security = WPA2-PSK, Password = "xxx"
  └─ Need to: Scan → Authenticate → Associate → Get IP
```

---

## Phase 2: Scanning for Networks

### Step 2.1: Application Requests Scan (Userspace)

**wpa_supplicant/ConnMan sends:**
```c
// Userspace: wpa_supplicant or ConnMan
// Uses nl80211 netlink interface

struct nl_msg *msg;
msg = nlmsg_alloc();

// NL80211_CMD_TRIGGER_SCAN command
genlmsg_put(msg, 0, 0, nl80211_id, 0, 0, NL80211_CMD_TRIGGER_SCAN, 0);

// Specify interface
nla_put_u32(msg, NL80211_ATTR_IFINDEX, if_nametoindex("wlan0"));

// Send to kernel
nl_send_auto(socket, msg);
```

**nl80211 message structure:**
```
Netlink Message:
  Command: NL80211_CMD_TRIGGER_SCAN
  Attributes:
    - IFINDEX: 3 (wlan0)
    - SCAN_SSIDS: ["HomeWiFi"] (optional, can be broadcast)
    - SCAN_FREQUENCIES: [2412, 2437, 2462, ...] (optional)
```

---

### Step 2.2: cfg80211 Processes Scan Request (Kernel)

**File:** `net/wireless/scan.c`

```c
// cfg80211 receives nl80211 command
int cfg80211_trigger_scan(struct cfg80211_registered_device *rdev,
                         struct wireless_dev *wdev,
                         struct cfg80211_scan_request *request)
{
    // Validate request
    if (!rdev->ops->scan)
        return -EOPNOTSUPP;
    
    // Check regulatory domain
    if (!cfg80211_is_allowed(request->channels))
        return -EINVAL;
    
    // Call driver's scan function
    return rdev->ops->scan(&rdev->wiphy, request);
}
```

**cfg80211 validates:**
- Is interface UP?
- Is scan allowed in current country?
- Are requested channels legal?
- Is another scan already running?

---

### Step 2.3: mac80211 Prepares Scan (Kernel)

**File:** `net/mac80211/scan.c`

```c
// mac80211 scan implementation
static int __ieee80211_start_scan(struct ieee80211_sub_if_data *sdata,
                                 struct cfg80211_scan_request *req)
{
    struct ieee80211_local *local = sdata->local;
    
    // Switch to scanning state
    local->scanning |= SCAN_SW_SCANNING;
    
    // For each channel to scan
    for (i = 0; i < req->n_channels; i++) {
        struct ieee80211_channel *chan = req->channels[i];
        
        // Switch to this channel
        ieee80211_hw_config(local, IEEE80211_CONF_CHANGE_CHANNEL);
        
        // Send probe request
        ieee80211_send_probe_req(sdata, NULL, req->ssids[0].ssid, 
                                req->ssids[0].ssid_len);
        
        // Wait for probe responses (typically 100-200ms per channel)
        msleep(scan_dwell_time);
    }
    
    // Scan complete, notify cfg80211
    cfg80211_scan_done(req, &info);
}
```

**What mac80211 does:**
1. Switch radio to each channel (2.4 GHz and 5 GHz)
2. On each channel:
   - Send Probe Request frame
   - Listen for Probe Response frames
   - Wait ~100ms
3. Collect all responses
4. Return results to cfg80211

---

### Step 2.4: ath12k Driver Executes Scan (Kernel)

**File:** `drivers/net/wireless/ath/ath12k/mac.c`

```c
// ath12k implements mac80211 scan callback
static int ath12k_mac_op_hw_scan(struct ieee80211_hw *hw,
                                struct ieee80211_vif *vif,
                                struct ieee80211_scan_request *hw_req)
{
    struct ath12k *ar = hw->priv;
    struct scan_req_params arg;
    int ret;
    
    // Build scan parameters for firmware
    arg.scan_id = ar->scan.scan_id++;
    arg.n_channels = hw_req->req.n_channels;
    arg.dwell_time_active = 100;  // ms to listen on each channel
    arg.dwell_time_passive = 200;
    
    // Copy channel list
    for (i = 0; i < arg.n_channels; i++) {
        arg.channels[i] = hw_req->req.channels[i]->center_freq;
    }
    
    // Send scan command to firmware via WMI
    ret = ath12k_wmi_send_scan_start_cmd(ar, &arg);
    if (ret) {
        ath12k_warn(ar->ab, "failed to send scan start: %d\n", ret);
        return ret;
    }
    
    // Firmware will scan and send results back via events
    return 0;
}
```

**What ath12k does:**
1. Translates mac80211 scan request to firmware format
2. Sends WMI (Wireless Management Interface) command to firmware
3. Firmware takes over and performs actual scan
4. Driver waits for scan complete event from firmware

---

### Step 2.5: Firmware Scans (WiFi Chip)

**Inside Qualcomm firmware (black box, but we know what it does):**

```
For each channel in list:
  1. Switch radio to channel frequency
  2. Configure RF frontend
  3. Send Probe Request 802.11 frame:
     ┌──────────────────────────────────────┐
     │ Frame Control: Probe Request         │
     │ Duration: 0                          │
     │ DA: ff:ff:ff:ff:ff:ff (broadcast)    │
     │ SA: 00:11:22:33:44:55 (our MAC)      │
     │ BSSID: ff:ff:ff:ff:ff:ff             │
     │ SSID: "HomeWiFi"                     │
     │ Supported Rates: 6, 12, 24 Mbps...  │
     │ HT Capabilities (WiFi 4)             │
     │ VHT Capabilities (WiFi 5)            │
     │ HE Capabilities (WiFi 6)             │
     │ EHT Capabilities (WiFi 7)            │
     └──────────────────────────────────────┘
  
  4. Listen for Probe Response frames
  5. Collect beacon frames
  6. Store RSSI, channel, capabilities
  7. Move to next channel
  
After all channels scanned:
  8. Send scan complete event to driver
```

**Probe Response from AP looks like:**
```
┌──────────────────────────────────────┐
│ Frame Control: Probe Response        │
│ SA: aa:bb:cc:dd:ee:ff (AP's MAC)     │
│ SSID: "HomeWiFi"                     │
│ Supported Rates: 6-540 Mbps          │
│ Channel: 36 (5 GHz)                  │
│ RSSI: -45 dBm                        │
│ Capabilities:                        │
│   - WPA2-PSK encryption              │
│   - 802.11ax (WiFi 6)                │
│   - 80 MHz bandwidth                 │
│ RSN (Security) Information:          │
│   - Cipher: CCMP (AES)               │
│   - Authentication: PSK              │
└──────────────────────────────────────┘
```

---

### Step 2.6: Scan Results Bubble Up

**Firmware → Driver → mac80211 → cfg80211 → Userspace**

```c
// ath12k receives scan complete event from firmware
static void ath12k_wmi_event_scan_complete(struct ath12k *ar,
                                          struct sk_buff *skb)
{
    ath12k_dbg(ar->ab, ATH12K_DBG_WMI, "scan complete event\n");
    
    // Notify mac80211 that scan is done
    ieee80211_scan_completed(ar->hw, &info);
}

// mac80211 notifies cfg80211
void ieee80211_scan_completed(struct ieee80211_hw *hw,
                             struct cfg80211_scan_info *info)
{
    struct ieee80211_local *local = hw_to_local(hw);
    
    local->scanning = 0;
    
    // Tell cfg80211
    cfg80211_scan_done(local->scan_req, info);
}

// cfg80211 notifies userspace via nl80211
void cfg80211_scan_done(struct cfg80211_scan_request *request,
                       struct cfg80211_scan_info *info)
{
    // Send NL80211_CMD_NEW_SCAN_RESULTS to userspace
    nl80211_send_scan_result(request->wiphy, NL80211_CMD_NEW_SCAN_RESULTS);
}
```

**Userspace receives:**
```
Netlink Event:
  Command: NL80211_CMD_NEW_SCAN_RESULTS
  
  BSS 1:
    SSID: "HomeWiFi"
    BSSID: aa:bb:cc:dd:ee:ff
    Channel: 36 (5180 MHz)
    Signal: -45 dBm
    Capabilities: ESS, Privacy
    RSN: WPA2-PSK
    
  BSS 2:
    SSID: "Neighbor_WiFi"
    ...
```

**wpa_supplicant/ConnMan now has scan results!**

---

## Phase 3: Authentication and Association

### Step 3.1: wpa_supplicant Selects Network

```c
// wpa_supplicant code (userspace)
void wpa_supplicant_select_network(struct wpa_supplicant *wpa_s,
                                  struct wpa_ssid *ssid)
{
    // Found "HomeWiFi" in scan results
    // It's WPA2-PSK
    // We have the password
    
    // Step 1: Authenticate (802.11 authentication)
    wpa_supplicant_authenticate(wpa_s, ssid);
}
```

---

### Step 3.2: 802.11 Authentication (Open System)

**wpa_supplicant sends authentication via nl80211:**

```c
// nl80211 command: NL80211_CMD_AUTHENTICATE
struct nl_msg *msg = nlmsg_alloc();
genlmsg_put(msg, 0, 0, nl80211_id, 0, 0, NL80211_CMD_AUTHENTICATE, 0);

nla_put_u32(msg, NL80211_ATTR_IFINDEX, ifindex);
nla_put(msg, NL80211_ATTR_MAC, ETH_ALEN, ap_bssid);  // AP's MAC
nla_put_u32(msg, NL80211_ATTR_AUTH_TYPE, NL80211_AUTHTYPE_OPEN_SYSTEM);
nla_put(msg, NL80211_ATTR_SSID, ssid_len, ssid);

nl_send_auto(socket, msg);
```

**cfg80211 → mac80211 → ath12k:**

```c
// mac80211: net/mac80211/mlme.c
static int ieee80211_auth(struct ieee80211_sub_if_data *sdata)
{
    struct ieee80211_local *local = sdata->local;
    struct ieee80211_mgmt *mgmt;
    struct sk_buff *skb;
    
    // Build Authentication frame (802.11 management frame)
    skb = alloc_skb(sizeof(*mgmt) + extra, GFP_KERNEL);
    mgmt = (struct ieee80211_mgmt *)skb_put(skb, sizeof(*mgmt));
    
    mgmt->frame_control = cpu_to_le16(IEEE80211_FTYPE_MGMT |
                                      IEEE80211_STYPE_AUTH);
    mgmt->duration = 0;
    memcpy(mgmt->da, ap_addr, ETH_ALEN);       // Destination: AP
    memcpy(mgmt->sa, sdata->vif.addr, ETH_ALEN); // Source: Our MAC
    memcpy(mgmt->bssid, ap_addr, ETH_ALEN);    // BSSID: AP
    
    // Authentication algorithm: Open System
    mgmt->u.auth.auth_alg = cpu_to_le16(WLAN_AUTH_OPEN);
    mgmt->u.auth.auth_transaction = cpu_to_le16(1);  // Transaction 1
    mgmt->u.auth.status_code = 0;
    
    // Send via driver
    ieee80211_tx_skb(sdata, skb);
}
```

**ath12k transmits:**

```c
// ath12k: drivers/net/wireless/ath/ath12k/mac.c
static void ath12k_mac_op_tx(struct ieee80211_hw *hw,
                            struct ieee80211_tx_control *control,
                            struct sk_buff *skb)
{
    struct ath12k *ar = hw->priv;
    
    // Management frames (like Authentication) use special queue
    if (ieee80211_is_mgmt(hdr->frame_control)) {
        // Send via management TX queue
        ath12k_dp_tx_send_mgmt_frame(ar, skb);
    }
}

static int ath12k_dp_tx_send_mgmt_frame(struct ath12k *ar,
                                       struct sk_buff *skb)
{
    // Get TX descriptor
    struct hal_tx_desc *desc = ath12k_hal_tx_desc_get();
    
    // DMA map the frame
    dma_addr_t paddr = dma_map_single(ar->ab->dev, skb->data, skb->len);
    
    // Fill descriptor
    desc->buf_addr = paddr;
    desc->buf_len = skb->len;
    desc->info0 = FIELD_PREP(TX_MGMT_DESC, 1);  // Management frame
    
    // Kick hardware
    ath12k_hal_tx_ring_doorbell(ar->ab, HAL_TCL_MGMT_RING);
    
    // Firmware will transmit this 802.11 management frame
}
```

**Over the air (802.11 Authentication frame):**
```
┌────────────────────────────────────────┐
│ Frame Type: Management                 │
│ Subtype: Authentication                │
│ To: aa:bb:cc:dd:ee:ff (AP)            │
│ From: 00:11:22:33:44:55 (our station) │
│ Algorithm: Open System                 │
│ Transaction Sequence: 1                │
│ Status: Reserved                       │
└────────────────────────────────────────┘
```

**AP responds:**
```
┌────────────────────────────────────────┐
│ Frame Type: Management                 │
│ Subtype: Authentication                │
│ To: 00:11:22:33:44:55 (our station)   │
│ From: aa:bb:cc:dd:ee:ff (AP)          │
│ Transaction Sequence: 2                │
│ Status: Success (0x0000)               │
└────────────────────────────────────────┘
```

**ath12k receives response:**

```c
// ath12k RX path: drivers/net/wireless/ath/ath12k/dp_rx.c
static void ath12k_dp_rx_process_received_packets(struct ath12k *ar)
{
    struct sk_buff *skb;
    struct ieee80211_hdr *hdr;
    
    // Get packet from RX ring
    skb = ath12k_dp_rx_get_skb_from_ring(ar);
    
    hdr = (struct ieee80211_hdr *)skb->data;
    
    if (ieee80211_is_auth(hdr->frame_control)) {
        // This is authentication response
        ath12k_dbg(ar->ab, ATH12K_DBG_MAC, "RX authentication frame\n");
    }
    
    // Pass to mac80211
    ieee80211_rx_napi(ar->hw, NULL, skb, napi);
}
```

**mac80211 processes:**

```c
// mac80211: net/mac80211/mlme.c
static void ieee80211_rx_mgmt_auth(struct ieee80211_sub_if_data *sdata,
                                  struct ieee80211_mgmt *mgmt, size_t len)
{
    u16 status = le16_to_cpu(mgmt->u.auth.status_code);
    
    if (status == WLAN_STATUS_SUCCESS) {
        // Authentication successful!
        sdata->u.mgd.auth_data->status = status;
        
        // Now proceed to Association
        ieee80211_assoc(sdata);
    }
}
```

---

### Step 3.3: 802.11 Association

**Similar flow, but Association Request frame:**

```
mac80211 sends Association Request
  ↓
ath12k transmits over the air
  ↓
┌────────────────────────────────────────┐
│ Frame Type: Management                 │
│ Subtype: Association Request           │
│ To: aa:bb:cc:dd:ee:ff (AP)            │
│ Capabilities: ESS, Privacy             │
│ Listen Interval: 10                    │
│ SSID: "HomeWiFi"                       │
│ Supported Rates: 6, 9, 12, 18...      │
│ HT Capabilities                        │
│ VHT Capabilities                       │
│ HE Capabilities (WiFi 6)               │
│ EHT Capabilities (WiFi 7 if supported) │
│ RSN (Security):                        │
│   - Cipher: CCMP (AES)                 │
│   - AKM: PSK                           │
└────────────────────────────────────────┘
```

**AP responds:**
```
┌────────────────────────────────────────┐
│ Frame Type: Management                 │
│ Subtype: Association Response          │
│ Status: Success (0x0000)               │
│ AID: 1 (Association ID)                │
│ Supported Rates                        │
│ HT Operation                           │
│ VHT Operation                          │
│ HE Operation (WiFi 6)                  │
└────────────────────────────────────────┘
```

**Now we're associated! But not encrypted yet...**

---

## Phase 4: WPA2 4-Way Handshake

### Step 4.1: Key Exchange (EAPOL Frames)

**This happens in wpa_supplicant (userspace):**

```
Station (us)                              AP
    │                                     │
    │◄──── EAPOL Message 1 ───────────────┤
    │      (ANonce from AP)               │
    │                                     │
    ├───── EAPOL Message 2 ──────────────►│
    │      (SNonce from Station, MIC)     │
    │                                     │
    │◄──── EAPOL Message 3 ───────────────┤
    │      (GTK, MIC)                     │
    │                                     │
    ├───── EAPOL Message 4 ──────────────►│
    │      (Acknowledgment)               │
    │                                     │
    └──────── Keys Installed ──────────────┘
```

**EAPOL frames go through normal data path:**
- wpa_supplicant creates EAPOL frame
- Sent via regular socket
- Goes through network stack → mac80211 → ath12k
- Transmitted as data frames (not management)

**After handshake:**
```c
// wpa_supplicant installs keys via nl80211
struct nl_msg *msg = nlmsg_alloc();
genlmsg_put(msg, 0, 0, nl80211_id, 0, 0, NL80211_CMD_NEW_KEY, 0);

nla_put(msg, NL80211_ATTR_KEY_DATA, key_len, pairwise_key);
nla_put_u8(msg, NL80211_ATTR_KEY_CIPHER, WLAN_CIPHER_SUITE_CCMP);
nla_put_u8(msg, NL80211_ATTR_KEY_IDX, 0);

nl_send_auto(socket, msg);
```

**cfg80211 → mac80211 → ath12k:**

```c
// ath12k: drivers/net/wireless/ath/ath12k/mac.c
static int ath12k_mac_op_set_key(struct ieee80211_hw *hw,
                                enum set_key_cmd cmd,
                                struct ieee80211_vif *vif,
                                struct ieee80211_sta *sta,
                                struct ieee80211_key_conf *key)
{
    struct ath12k *ar = hw->priv;
    
    if (cmd == SET_KEY) {
        // Install key in hardware/firmware
        ret = ath12k_wmi_vdev_install_key(ar, key);
        
        // For CCMP (AES), hardware will encrypt/decrypt
        key->flags |= IEEE80211_KEY_FLAG_HW_MGMT_TX;
    }
    
    return ret;
}
```

**Firmware now has encryption keys! All data will be encrypted.**

---

## Phase 5: DHCP (Get IP Address)

### Step 5.1: DHCP Discovery

**Now wpa_supplicant notifies system: "Connected!"**

```c
// wpa_supplicant sends event
wpa_msg(wpa_s, MSG_INFO, WPA_EVENT_CONNECTED);
```

**ConnMan (or NetworkManager) receives this and starts DHCP:**

```c
// ConnMan: src/dhcp.c
int __connman_dhcp_start(struct connman_ipconfig *ipconfig,
                        struct connman_network *network,
                        dhcp_cb callback, gpointer user_data)
{
    // Start DHCP client (gdhcp)
    dhcp_client = g_dhcp_client_new(G_DHCP_IPV4, index, &error);
    
    g_dhcp_client_start(dhcp_client, last_address);
}
```

**DHCP packets go through regular data path:**
```
DHCP DISCOVER
  ↓ Socket
  ↓ Network stack
  ↓ mac80211 (encrypted with CCMP now!)
  ↓ ath12k TX
  ↓ Over the air (encrypted 802.11 data frame)
  ↓ AP decrypts and forwards to DHCP server
  ↓ DHCP OFFER comes back
  ↓ ath12k RX
  ↓ mac80211 (decrypted)
  ↓ Network stack
  ↓ DHCP client receives
```

**After DHCP completes:**
- IP address: 192.168.1.100
- Gateway: 192.168.1.1
- DNS: 192.168.1.1

---

## Phase 6: Data Transfer

### Step 6.1: Normal Data Flow

**Application wants to fetch google.com:**

```
Application: curl https://google.com
  ↓ Socket API (TCP)
  ↓ TCP/IP stack
  ↓ Routing (via 192.168.1.1)
  ↓ ARP (find gateway MAC)
  ↓ Create Ethernet frame
  ↓ Convert to 802.11 frame
  ↓ mac80211
  ↓ Encrypt with CCMP (hardware)
  ↓ ath12k TX path
  ↓ DMA to WiFi chip
  ↓ Firmware adds PHY header
  ↓ Transmit over the air
  ↓ AP receives, decrypts, forwards to internet
```

**ath12k TX for data:**

```c
// ath12k: drivers/net/wireless/ath/ath12k/dp_tx.c
int ath12k_dp_tx(struct ath12k *ar, struct sk_buff *skb)
{
    struct ath12k_base *ab = ar->ab;
    struct hal_tx_desc *desc;
    
    // Get TX descriptor from ring
    desc = ath12k_hal_tx_desc_get(ab->hal.tcl_data_ring);
    
    // Map packet for DMA
    dma_addr_t paddr = dma_map_single(ab->dev, skb->data, skb->len);
    
    // Fill descriptor with:
    desc->buf_addr = paddr;
    desc->buf_len = skb->len;
    
    // Rate information (WiFi 6/7 specific)
    desc->rate_info = calculate_rate_info(ar, skb);
    
    // For WiFi 7 (802.11be):
    if (ar->ab->hw_params.is_wifi7) {
        // Use 320 MHz bandwidth if available
        desc->bw = HAL_BW_320;
        // Use 4096-QAM if conditions good
        desc->mcs = HAL_MCS_14;
    }
    
    // Encryption flag (hardware will encrypt)
    desc->encrypt_type = HAL_ENCRYPT_TYPE_CCMP_128;
    
    // Ring doorbell (tell hardware about new packet)
    ath12k_hal_tx_ring_doorbell(ab);
}
```

**Firmware transmits 802.11 frame:**
```
┌───────────────────────────────────────────┐
│ 802.11 Data Frame (Encrypted)             │
├───────────────────────────────────────────┤
│ Frame Control: Data, Protected            │
│ To: aa:bb:cc:dd:ee:ff (AP)               │
│ From: 00:11:22:33:44:55 (us)             │
│ Sequence: 1234                            │
│ Encrypted Payload:                        │
│   [CCMP Header]                           │
│   [Encrypted IP packet to google.com]     │
│   [MIC for integrity]                     │
└───────────────────────────────────────────┘
```

---
