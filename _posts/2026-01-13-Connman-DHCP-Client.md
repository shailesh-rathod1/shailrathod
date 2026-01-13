---
layout: post
title: "How DHCP Works in ConnMan - Complete Technical Deep Dive"
date: 2026-01-13 9:00:00 +0530
categories: [connman]
tags: [connman,wlan]
description: "Complete Technical Deep Dive"

---

# How DHCP Works in ConnMan - Complete Technical Deep Dive

## The Core Answer

**ConnMan implements its own DHCP client in userspace (gdhcp library)**

- ✅ ConnMan sends/receives all DHCP packets
- ✅ Complete DHCP state machine runs in userspace
- ✅ Kernel only provides socket interface and applies final configuration
- ❌ Kernel does NOT handle DHCP protocol logic

---

## Part 1: Why Userspace DHCP?

### Design Decision: Userspace vs Kernel

**Alternative Approaches:**
1. **Kernel DHCP** - Some systems have DHCP in kernel (rare, mostly embedded)
2. **External DHCP client** - Run separate `dhclient` or `udhcpc` process
3. **Built-in library** - Embed DHCP in ConnMan (← ConnMan's choice)

### Why ConnMan Chose Built-in Userspace DHCP

**Advantages:**
1. **Tight Integration**
   - Direct control over DHCP timing
   - Immediate callback when lease obtained
   - Can coordinate DHCP with ACD (Address Conflict Detection)
   - No IPC overhead with external process

2. **Event Loop Integration**
   - DHCP state machine integrated with GMainLoop
   - Uses same epoll for all network events
   - Consistent timeout handling

3. **No External Dependencies**
   - Don't need to ship/maintain separate dhclient
   - Guaranteed compatibility
   - Smaller attack surface

4. **Flexibility**
   - Can customize DHCP behavior
   - Support vendor-specific options
   - Implement features like rapid commit

**Trade-offs:**
- ❌ Need to maintain DHCP code (security updates, RFC compliance)
- ❌ Potential for bugs in protocol implementation
- ✅ But: Full control over behavior and integration

---

## Part 2: The GDHCP Library Architecture

### 2.1 Core Components

ConnMan's DHCP implementation lives in `gdhcp/`:

```
gdhcp/
├── client.c      - DHCP client state machine
├── server.c      - DHCP server (for tethering)
├── common.c      - Packet parsing/building
├── ipv4ll.c      - IPv4 Link-Local (169.254.x.x)
└── gdhcp.h       - Public API
```

### 2.2 DHCP Client Structure

```c
struct _GDHCPClient {
    // Identity
    int ifindex;                    // Interface index (e.g., wlan0)
    char *interface;                // Interface name
    uint8_t mac_address[6];         // Hardware address
    
    // DHCP State
    ClientState state;              // Current DHCP state
    uint32_t xid;                   // Transaction ID (random)
    
    // IP Addressing
    uint32_t server_ip;             // DHCP server IP
    uint32_t requested_ip;          // IP we're requesting
    char *assigned_ip;              // IP we got
    
    // Lease Information
    uint32_t lease_seconds;         // Lease duration
    time_t start;                   // Lease start time
    
    // Socket Management
    ListenMode listen_mode;         // L2 (raw) or L3 (UDP)
    int listener_sockfd;            // Socket for receiving
    guint listener_watch;           // GLib event source
    
    // Timers
    guint timeout;                  // Current operation timeout
    guint t1_timeout;               // Renewal timer (T1)
    guint t2_timeout;               // Rebinding timer (T2)
    guint lease_timeout;            // Lease expiration
    
    // Retry Logic
    uint8_t retry_times;            // Retry counter
    uint8_t ack_retry_times;        // ACK retry counter
    
    // Options
    GList *require_list;            // Required DHCP options
    GList *request_list;            // Requested DHCP options
    GHashTable *code_value_hash;    // Received option values
    
    // Callbacks
    GDHCPClientEventFunc lease_available_cb;
    gpointer lease_available_data;
    GDHCPClientEventFunc lease_lost_cb;
    // ... more callbacks
};
```

**Key Points:**
1. **Two Listen Modes:**
   - L2 (raw packets): Before we have an IP
   - L3 (UDP): After we have an IP (for renewals)
   
2. **Three Timers:**
   - T1 (50% of lease): Try to renew with same server
   - T2 (87.5% of lease): Try to rebind with any server
   - Lease expiration: Must release and start over

### 2.3 DHCP States

```c
typedef enum _dhcp_client_state {
    INIT_SELECTING,   // Sending DISCOVER, waiting for OFFER
    REQUESTING,       // Sent REQUEST, waiting for ACK
    BOUND,           // Have valid lease
    RENEWING,        // Trying to renew (T1 expired)
    REBINDING,       // Trying to rebind (T2 expired)
    REBOOTING,       // Trying to reuse previous lease
    // ... IPv4LL states
    IPV4LL_PROBE,
    IPV4LL_ANNOUNCE,
    // ... IPv6 states  
    SOLICITATION,
    REQUEST,
    // ...
} ClientState;
```

---

## Part 3: Packet Construction and Transmission

### 3.1 How DHCP Packets Are Built

DHCP packets are constructed **entirely in userspace**:

```c
// From gdhcp/common.c
struct dhcp_packet {
    uint8_t op;          // BOOTREQUEST or BOOTREPLY
    uint8_t htype;       // Hardware type (Ethernet = 1)
    uint8_t hlen;        // Hardware address length (6 for MAC)
    uint8_t hops;        // Hop count
    uint32_t xid;        // Transaction ID
    uint16_t secs;       // Seconds since start
    uint16_t flags;      // Broadcast flag, etc.
    uint32_t ciaddr;     // Client IP (if known)
    uint32_t yiaddr;     // Your IP (from server)
    uint32_t siaddr;     // Server IP
    uint32_t giaddr;     // Gateway IP
    uint8_t chaddr[16];  // Client hardware address
    uint8_t sname[64];   // Server name
    uint8_t file[128];   // Boot file name
    uint32_t magic;      // DHCP magic cookie
    uint8_t options[308]; // DHCP options
} __attribute__((packed));
```

**Building a DHCP DISCOVER:**

```c
// Simplified from gdhcp/client.c
static int send_discover(GDHCPClient *dhcp_client)
{
    struct dhcp_packet packet;
    
    // 1. Initialize packet
    memset(&packet, 0, sizeof(packet));
    packet.op = BOOTREQUEST;
    packet.htype = ARPHRD_ETHER;
    packet.hlen = 6;
    packet.xid = dhcp_client->xid;
    packet.flags = htons(BROADCAST_FLAG);
    
    // 2. Copy MAC address
    memcpy(packet.chaddr, dhcp_client->mac_address, 6);
    
    // 3. Add DHCP magic cookie
    packet.magic = htonl(DHCP_MAGIC);
    
    // 4. Add DHCP options
    add_option(&packet, DHCP_MESSAGE_TYPE, DHCPDISCOVER);
    add_option(&packet, DHCP_REQUESTED_IP, requested_ip);
    add_option(&packet, DHCP_HOST_NAME, hostname);
    add_option(&packet, DHCP_PARAM_REQ, requested_options);
    
    // 5. Send via raw socket
    return dhcp_send_raw_packet(&packet, 
                                INADDR_ANY,        // Source: 0.0.0.0
                                CLIENT_PORT,       // Source port: 68
                                INADDR_BROADCAST,  // Dest: 255.255.255.255
                                SERVER_PORT,       // Dest port: 67
                                MAC_BCAST_ADDR,    // Dest MAC: ff:ff:ff:ff:ff:ff
                                dhcp_client->ifindex);
}
```

### 3.2 How Packets Are Actually Sent

#### Option 1: Raw Sockets (L2) - Used When No IP Address

```c
// From gdhcp/client.c
static int dhcp_send_raw_packet(struct dhcp_packet *dhcp_pkt,
                                uint32_t source_ip,
                                int source_port,
                                uint32_t dest_ip,
                                int dest_port,
                                const uint8_t *dest_arp,
                                int ifindex)
{
    struct sockaddr_ll dest;
    struct ip_udp_dhcp_packet packet;
    int fd, n;
    
    // 1. Create RAW socket
    fd = socket(PF_PACKET, SOCK_DGRAM, htons(ETH_P_IP));
    
    // 2. Build full Ethernet + IP + UDP + DHCP packet
    memset(&packet, 0, sizeof(packet));
    
    // IP header
    packet.ip.version = IPVERSION;
    packet.ip.ihl = sizeof(packet.ip) >> 2;
    packet.ip.tot_len = htons(sizeof(packet));
    packet.ip.protocol = IPPROTO_UDP;
    packet.ip.saddr = source_ip;
    packet.ip.daddr = dest_ip;
    packet.ip.check = dhcp_checksum(&packet.ip, sizeof(packet.ip));
    
    // UDP header
    packet.udp.source = htons(source_port);
    packet.udp.dest = htons(dest_port);
    packet.udp.len = htons(sizeof(packet.udp) + sizeof(struct dhcp_packet));
    
    // DHCP payload
    memcpy(&packet.data, dhcp_pkt, sizeof(struct dhcp_packet));
    
    // UDP checksum (with pseudo-header)
    packet.udp.check = dhcp_checksum(&packet, sizeof(packet));
    
    // 3. Set destination (link layer)
    memset(&dest, 0, sizeof(dest));
    dest.sll_family = AF_PACKET;
    dest.sll_protocol = htons(ETH_P_IP);
    dest.sll_ifindex = ifindex;
    dest.sll_halen = 6;
    memcpy(dest.sll_addr, dest_arp, 6);  // Destination MAC
    
    // 4. SEND THE PACKET
    n = sendto(fd, &packet, sizeof(packet), 0,
               (struct sockaddr *)&dest, sizeof(dest));
    
    close(fd);
    return n;
}
```

**Key Points:**
- Uses `PF_PACKET` socket (raw Ethernet access)
- **ConnMan constructs entire packet**: Ethernet → IP → UDP → DHCP
- Kernel just transmits raw bytes
- Required because we don't have an IP address yet!

#### Option 2: UDP Sockets (L3) - Used for Renewals

```c
// From gdhcp/client.c
static int dhcp_send_kernel_packet(struct dhcp_packet *dhcp_pkt,
                                   uint32_t source_ip,
                                   int source_port,
                                   uint32_t dest_ip,
                                   int dest_port,
                                   int ifindex)
{
    struct sockaddr_in dest;
    int fd, n;
    
    // 1. Create UDP socket
    fd = socket(PF_INET, SOCK_DGRAM, IPPROTO_UDP);
    
    // 2. Bind to source IP
    struct sockaddr_in src;
    src.sin_family = AF_INET;
    src.sin_port = htons(source_port);
    src.sin_addr.s_addr = source_ip;
    bind(fd, (struct sockaddr *)&src, sizeof(src));
    
    // 3. Set destination
    dest.sin_family = AF_INET;
    dest.sin_port = htons(dest_port);
    dest.sin_addr.s_addr = dest_ip;
    
    // 4. SEND (kernel adds IP/UDP headers)
    n = sendto(fd, dhcp_pkt, sizeof(*dhcp_pkt), 0,
               (struct sockaddr *)&dest, sizeof(dest));
    
    close(fd);
    return n;
}
```

**When Used:**
- During RENEWING (we have valid IP)
- During REBINDING
- Kernel adds IP and UDP headers
- More efficient than raw sockets

### 3.3 Receiving DHCP Packets

```c
// Simplified from gdhcp/client.c
static gboolean listener_event(GIOChannel *channel, 
                               GIOCondition condition,
                               gpointer user_data)
{
    GDHCPClient *dhcp_client = user_data;
    struct dhcp_packet packet;
    uint8_t *message_type;
    int bytes;
    
    // 1. Read from socket
    bytes = read(dhcp_client->listener_sockfd, &packet, sizeof(packet));
    
    // 2. Validate packet
    if (packet.xid != dhcp_client->xid)
        return TRUE;  // Not our transaction
    
    if (packet.op != BOOTREPLY)
        return TRUE;  // Not a reply
    
    // 3. Extract message type
    message_type = dhcp_get_option(&packet, DHCP_MESSAGE_TYPE);
    
    // 4. Handle based on current state and message type
    switch (dhcp_client->state) {
    case INIT_SELECTING:
        if (*message_type == DHCPOFFER) {
            handle_offer(dhcp_client, &packet);
        }
        break;
        
    case REQUESTING:
    case RENEWING:
    case REBINDING:
        if (*message_type == DHCPACK) {
            handle_ack(dhcp_client, &packet);
        } else if (*message_type == DHCPNAK) {
            handle_nak(dhcp_client, &packet);
        }
        break;
    }
    
    return TRUE;  // Keep watching
}
```

**Integration with GMainLoop:**
```c
// When starting DHCP
dhcp_client->listener_watch = g_io_add_watch(
    channel,
    G_IO_IN,
    listener_event,
    dhcp_client
);
```

---

## Part 4: Complete DHCP Flow in ConnMan

### 4.1 Initialization

```
User Connects to Network
  ↓
service.c: __connman_service_connect()
  ↓
network.c: __connman_network_connect()
  ↓
ipconfig.c: __connman_ipconfig_enable()
  ↓
dhcp.c: __connman_dhcp_start(ipconfig, device)
  ↓
gdhcp/client.c: g_dhcp_client_start(dhcp_client)
```

### 4.2 DHCP Discovery Phase

```c
// gdhcp/client.c
int g_dhcp_client_start(GDHCPClient *dhcp_client,
                        const char *last_address)
{
    // 1. Generate random transaction ID
    dhcp_client->xid = rand();
    
    // 2. Check if we should try to reuse last address
    if (last_address) {
        dhcp_client->requested_ip = inet_addr(last_address);
        dhcp_client->state = REBOOTING;
        // Send DHCPREQUEST directly (faster)
        send_request(dhcp_client);
    } else {
        dhcp_client->state = INIT_SELECTING;
        send_discover(dhcp_client);
    }
    
    // 3. Open socket for receiving replies
    switch_listening_mode(dhcp_client, L2);  // Raw socket
    
    // 4. Set timeout for retries
    dhcp_client->timeout = g_timeout_add_seconds(
        DISCOVER_TIMEOUT,
        discover_timeout,
        dhcp_client
    );
    
    return 0;
}
```

**Timeline:**
```
T=0s:    Send DHCPDISCOVER
         Start 5s timeout
         
T=1.2s:  Receive DHCPOFFER from server
         Cancel timeout
         Extract offered IP (192.168.1.100)
         Send DHCPREQUEST
         Start new 5s timeout
         
T=1.3s:  Receive DHCPACK
         Lease obtained!
         Call callback to ConnMan
```

### 4.3 Handling DHCP OFFER

```c
static void handle_offer(GDHCPClient *dhcp_client,
                        struct dhcp_packet *packet)
{
    // 1. Extract offered IP
    dhcp_client->requested_ip = packet->yiaddr;
    
    // 2. Extract server IP
    uint8_t *server_id = dhcp_get_option(packet, DHCP_SERVER_ID);
    memcpy(&dhcp_client->server_ip, server_id, 4);
    
    // 3. Move to REQUESTING state
    dhcp_client->state = REQUESTING;
    
    // 4. Send DHCPREQUEST
    send_request(dhcp_client);
    
    // 5. Set timeout
    dhcp_client->timeout = g_timeout_add_seconds(
        REQUEST_TIMEOUT,
        request_timeout,
        dhcp_client
    );
}
```

### 4.4 Handling DHCP ACK

```c
static void handle_ack(GDHCPClient *dhcp_client,
                      struct dhcp_packet *packet)
{
    // 1. Extract lease time
    uint8_t *lease = dhcp_get_option(packet, DHCP_LEASE_TIME);
    dhcp_client->lease_seconds = ntohl(*(uint32_t *)lease);
    
    // 2. Extract IP configuration
    dhcp_client->assigned_ip = inet_ntoa(packet->yiaddr);
    
    uint8_t *subnet = dhcp_get_option(packet, DHCP_SUBNET);
    uint8_t *router = dhcp_get_option(packet, DHCP_ROUTER);
    uint8_t *dns = dhcp_get_option(packet, DHCP_DNS_SERVER);
    
    // 3. Store all options in hash table
    g_hash_table_insert(dhcp_client->code_value_hash, 
                       GINT_TO_POINTER(DHCP_SUBNET), subnet);
    // ... store other options
    
    // 4. Calculate renewal times
    // T1 = 50% of lease
    // T2 = 87.5% of lease
    uint32_t t1 = dhcp_client->lease_seconds / 2;
    uint32_t t2 = (dhcp_client->lease_seconds * 7) / 8;
    
    // 5. Set state to BOUND
    dhcp_client->state = BOUND;
    
    // 6. Schedule renewal timers
    dhcp_client->t1_timeout = g_timeout_add_seconds(
        t1, t1_renew_timeout, dhcp_client
    );
    
    dhcp_client->t2_timeout = g_timeout_add_seconds(
        t2, t2_rebind_timeout, dhcp_client
    );
    
    // 7. Switch to UDP socket for future renewals
    switch_listening_mode(dhcp_client, L3);
    
    // 8. CALLBACK TO CONNMAN!
    if (dhcp_client->lease_available_cb) {
        dhcp_client->lease_available_cb(dhcp_client,
                                       dhcp_client->lease_available_data);
    }
}
```

### 4.5 Back to ConnMan (Callback Handling)

```c
// dhcp.c in ConnMan
static void lease_available_cb(GDHCPClient *dhcp_client, gpointer user_data)
{
    struct connman_dhcp *dhcp = user_data;
    struct connman_ipconfig *ipconfig = dhcp->ipconfig;
    char *address, *netmask, *gateway, *nameserver;
    
    // 1. Extract configuration from DHCP client
    address = g_dhcp_client_get_address(dhcp_client);
    netmask = g_dhcp_client_get_netmask(dhcp_client);
    gateway = g_dhcp_client_get_gateway(dhcp_client);
    
    // 2. Apply to IPConfig
    __connman_ipconfig_set_local(ipconfig, address);
    __connman_ipconfig_set_prefixlen(ipconfig, 
                                    netmask_to_prefixlen(netmask));
    __connman_ipconfig_set_gateway(ipconfig, gateway);
    
    // 3. Store DHCP address for future reuse
    ipconfig->last_dhcp_address = g_strdup(address);
    
    // 4. Trigger Address Conflict Detection (ACD)
    struct connman_network *network;
    network = __connman_service_get_network(service);
    start_acd(network);  // Will eventually call ipconfig_address_add
}
```

### 4.6 ACD and Final Configuration

```c
// After ACD succeeds (no conflict detected)
static void acd_host_ipv4_available(struct acd_host *acd, gpointer user_data)
{
    struct connman_network *network = user_data;
    struct connman_service *service;
    struct connman_ipconfig *ipconfig_ipv4;
    
    service = connman_service_lookup_from_network(network);
    ipconfig_ipv4 = __connman_service_get_ip4config(service);
    
    // 1. Add IP address to kernel
    __connman_ipconfig_address_add(ipconfig_ipv4);
    //    ↓ This sends RTM_NEWADDR via netlink
    
    // 2. Add default gateway
    __connman_ipconfig_gateway_add(ipconfig_ipv4);
    //    ↓ This sends RTM_NEWROUTE via netlink
    
    // 3. Configure DNS
    __connman_resolver_append(ipconfig_ipv4, nameservers);
    
    // 4. Update service state
    __connman_service_ipconfig_indicate_state(service,
                                             CONNMAN_SERVICE_STATE_READY,
                                             CONNMAN_IPCONFIG_TYPE_IPV4);
}
```

---

## Part 5: Lease Renewal Process

### 5.1 T1 Timer Expires (50% of lease)

```c
static gboolean t1_ren
