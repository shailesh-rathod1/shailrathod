---
layout: post
title: "Connman wpa_supplicant wifi67"
date: 2026-01-18 9:00:00 +0530
categories: [connman]
tags: [connman,wlan]
description: "Connman wpa_supplicant wifi67 in  Depth"

---

# ConnMan + ath Driver Integration - WiFi 6/7 Enhancement Project

## Project Overview

**Goal:** Enhance ConnMan to better expose and utilize WiFi 6 (802.11ax) and WiFi 7 (802.11be) features available in ath11k/ath12k drivers.

**Why This is Valuable:**
- ✅ Real-world impact (improves actual user experience)
- ✅ Leverages your ConnMan knowledge
- ✅ Connects to cutting-edge WiFi 7 technology
- ✅ Demonstrates full-stack understanding
- ✅ Good for resume/interviews (shows breadth)
- ✅ Practical contribution (not just theoretical)

---

## The Gap: What's Missing Today

### Current State

**ConnMan knows:**
```
- Network SSID
- Signal strength (RSSI)
- Security type (WPA2, WPA3)
- Frequency (2.4/5 GHz)
- Connection speed (reported by driver)
```

**ConnMan does NOT expose:**
```
❌ WiFi 6/7 specific capabilities
❌ Channel width (20/40/80/160/320 MHz)
❌ MCS (Modulation and Coding Scheme) rates
❌ Spatial streams (MIMO configuration)
❌ BSS coloring status
❌ Target Wake Time (TWT) usage
❌ OFDMA vs MU-MIMO mode
❌ 6 GHz band information
```

**Why it matters:**
```
User connects to WiFi and sees:
  "Signal: Good"
  
But doesn't know:
  - Am I getting WiFi 6 speeds?
  - Is 160 MHz being used?
  - Could I get better with WiFi 7?
  - Why is battery draining (no TWT)?
```

---

## Architecture: How ConnMan Talks to ath Drivers

### Current Flow

```
┌─────────────────────────────────────────────────────┐
│  ConnMan (Connection Manager Daemon)                │
│  plugins/wifi.c                                     │
└───────────────────┬─────────────────────────────────┘
                    │ D-Bus
                    ▼
┌─────────────────────────────────────────────────────┐
│  wpa_supplicant                                     │
│  (WiFi authentication daemon)                       │
└───────────────────┬─────────────────────────────────┘
                    │ D-Bus (fi.w1.wpa_supplicant1)
                    ▼
┌─────────────────────────────────────────────────────┐
│  Kernel: cfg80211 / nl80211                         │
└───────────────────┬─────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│  Kernel: mac80211                                   │
└───────────────────┬─────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│  Kernel: ath11k / ath12k driver                     │
└─────────────────────────────────────────────────────┘
```

**The Problem:**
- WiFi 6/7 info exists in ath driver
- Gets passed to mac80211
- Available via nl80211
- But wpa_supplicant doesn't expose all of it
- So ConnMan can't show it to users

---

## Project Phases

### Phase 1: Information Exposure (2-3 days)

**Goal:** Make WiFi 6/7 information visible in ConnMan

#### Task 1.1: Extend wpa_supplicant D-Bus Interface

**File to modify:** `wpa_supplicant/dbus/dbus_new_handlers.c`

**Add new properties to Station interface:**

```c
// In wpa_supplicant source
static const struct wpa_dbus_property_desc wpas_dbus_bss_properties[] = {
    // Existing properties...
    { "Signal", WPAS_DBUS_NEW_IFACE_BSS, "n",
      wpas_dbus_getter_bss_signal, NULL },
    
    // NEW: Add WiFi 6/7 properties
    { "HECapabilities", WPAS_DBUS_NEW_IFACE_BSS, "ay",
      wpas_dbus_getter_bss_he_capabilities, NULL },
    { "EHTCapabilities", WPAS_DBUS_NEW_IFACE_BSS, "ay",
      wpas_dbus_getter_bss_eht_capabilities, NULL },
    { "ChannelWidth", WPAS_DBUS_NEW_IFACE_BSS, "u",
      wpas_dbus_getter_bss_channel_width, NULL },
    { "MaxTxRate", WPAS_DBUS_NEW_IFACE_BSS, "u",
      wpas_dbus_getter_bss_max_rate, NULL },
    { "WiFiGeneration", WPAS_DBUS_NEW_IFACE_BSS, "u",
      wpas_dbus_getter_wifi_generation, NULL },
    
    { NULL, NULL, NULL, NULL, NULL }
};

// Implementation
DBusMessage * wpas_dbus_getter_wifi_generation(
    DBusMessage *message, void *user_data)
{
    struct wpa_supplicant *wpa_s = user_data;
    struct wpa_bss *bss = wpa_s->current_bss;
    dbus_uint32_t generation = 0;
    
    if (!bss)
        return NULL;
    
    // Check for WiFi 7 (EHT)
    if (bss->eht_capab)
        generation = 7;
    // Check for WiFi 6 (HE)
    else if (bss->he_capab)
        generation = 6;
    // Check for WiFi 5 (VHT)
    else if (bss->vht_capab)
        generation = 5;
    // Check for WiFi 4 (HT)
    else if (bss->ht_capab)
        generation = 4;
    else
        generation = 0;  // Legacy
    
    return wpas_dbus_simple_property_getter(
        message, DBUS_TYPE_UINT32, &generation);
}

DBusMessage * wpas_dbus_getter_bss_channel_width(
    DBusMessage *message, void *user_data)
{
    struct wpa_supplicant *wpa_s = user_data;
    struct wpa_bss *bss = wpa_s->current_bss;
    dbus_uint32_t width = 20;  // Default
    
    if (!bss)
        return NULL;
    
    // Parse HE/VHT operation element
    if (bss->eht_operation) {
        // WiFi 7: Can be 320 MHz
        width = parse_eht_channel_width(bss->eht_operation);
    } else if (bss->he_operation) {
        // WiFi 6: Up to 160 MHz
        width = parse_he_channel_width(bss->he_operation);
    } else if (bss->vht_operation) {
        // WiFi 5: Up to 160 MHz
        width = parse_vht_channel_width(bss->vht_operation);
    }
    
    return wpas_dbus_simple_property_getter(
        message, DBUS_TYPE_UINT32, &width);
}
```

#### Task 1.2: Extend ConnMan WiFi Plugin

**File to modify:** `connman/plugins/wifi.c`

**Add structure to hold WiFi 6/7 info:**

```c
// In ConnMan source: plugins/wifi.c

struct wifi_data {
    char *identifier;
    struct connman_device *device;
    struct connman_network *network;
    
    // Existing fields...
    int rssi;
    
    // NEW: WiFi 6/7 information
    struct wifi6_7_info {
        unsigned int wifi_generation;  // 4, 5, 6, 7
        unsigned int channel_width;    // 20, 40, 80, 160, 320 MHz
        unsigned int max_tx_rate;      // Mbps
        bool he_capable;               // WiFi 6
        bool eht_capable;              // WiFi 7
        bool twt_supported;            // Target Wake Time
        bool ofdma_supported;          // OFDMA
        unsigned int spatial_streams;  // MIMO streams
        char band[16];                 // "2.4GHz", "5GHz", "6GHz"
    } wifi67;
};

// Function to retrieve WiFi 6/7 info from wpa_supplicant
static void wifi_update_wifi67_info(GSupplicantNetwork *supplicant_network,
                                    struct wifi_data *wifi)
{
    const unsigned char *he_capab, *eht_capab;
    unsigned int he_len, eht_len;
    
    // Get HE (WiFi 6) capabilities
    he_capab = g_supplicant_network_get_he_capabilities(
        supplicant_network, &he_len);
    
    if (he_capab && he_len > 0) {
        wifi->wifi67.he_capable = true;
        wifi->wifi67.wifi_generation = 6;
        
        // Parse HE capabilities
        parse_he_capabilities(he_capab, he_len, &wifi->wifi67);
    }
    
    // Get EHT (WiFi 7) capabilities
    eht_capab = g_supplicant_network_get_eht_capabilities(
        supplicant_network, &eht_len);
    
    if (eht_capab && eht_len > 0) {
        wifi->wifi67.eht_capable = true;
        wifi->wifi67.wifi_generation = 7;
        
        // Parse EHT capabilities
        parse_eht_capabilities(eht_capab, eht_len, &wifi->wifi67);
    }
    
    // Get channel width from current connection
    wifi->wifi67.channel_width = 
        g_supplicant_network_get_channel_width(supplicant_network);
    
    // Get max data rate
    wifi->wifi67.max_tx_rate = 
        calculate_max_rate(&wifi->wifi67);
}

static unsigned int calculate_max_rate(struct wifi6_7_info *info)
{
    unsigned int rate = 0;
    
    if (info->eht_capable) {
        // WiFi 7 with 320 MHz, 4096-QAM, 8 streams = 46 Gbps
        if (info->channel_width == 320 && info->spatial_streams >= 8)
            rate = 46080;  // Mbps
        else if (info->channel_width == 160)
            rate = 23040;
        else if (info->channel_width == 80)
            rate = 11520;
    } else if (info->he_capable) {
        // WiFi 6 with 160 MHz, 1024-QAM, 8 streams = 9.6 Gbps
        if (info->channel_width == 160 && info->spatial_streams >= 8)
            rate = 9608;
        else if (info->channel_width == 80)
            rate = 4804;
    }
    
    return rate;
}
```

#### Task 1.3: Expose via ConnMan D-Bus API

**File to modify:** `connman/src/service.c`

**Add new D-Bus properties:**

```c
// In ConnMan source: src/service.c

static void append_wifi67_properties(DBusMessageIter *dict,
                                     struct connman_service *service)
{
    struct connman_network *network;
    struct wifi_data *wifi;
    
    network = __connman_service_get_network(service);
    if (!network)
        return;
    
    wifi = connman_network_get_data(network);
    if (!wifi)
        return;
    
    // WiFi Generation (4, 5, 6, 7)
    connman_dbus_dict_append_basic(dict, "WiFiGeneration",
                                   DBUS_TYPE_UINT32,
                                   &wifi->wifi67.wifi_generation);
    
    // Channel Width (MHz)
    connman_dbus_dict_append_basic(dict, "ChannelWidth",
                                   DBUS_TYPE_UINT32,
                                   &wifi->wifi67.channel_width);
    
    // Maximum TX Rate (Mbps)
    connman_dbus_dict_append_basic(dict, "MaxTxRate",
                                   DBUS_TYPE_UINT32,
                                   &wifi->wifi67.max_tx_rate);
    
    // WiFi 6/7 Features
    connman_dbus_dict_append_basic(dict, "WiFi6Capable",
                                   DBUS_TYPE_BOOLEAN,
                                   &wifi->wifi67.he_capable);
    
    connman_dbus_dict_append_basic(dict, "WiFi7Capable",
                                   DBUS_TYPE_BOOLEAN,
                                   &wifi->wifi67.eht_capable);
    
    connman_dbus_dict_append_basic(dict, "TWTSupported",
                                   DBUS_TYPE_BOOLEAN,
                                   &wifi->wifi67.twt_supported);
    
    connman_dbus_dict_append_basic(dict, "OFDMASupported",
                                   DBUS_TYPE_BOOLEAN,
                                   &wifi->wifi67.ofdma_supported);
    
    // Spatial Streams
    connman_dbus_dict_append_basic(dict, "SpatialStreams",
                                   DBUS_TYPE_UINT32,
                                   &wifi->wifi67.spatial_streams);
    
    // Band
    connman_dbus_dict_append_basic(dict, "Band",
                                   DBUS_TYPE_STRING,
                                   &wifi->wifi67.band);
}

static void append_properties(DBusMessageIter *dict,
                             dbus_bool_t limited,
                             struct connman_service *service)
{
    // ... existing properties ...
    
    // Add WiFi 6/7 properties for WiFi services
    if (service->type == CONNMAN_SERVICE_TYPE_WIFI) {
        append_wifi67_properties(dict, service);
    }
}
```

#### Task 1.4: Update connmanctl to Display Info

**File to modify:** `connman/client/services.c`

```c
// In ConnMan client source

static void print_wifi67_info(DBusMessageIter *dict)
{
    dbus_uint32_t generation = 0;
    dbus_uint32_t width = 0;
    dbus_uint32_t rate = 0;
    dbus_bool_t wifi6 = FALSE;
    dbus_bool_t wifi7 = FALSE;
    
    // Get WiFi generation
    if (dbus_message_iter_get_uint32(dict, "WiFiGeneration", &generation)) {
        printf("  WiFi Standard: ");
        switch (generation) {
        case 7:
            printf("WiFi 7 (802.11be)\n");
            break;
        case 6:
            printf("WiFi 6 (802.11ax)\n");
            break;
        case 5:
            printf("WiFi 5 (802.11ac)\n");
            break;
        case 4:
            printf("WiFi 4 (802.11n)\n");
            break;
        default:
            printf("Legacy (802.11a/b/g)\n");
        }
    }
    
    // Get channel width
    if (dbus_message_iter_get_uint32(dict, "ChannelWidth", &width)) {
        printf("  Channel Width: %u MHz\n", width);
    }
    
    // Get max rate
    if (dbus_message_iter_get_uint32(dict, "MaxTxRate", &rate)) {
        if (rate >= 1000)
            printf("  Maximum Speed: %.1f Gbps\n", rate / 1000.0);
        else
            printf("  Maximum Speed: %u Mbps\n", rate);
    }
    
    // Get features
    if (dbus_message_iter_get_bool(dict, "TWTSupported", &value)) {
        if (value)
            printf("  Features: Target Wake Time (power saving)\n");
    }
    
    if (dbus_message_iter_get_bool(dict, "OFDMASupported", &value)) {
        if (value)
            printf("  Features: OFDMA (efficient multi-user)\n");
    }
}
```

**Example output:**
```bash
$ connmanctl services wifi_*

Service: HomeWiFi
  State: online
  WiFi Standard: WiFi 7 (802.11be)
  Channel Width: 320 MHz
  Maximum Speed: 46.0 Gbps
  Signal Strength: -42 dBm (Excellent)
  Band: 6 GHz
  Features: Target Wake Time, OFDMA
  Spatial Streams: 8x8 MIMO
```

---

### Phase 2: Real-Time Statistics (3-4 days)

**Goal:** Show live connection quality metrics

#### Task 2.1: Poll Current Connection Stats

**Add to ConnMan WiFi plugin:**

```c
// plugins/wifi.c

struct wifi_stats {
    unsigned int current_tx_rate;     // Current TX rate (Mbps)
    unsigned int current_rx_rate;     // Current RX rate
    unsigned int current_mcs;         // Current MCS index
    unsigned int current_bandwidth;   // Current BW (MHz)
    int current_rssi;                 // Current RSSI
    unsigned int tx_packets;          // TX packet count
    unsigned int rx_packets;          // RX packet count
    unsigned int tx_errors;           // TX errors
    unsigned int rx_errors;           // RX errors
    unsigned long long tx_bytes;      // Total TX bytes
    unsigned long long rx_bytes;      // Total RX bytes
};

// Function to get stats from nl80211
static int wifi_get_connection_stats(struct wifi_data *wifi,
                                     struct wifi_stats *stats)
{
    struct nl_msg *msg;
    int ret;
    
    // Create netlink message
    msg = nlmsg_alloc();
    genlmsg_put(msg, 0, 0, nl80211_id, 0, NLM_F_DUMP,
               NL80211_CMD_GET_STATION, 0);
    
    // Request station info
    nla_put_u32(msg, NL80211_ATTR_IFINDEX, wifi->ifindex);
    
    // Send and process response
    ret = nl_send_auto(nl_sock, msg);
    
    // Parse response to fill stats
    // (Response contains rate, MCS, bandwidth, etc.)
    
    return ret;
}

// Periodic update (every 2 seconds)
static gboolean wifi_stats_update(gpointer user_data)
{
    struct wifi_data *wifi = user_data;
    struct wifi_stats stats;
    
    wifi_get_connection_stats(wifi, &stats);
    
    // Update service properties
    update_service_stats(wifi->service, &stats);
    
    // Emit D-Bus PropertyChanged signal
    connman_dbus_property_changed_dict(
        connman_service_get_path(wifi->service),
        CONNMAN_SERVICE_INTERFACE,
        append_connection_stats, &stats);
    
    return TRUE;  // Continue timer
}
```

#### Task 2.2: Create Stats D-Bus Interface

```c
// src/service.c

static void append_connection_stats(DBusMessageIter *dict,
                                   void *user_data)
{
    struct wifi_stats *stats = user_data;
    
    connman_dbus_dict_append_basic(dict, "CurrentTxRate",
                                   DBUS_TYPE_UINT32,
                                   &stats->current_tx_rate);
    
    connman_dbus_dict_append_basic(dict, "CurrentRxRate",
                                   DBUS_TYPE_UINT32,
                                   &stats->current_rx_rate);
    
    connman_dbus_dict_append_basic(dict, "CurrentMCS",
                                   DBUS_TYPE_UINT32,
                                   &stats->current_mcs);
    
    connman_dbus_dict_append_basic(dict, "CurrentBandwidth",
                                   DBUS_TYPE_UINT32,
                                   &stats->current_bandwidth);
    
    connman_dbus_dict_append_basic(dict, "TxBytes",
                                   DBUS_TYPE_UINT64,
                                   &stats->tx_bytes);
    
    connman_dbus_dict_append_basic(dict, "RxBytes",
                                   DBUS_TYPE_UINT64,
                                   &stats->rx_bytes);
}
```

---

### Phase 3: Feature Control (2-3 days)

**Goal:** Allow enabling/disabling WiFi 6/7 features

#### Task 3.1: Add Configuration Options

**File:** `src/main.conf`

```ini
[WiFi6]
# Enable WiFi 6 features
Enable=true

# Target Wake Time (power saving)
EnableTWT=true

# Preferred channel widths (comma-separated)
PreferredBandwidths=160,80,40

# Maximum spatial streams
MaxSpatialStreams=4

[WiFi7]
# Enable WiFi 7 features (if hardware supports)
Enable=true

# Prefer 6 GHz band
Prefer6GHz=true

# Enable 320 MHz channels
Enable320MHz=true

# Multi-Link Operation
EnableMLO=false
```

#### Task 3.2: Implement Feature Toggle

```c
// plugins/wifi.c

static int wifi_configure_wifi67_features(struct wifi_data *wifi)
{
    struct nl_msg *msg;
    bool enable_twt;
    unsigned int max_bw;
    
    // Read configuration
    enable_twt = connman_setting_get_bool("WiFi6.EnableTWT");
    max_bw = connman_setting_get_uint("WiFi6.PreferredBandwidth");
    
    // Configure via nl80211
    msg = nlmsg_alloc();
    genlmsg_put(msg, 0, 0, nl80211_id, 0, 0,
               NL80211_CMD_SET_INTERFACE, 0);
    
    // Set TWT
    if (enable_twt)
        nla_put_flag(msg, NL80211_ATTR_TWT_RESPONDER);
    
    // Set bandwidth preference
    nla_put_u32(msg, NL80211_ATTR_CHANNEL_WIDTH, max_bw);
    
    nl_send_auto(nl_sock, msg);
    
    return 0;
}
```

---

### Phase 4: Debugging & Diagnostics (2 days)

**Goal:** Add debugging tools for WiFi 6/7

#### Task 4.1: Enhanced Debug Logging

```c
// plugins/wifi.c

static void wifi_debug_print_wifi67(struct wifi_data *wifi)
{
    DBG("WiFi 6/7 Information:");
    DBG("  Generation: WiFi %u", wifi->wifi67.wifi_generation);
    DBG("  Channel Width: %u MHz", wifi->wifi67.channel_width);
    DBG("  Max TX Rate: %u Mbps", wifi->wifi67.max_tx_rate);
    DBG("  HE Capable: %s", wifi->wifi67.he_capable ? "yes" : "no");
    DBG("  EHT Capable: %s", wifi->wifi67.eht_capable ? "yes" : "no");
    DBG("  TWT Supported: %s", wifi->wifi67.twt_supported ? "yes" : "no");
    DBG("  OFDMA Supported: %s", wifi->wifi67.ofdma_supported ? "yes" : "no");
    DBG("  Spatial Streams: %u", wifi->wifi67.spatial_streams);
    DBG("  Band: %s", wifi->wifi67.band);
}

// Enable with: connmand -d plugins/wifi.c
```

#### Task 4.2: Add WiFi 6/7 Test Tool

**Create:** `client/wifi67-test.c`

```c
// ConnMan client tool to test WiFi 6/7 features

int main(int argc, char *argv[])
{
    DBusConnection *conn;
    
    conn = dbus_bus_get(DBUS_BUS_SYSTEM, NULL);
    
    // Get WiFi 6/7 capabilities
    print_wifi67_capabilities(conn);
    
    // Run speed test at different bandwidths
    test_bandwidth(conn, 20);
    test_bandwidth(conn, 40);
    test_bandwidth(conn, 80);
    test_bandwidth(conn, 160);
    test_bandwidth(conn, 320);  // WiFi 7
    
    // Test TWT power savings
    test_twt_power_savings(conn);
    
    return 0;
}
```

---

## Practical Implementation Guide

### Step-by-Step Execution

**Week 1: Phase 1 - Information Exposure**

Day 1-2: Modify wpa_supplicant
```bash
git clone git://w1.fi/srv/git/hostap.git
cd hostap/wpa_supplicant
# Modify dbus/dbus_new_handlers.c
# Add new properties
make
sudo make install
```

Day 3-4: Modify ConnMan
```bash
cd ~/connman-1.45
# Modify plugins/wifi.c
# Add wifi67 structure
# Implement update functions
make
sudo make install
```

Day 5: Test
```bash
sudo connmand -n -d plugins/wifi.c
connmanctl services wifi_*
# Should show WiFi 6/7 info!
```

**Week 2: Phase 2 - Statistics**

Day 6-8: Implement stats collection
Day 9-10: Test and debug

**Week 3: Phase 3 & 4 - Features & Debugging**

Day 11-13: Feature toggles
Day 14-15: Debug tools and documentation

---

## Testing Strategy

### Test Environment Setup

**Hardware needed:**
- WiFi 6 or WiFi 7 capable router
- Linux laptop with ath11k/ath12k WiFi card
- Or: Qualcomm development board (QCN9274)

**Test scenarios:**
1. Connect to WiFi 6 AP, verify 160 MHz displayed
2. Connect to WiFi 7 AP, verify 320 MHz displayed
3. Monitor stats during file transfer
4. Toggle TWT, measure power consumption
5. Test on 2.4/5/6 GHz bands

---

## Expected Results

### Before Your Changes:
```bash
$ connmanctl services wifi_*
*AO HomeWiFi     wifi_abcd1234_HomeWiFi_managed_psk
    State = online
    Strength = 80
```

### After Your Changes:
```bash
$ connmanctl services wifi_*
*AO HomeWiFi     wifi_abcd1234_HomeWiFi_managed_psk
    State = online
    WiFi Standard = WiFi 7 (802.11be)
    Channel Width = 320 MHz
    Maximum Speed = 46.0 Gbps
    Current TX Rate = 23.5 Gbps
    Current RX Rate = 24.1 Gbps
    Signal Strength = -42 dBm (Excellent)
    Band = 6 GHz
    Spatial Streams = 8x8 MIMO
    Features = TWT, OFDMA, MU-MIMO
    TX Bytes = 1.2 GB
    RX Bytes = 5.4 GB
```

---

## Submission Plan

### Patch Series Structure

```
[PATCH 0/6] ConnMan: Add WiFi 6/7 feature support

[PATCH 1/6] wifi: Add WiFi generation detection
[PATCH 2/6] wifi: Expose channel width information
[PATCH 3/6] wifi: Add real-time connection statistics
[PATCH 4/6] wifi: Add WiFi 6/7 capability flags
[PATCH 5/6] client: Display WiFi 6/7 information in connmanctl
[PATCH 6/6] doc: Document new WiFi 6/7 properties
```

### Where to Submit

**ConnMan patches:**
```
To: connman@lists.linux.dev
Cc: daniel.wagner@siemens.com
```

**wpa_supplicant patches (if needed):**
```
To: hostap@lists.infradead.org
Cc: j@w1.fi (Jouni Malinen)
```

---

## Why This Project is Perfect

**1. Real Impact**
- Users actually benefit
- Shows WiFi 7 capability
- Helps with network debugging

**2. Full Stack**
- Touches kernel (nl80211)
- Modifies wpa_supplicant
- Enhances ConnMan
- Demonstrates breadth

**3. Manageable Scope**
- ~2-3 weeks part-time
- Clear deliverables
- Incremental progress

**4. Career Value**
- Shows WiFi 7 knowledge
- Demonstrates contribution skill
- Good for interviews
- Resume-worthy

**5. Learning**
- nl80211 netlink protocol
- D-Bus IPC
- WiFi 6/7 standards
- Open source process

---
