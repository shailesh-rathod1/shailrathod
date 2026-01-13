---
layout: post
title: "Network and Ipconfig layers"
date: 2026-01-13 9:00:00 +0530
categories: [connman]
tags: [connman,wlan]
description: "Network and Ipconfig layers"

---

# ConnMan IPConfig and Network Design Deep Dive

## Core Design Philosophy

Before diving into the details, let's understand the **separation of concerns** that drives this design:

```
Network Layer    = "What network am I connecting to?" (WiFi AP, Ethernet cable)
IPConfig Layer   = "What IP address should I use on this network?"
```

This separation allows:
- **Same network, different IP methods** (WiFi AP with DHCP or static IP)
- **Dual-stack support** (IPv4 + IPv6 independently)
- **IP failover** (DHCP fails → IPv4LL without reconnecting to network)

---

## Part 1: Understanding the Data Structures

### 1.1 The Network Structure

```c
struct connman_network {
    // Identity
    int refcount;
    enum connman_network_type type;        // WiFi, Ethernet, Cellular
    char *identifier;                      // Unique ID for this network
    char *name;                           // Human-readable name
    char *path;                           // D-Bus path
    
    // Physical Connection State
    bool available;                        // Network is in range
    bool connected;                        // Physical layer connected
    bool associating;                      // In process of connecting
    bool connecting;
    uint8_t strength;                     // Signal strength (WiFi)
    
    // Parent Device
    struct connman_device *device;         // Which device owns this network
    int index;                            // Linux interface index
    
    // Driver Interface
    struct connman_network_driver *driver; // Technology-specific driver
    void *driver_data;                    // Driver's private data
    
    // IP Configuration Management
    // (Note: These are accessed via device, not directly stored here)
    
    // DHCP State
    guint dhcp_timeout;                   // DHCP retry timer
    
    // ACD (Address Conflict Detection)
    struct acd_host *acd_host;            // For checking IP conflicts
    guint ipv4ll_timeout;                 // IPv4LL fallback timer
    
    // IPv6 Router Discovery
    int router_solicit_count;             // How many RS sent
    int router_solicit_refresh_count;     // RS for RDNSS refresh
    
    // Technology-Specific Data (Union-like structure)
    struct {
        void *ssid;
        int ssid_len;
        char *mode;
        char *security;
        char *passphrase;
        char *eap;
        // ... WiFi-specific fields
    } wifi;
};
```

**Key Design Points:**
1. **Network owns physical connection state** - It knows if you're connected to the AP
2. **Does NOT store IPConfig directly** - This is by design! IPConfigs are managed by the device
3. **Technology-specific data in substruct** - Clean separation (wifi, cellular would have different fields)

### 1.2 The IPDevice Structure (The Bridge)

This is the **crucial intermediate layer** that many people miss:

```c
struct connman_ipdevice {
    // Interface Identity
    int index;                            // Linux interface index (eth0 = 2, wlan0 = 3)
    unsigned short type;                  // ARPHRD_ETHER, etc.
    char *address;                        // MAC address
    
    // Interface State (from kernel)
    unsigned int flags;                   // IFF_UP, IFF_RUNNING, IFF_LOWER_UP
    uint16_t mtu;
    
    // Statistics (from RTNL)
    uint32_t rx_packets;
    uint32_t tx_packets;
    uint32_t rx_bytes;
    uint32_t tx_bytes;
    uint32_t rx_errors;
    uint32_t tx_errors;
    
    // IP Configuration Objects
    struct connman_ipconfig *config_ipv4;  // IPv4 configuration
    struct connman_ipconfig *config_ipv6;  // IPv6 configuration
    
    // Kernel Address List
    GSList *address_list;                  // Addresses kernel knows about
    
    // Gateway Information
    char *ipv4_gateway;
    char *ipv6_gateway;
    
    // IPv6 Settings
    bool ipv6_enabled;
    int ipv6_privacy;                      // Privacy extensions
    
    // Proxy Auto-Config
    char *pac;
};
```

**Why This Layer Exists:**
- **One IPDevice per Linux interface** (persistent across network changes)
- **Bridges kernel state to ConnMan's logic** (RTNL events → IPDevice → IPConfig)
- **Manages both IPv4 and IPv6** for the same interface
- **Persists when network changes** (WiFi roaming, Ethernet reconnect)

### 1.3 The IPConfig Structure

```c
struct connman_ipconfig {
    // Reference Management
    int refcount;
    int index;                            // Linux interface index
    
    // IP Version
    enum connman_ipconfig_type type;      // IPv4 or IPv6
    
    // Configuration Method
    enum connman_ipconfig_method method;  // DHCP, Manual, Auto, Fixed, Off
    
    // Address Information
    struct connman_ipaddress *address;    // Desired/configured address
    struct connman_ipaddress *system;     // What kernel actually has
    
    // Callbacks
    const struct connman_ipconfig_ops *ops;
    void *ops_data;                       // Usually points to service
    
    // IPv6 Privacy
    int ipv6_privacy_config;              // Privacy extension settings
    
    // DHCP History (for faster reconnection)
    char *last_dhcp_address;              // Last DHCP-assigned address
    char **last_dhcpv6_prefixes;          // DHCPv6 prefix delegation history
};

struct connman_ipaddress {
    int family;                           // AF_INET or AF_INET6
    unsigned char prefixlen;              // CIDR prefix length (24 = /24)
    char *local;                          // Local IP address
    char *peer;                           // Peer address (P2P, VPN)
    char *broadcast;                      // Broadcast address
    char *gateway;                        // Default gateway
    bool is_p2p;                          // Point-to-point link
};
```

**Key Design Points:**
1. **Two addresses: desired vs actual**
   - `address`: What we want to configure
   - `system`: What kernel actually has (may differ temporarily)
2. **Method determines behavior**
   - DHCP → Start DHCP client
   - Manual → Apply user-provided settings
   - Auto → SLAAC for IPv6, IPv4LL for IPv4
3. **Callbacks to service** - When IP state changes, notify owning service

---

## Part 2: Relationship and Data Flow

### 2.1 Ownership Hierarchy

```
Service (user-visible connection)
   ↓
Network (physical connection to AP/cable)
   ↓
Device (hardware interface)
   ↓
IPDevice (kernel interface state)
   ↓
IPConfig (IPv4)  +  IPConfig (IPv6)
```

**Storage Locations:**
```c
// Where IPConfigs live:
IPDevice.config_ipv4 → IPConfig
IPDevice.config_ipv6 → IPConfig

// How to get them:
network → device → ipdevice → ipconfig
service → network → device → ipdevice → ipconfig
```

### 2.2 Why Separate Network and IPConfig?

#### Scenario 1: Network Stays Same, IP Method Changes
```
User connects to "CoffeeShop WiFi" with DHCP
 → Network: connected=true
 → IPConfig: method=DHCP, address=192.168.1.100

User changes to Static IP
 → Network: connected=true (UNCHANGED!)
 → IPConfig: method=MANUAL, address=192.168.1.50 (CHANGED)
```
**Benefit:** Don't need to disconnect/reconnect physically

#### Scenario 2: DHCP Fails, Fallback to IPv4LL
```
DHCP timeout occurs
 → Network: connected=true (still connected to AP!)
 → IPConfig: method=AUTO, trigger IPv4LL
 → Generate 169.254.x.x address
 → Check for conflicts with ACD
```
**Benefit:** Stay connected physically while IP layer recovers

#### Scenario 3: Dual Stack Independence
```
IPv4 DHCP succeeds → Service state: READY (can communicate)
IPv6 SLAAC fails   → IPv6 IPConfig: failed
Service is still usable via IPv4!
```

### 2.3 Complete Connection Flow with IPConfig/Network

Let me trace a WiFi connection from start to finish:

```
1. USER INITIATES CONNECTION
   __connman_service_connect()
   
2. SERVICE → NETWORK
   __connman_network_connect(network)
   
3. NETWORK → DRIVER (WiFi plugin)
   driver->connect()  // e.g., wifi_network_connect()
   
4. DRIVER → WPA_SUPPLICANT
   Send D-Bus commands to wpa_supplicant
   
5. WPA_SUPPLICANT AUTHENTICATES
   4-way handshake, WPA2-PSK or 802.1X
   
6. DRIVER CALLBACK: Connected
   network->driver->connected(network)
   
7. UPDATE NETWORK STATE
   network->connected = true
   network->associating = false
   
8. TRIGGER IP CONFIGURATION
   set_configuration(network, CONNMAN_IPCONFIG_TYPE_ALL)
   
9. GET IPCONFIGS FROM DEVICE
   device = network->device
   ipdevice = find_ipdevice(device->index)
   ipconfig_ipv4 = ipdevice->config_ipv4
   ipconfig_ipv6 = ipdevice->config_ipv6
   
10. START IPv4 CONFIGURATION
    if (method == DHCP):
        __connman_dhcp_start(ipconfig_ipv4)
    elif (method == MANUAL):
        __connman_ipconfig_set_local(ipconfig_ipv4, user_address)
        start_acd(network)  // Check for conflicts
    elif (method == AUTO):
        start_ipv4ll(network)
        
11. DHCP STATE MACHINE (if DHCP)
    a. Send DHCPDISCOVER
    b. Receive DHCPOFFER
    c. Send DHCPREQUEST
    d. Receive DHCPACK
    e. Call dhcp_callback() with IP info
    
12. DHCP CALLBACK
    __connman_ipconfig_set_dhcp(ipconfig_ipv4, dhcp_address)
    start_acd(network)  // Verify address is not in use
    
13. ACD (Address Conflict Detection)
    Send ARP probe for our IP
    Wait for conflicts
    
14. ACD SUCCESS
    acd_host_ipv4_available()
    
15. APPLY IP CONFIGURATION
    __connman_ipconfig_address_add(ipconfig_ipv4)
        ↓
    Netlink: RTM_NEWADDR to kernel
    Kernel sets IP on interface
    
16. ADD GATEWAY
    __connman_ipconfig_gateway_add(ipconfig_ipv4)
        ↓
    Netlink: RTM_NEWROUTE to kernel
    
17. CONFIGURE DNS
    __connman_resolver_append(ipconfig_ipv4, dns_servers)
    
18. UPDATE SERVICE STATE
    __connman_service_ipconfig_indicate_state(service, READY, IPv4)
    
19. START ONLINE CHECK (if enabled)
    __connman_wispr_start(service, ...)
    HTTP GET to http://ipv4.connman.net/online/status.html
    
20. ONLINE CHECK SUCCESS
    service->state = ONLINE
    D-Bus: emit PropertyChanged signal
```

---

## Part 3: Critical Design Patterns

### 3.1 The IPDevice Hash Table

```c
static GHashTable *ipdevice_hash = NULL;  // Key: interface index

// Why hash table?
// - O(1) lookup by interface index
// - Frequent access from RTNL events (address/link changes)
// - Interface index is stable (doesn't change during lifetime)
```

**Usage:**
```c
// RTNL receives "interface wlan0 (index 3) got new address"
ipdevice = g_hash_table_lookup(ipdevice_hash, GINT_TO_POINTER(3));
// Now we can update ipdevice->config_ipv4->system address
```

### 3.2 Dual IPConfig Design

**Why separate IPv4 and IPv6 IPConfigs?**

1. **Different configuration methods:**
   - IPv4: DHCP (client-server)
   - IPv6: SLAAC (stateless autoconfiguration)

2. **Independent state machines:**
   ```
   IPv4: DHCP succeeds → READY
   IPv6: SLAAC fails → FAILURE
   Service can still function on IPv4!
   ```

3. **Different address structures:**
   - IPv4: Single address + netmask
   - IPv6: Multiple addresses (link-local + global + temporary)

4. **Different failure handling:**
   - IPv4 DHCP fails → Try IPv4LL
   - IPv6 SLAAC fails → Service still works if IPv4 succeeds

### 3.3 Address vs System Pattern

```c
struct connman_ipconfig {
    struct connman_ipaddress *address;  // What we WANT to configure
    struct connman_ipaddress *system;   // What kernel ACTUALLY has
};
```

**Why this split?**

**Scenario: Manual IP configuration**
```c
// User sets static IP
__connman_ipconfig_set_local(ipconfig, "192.168.1.50");
// This updates ipconfig->address->local

// But kernel still has old DHCP address!
// ipconfig->system->local = "192.168.1.100"

// Now apply the change
__connman_ipconfig_address_add(ipconfig);
// Send netlink message to kernel
// Wait for confirmation

// RTNL receives RTM_NEWADDR from kernel
rtnl_newaddr() {
    // Update system address to match
    ipconfig->system->local = "192.168.1.50"
}
```

**Benefits:**
- Detect configuration drift (someone else changed IP)
- Handle async kernel operations
- Rollback on failure

### 3.4 The Callback Pattern

IPConfig has callbacks to notify owner (usually Service) of state changes:

```c
struct connman_ipconfig_ops {
    void (*up)(struct connman_ipconfig *ipconfig, const char *ifname);
    void (*down)(struct connman_ipconfig *ipconfig, const char *ifname);
    void (*lower_up)(struct connman_ipconfig *ipconfig, const char *ifname);
    void (*lower_down)(struct connman_ipconfig *ipconfig, const char *ifname);
    void (*ip_bound)(struct connman_ipconfig *ipconfig, const char *ifname);
    void (*ip_release)(struct connman_ipconfig *ipconfig, const char *ifname);
    void (*route_set)(struct connman_ipconfig *ipconfig, const char *ifname);
    void (*route_unset)(struct connman_ipconfig *ipconfig, const char *ifname);
};
```

**Flow:**
```
IPConfig: DHCP succeeded, got IP address
   ↓
Call ipconfig->ops->ip_bound(ipconfig, "wlan0")
   ↓
Service: Receives callback
   ↓
Service: Update state to READY
   ↓
Service: Emit D-Bus PropertyChanged signal
```

---

## Part 4: Advanced Scenarios

### 4.1 WiFi Roaming

```
Connected to "Home_WiFi_AP1"
 → Network1: connected=true
 → IPConfig: 192.168.1.50 (from DHCP)

Signal weakens, move to "Home_WiFi_AP2" (same SSID, same network)
 → Network1: Still represents the logical WiFi network
 → Device: Changes BSS (Basic Service Set)
 → IPConfig: UNCHANGED! Same IP, no DHCP needed

If IP lease expires during roaming:
 → DHCP renew happens transparently
 → Network stays connected
```

### 4.2 DHCP Failure → IPv4LL Fallback

```c
static void dhcp_timeout(struct connman_network *network)
{
    // DHCP failed after retries
    
    // Check if IPv4LL fallback is allowed
    if (ipconfig->method == CONNMAN_IPCONFIG_METHOD_AUTO) {
        // Schedule IPv4LL with delay
        network->ipv4ll_timeout = g_timeout_add_seconds(
            DHCP_RETRY_TIMEOUT, 
            start_ipv4ll_ontimeout, 
            network
        );
    }
}

static int start_ipv4ll(struct connman_network *network)
{
    // Generate random 169.254.x.x address
    addr.s_addr = htonl(arp_random_ip());
    
    // Configure it in IPConfig
    __connman_ipconfig_set_local(ipconfig, address);
    
    // Start ACD to check for conflicts
    return start_acd(network);
}
```

**Note:** Network stays connected! Only IP layer changes.

### 4.3 Address Conflict Detection

```c
static int start_acd(struct connman_network *network)
{
    // Create ACD host if doesn't exist
    if (!network->acd_host) {
        network->acd_host = acd_host_new(
            network->index,
            hw_address
        );
        acd_host_set_callback(
            network->acd_host,
            acd_host_ipv4_available,  // Success callback
            acd_host_ipv4_lost,       // Conflict callback
            network
        );
    }
    
    // Start probing
    ip_address = __connman_ipconfig_get_local(ipconfig_ipv4);
    return acd_host_start(network->acd_host, ip_address);
}
```

**ACD States:**
```
1. Send ARP probe (sender IP = 0.0.0.0, target = our IP)
2. Wait for response (500ms typically)
3. If conflict:
     → acd_host_ipv4_conflict()
     → If DHCP: Decline and retry
     → If IPv4LL: Pick new random IP and retry
4. If no conflict:
     → acd_host_ipv4_available()
     → Apply IP address to kernel
```

### 4.4 IPv6 Router Solicitation

Network manages IPv6 autoconfiguration:

```c
// Send Router Solicitation to get Router Advertisement
static int send_router_solicitation(struct connman_network *network)
{
    // Construct RS packet
    // Send to ff02::2 (all routers multicast)
    
    network->router_solicit_count++;
    
    if (network->router_solicit_count < MAX_RTR_SOLICITATIONS) {
        // Schedule retry in 4 seconds
        g_timeout_add_seconds(
            RTR_SOLICITATION_INTERVAL,
            send_router_solicitation,
            network
        );
    }
}

// When RA received (via RTNL):
rtnl_newroute() {
    // Extract prefix, gateway, DNS
    // Update ipconfig_ipv6
    __connman_ipconfig_set_gateway(ipconfig_ipv6, gateway);
}
```

---

## Part 5: Design Trade-offs

### Why NOT Store IPConfig in Network?

**Current Design:**
```
Network → Device → IPDevice → IPConfig
```

**Alternative (NOT used):**
```
Network → IPConfig (directly)
```

**Why current design is better:**

1. **Interface persistence:** When you disconnect from WiFi and reconnect, the Network object might be destroyed and recreated, but the IPDevice (representing wlan0) persists. This allows:
   - Keeping IP configuration across reconnects
   - Reusing DHCP leases
   - Maintaining statistics

2. **Shared interface:** Multiple networks could use same interface (rarely, but possible with VLANs or bridge scenarios)

3. **Kernel alignment:** IPDevice maps 1:1 with kernel's network interface, making RTNL event handling clean

### Why Separate IPv4 and IPv6 IPConfigs?

**Alternative:** Single IPConfig with dual-stack support

**Why separation is better:**

1. **Independent methods:** IPv4 DHCP vs IPv6 SLAAC are fundamentally different
2. **Independent failure:** One can fail while other succeeds
3. **Simpler state machine:** Each IPConfig has clear, simple state
4. **Code reuse:** Same IPConfig code works for both, just different parameters

### Why Reference Counting?

Multiple components may reference an IPConfig:
- Service (owner)
- IPDevice (container)
- DHCP client (during configuration)
- Gateway manager (when adding routes)

Reference counting ensures safe cleanup when all references are released.

---

## Part 6: Key Takeaways

1. **Network = Physical Connection**
   - Connected to AP? Connected to Ethernet cable?
   - Technology-specific (WiFi, Ethernet, Cellular)

2. **IPConfig = Logical Addressing**
   - What IP address am I using?
   - How did I get it (DHCP, manual, auto)?
   - Independent of physical connection

3. **IPDevice = Kernel Bridge**
   - Represents Linux network interface (eth0, wlan0)
   - Persists across network changes
   - Owns both IPv4 and IPv6 IPConfigs

4. **Separation Enables:**
   - IP method changes without physical reconnect
   - DHCP fallback to IPv4LL seamlessly
   - Dual-stack with independent failure handling
   - Clean abstraction layers

5. **Data Flow:**
   ```
   User Action → Service → Network → Device → IPDevice → IPConfig → Kernel
   Kernel Event → RTNL → IPDevice → IPConfig → Service → D-Bus → User
   ```

This architecture is a great example of **separation of concerns** and **layered design** in systems programming!

