---
title: Routing guidelines and scenarios
last update: 02/28/2022
---

# Routing guidelines and scenarios

- [Overview](#overview)
- [Routing examples](#routing-examples)
  - [Explicit LPM](#explicit-lpm)
  - [Peered VNET use Mappings](#peered-vnet-use-mappings)
  - [VNET_A w/Subnets](#vnet_a-wsubnets)
    - [Direct communication between subnets w/mapping](#direct-communication-between-subnets-wmapping)
  - [Add firewall hop to the routes (Communication between subnets w/firewall (NVA) w/firewall next hop route entry)](#add-firewall-hop-to-the-routes-communication-between-subnets-wfirewall-nva-wfirewall-next-hop-route-entry)
    - [Mapping](#mapping)
    - [RouteTable (LPM)](#routetable-lpm)
  - [Set default route](#set-default-route)
    - [Mapping](#mapping-1)
    - [RouteTable (LPM)](#routetable-lpm-1)
  - [Next hop in peered VNET w/mapping via firewall (NVA)](#next-hop-in-peered-vnet-wmapping-via-firewall-nva)
  - [Set a specific Internet route w/firewall](#set-a-specific-internet-route-wfirewall)
    - [Mapping](#mapping-2)
    - [RouteTable (LPM)](#routetable-lpm-2)
  - [Set an on premises route to a express route (ExR) PA](#set-an-on-premises-route-to-a-express-route-exr-pa)
    - [RouteTable (LPM)](#routetable-lpm-3)
  - [Set an on premises route to a next hop express route (ExR) PA with two private addresses (usually paired 2 endpoints) w/different GRE key](#set-an-on-premises-route-to-a-next-hop-express-route-exr-pa-with-two-private-addresses-usually-paired-2-endpoints-wdifferent-gre-key)
    - [RouteTable (LPM)](#routetable-lpm-4)
  - [Set private links routes using mapping, routes, or peered VNETs](#set-private-links-routes-using-mapping-routes-or-peered-vnets)
- [Counters](#counters)
- [Terminology](#terminology)

## Overview

This article explains the basic steps to build a **routing table** (also known
as a *forwarding* table) and how to use **mappings**.  
The route is a concept of ENI/VNIC, not a VNET (i.e. the route table is attached
to ENI/VNIC) It is important to notice from the get go, **routing** and
**mapping** are two different but complementary concepts, specifically:

**Routing** 

The route table is configured by the customer to provide the desired traffic routing behavior; traffic can also be intercepted or redirected.  
It must be clear that the routing table has the final say in the way the traffic is routed (Longest Prefix Match = wins). Routes can intercept **part** of the
traffic and forward to a next hop for the purpose of filtering.  The order is:  LPM->Route->Mapping.  We ONLY look at mappings, AFTER LPM decides
that a route wins.

For example, a default route appears as below:
- [Overview](#overview)
  
    `0/0 -> Internet (Default)`

    The entry following shows how a customer can override the entry and route the traffic differently:

    `8.8.0.0/16 -> Internet (SNAT to VIP)`

    `0/0 -> Default Hop: 10.1.2.11 (direct to a Firewall in current VNET)`

**Mapping**

Mapping lookups determine the network physical address (PA) spaces to redirect traffic.  
A mapping is a PA to CA (Physical Address to Customer Address) lookup table, and Encap determination (for example).

    `10.3.0.0/16 -> VNET C (Peered) (use mapping)`

Notice that a routing table has a size limit of about 100K while mapping table
has a limit of 1M. Using mappings extends the amount of data
that can be contained in the routing table.  

One of the main objectives of a routing table, more specifically **LPM routing
table**, is to allow the customers to enter static or mapped entries per their
requirements. The LPM routing rules determine the order. The rules can be either
static or can refer to a mapping. But mappings do not control routing, which is
decided via the LPM table.  

- **Static** :  when an entry is created in the table, the exact physical address (PA) is known; there is no mapping (lookup).
- **Mapping** : for a particular entry, the desired behavior is to intercept the
traffic and exempt it from the standard routing, in order to apply
different actions than the ones associated with the rest of the traffic.

## Routing examples

This section provides guidelines, along with some examples, for how to build
routing tables statically and/or by using mapping.  It includes the types of entries an LPM routing table may
contain. We'll describe the various entries as we progess with the explanation.

**Example route table (basic customer setup)**

```

- 10.1/16 -> VNET A (via mapping lookup)
- 10.1.0/24 -> UDR to transit next hop 10.2.0.5 (ex. intercept this subnet through firewall VM in peered vnet)
- 10.1.0.7/32 -> VNET A (exempt this IP from being intercepted by UDR above and use normal VNET route, as LPM on /32 wins)
- 10.1.0.10/32 -> UDR to transit next hop 10.2.0.5 (ex. customer wants to intercept traffic to this destination and filter it via firewall)
- 10.1.0.11/32 -> This is a Private Endpoint plumbed as /32 route
- 10.1.0.12/32 -> This is another Private Endpoint plumbed as /32 route
- 10.2/16 -> Peered VNET B (via mapping lookup)
- 10.2.0.8/32 -> This is another Private Endpoint in peered vnet plumbed as /32 route
- 50/8 -> Internet (allow this 50/8 traffic to be exempt from transiting the firewall, and allow it to go directly to internet)
- 20.1/16 -> Static Route to on-prem (encap with some GRE key and send to CISCO Express Route device, that later redirects to onprem)
- 20.2/16 -> Static Route to on-prem (encap with some GRE key and send to CISCO Express Route device, that later redirects to onprem)
- 0/0 -> UDR to transit next hop 10.1.0.7 (ex. firewall all traffic going originally through internet via firewall which is in the same vnet)

```

**VNET, Mappings, Private Link, Express Route, Internet Examples**

```
VNET: 10.1.0.0/16
- Subnet 1: 10.1.1.0/24
- Subnet 2: 10.1.2.0/24  (VM/NVA: 10.1.2.11 - Firewall)
- Subnet 3: 10.1.3.0/24
- Mappings: . VM 1: 10.1.1.1 (y) . VM 2: 10.1.3.2 . Private Link 1: 10.1.3.3 .
 Private Link 2: 10.1.3.4 . VM 3: 10.1.3.5
 
RouteTable attached to VM 10.1.1.1
- 10.1.0.0/16 -> VNET (use mappings)
- 10.1.3.0/24 -> Hop: 10.1.2.11 Customer Address (CA) -> Private Address (PA)
  (Firewall in current VNET)
- 10.1.3.0/26 -> Hop: 10.1.2.88 Customer Address (CA) -> Private Address
  (PA)(Firewall in peered VNET)
- 10.1.3.5/27 -> VNET A (mapping)
- 10.1.3.3/32 -> Private Link Route (Private Link 1)
- 10.2.0.0/16 -> VNET B (Peered) (use mapping)
- 10.2.1.0/24 -> Hop: 10.1.2.11 Hop: 10.1.2.88(CA->PA) (Firewall in peered VNET)
- 10.2.0.0/16 -> VNET B (Peered) (use mappings)
- 10.3.0.0/16 -> VNET C (Peered)  (use mappings)
- 50.3.5.2/32 -> Private Link Route (Private Link 7)
- 50.1.0.0/16 -> Internet
- 50.0.0.0/8 -> Hop: CISCO ER device PA (100.1.2.3, 10.1.2.4), GRE Key: X
- 8.8.0.0/16 -> Internet (SNAT to VIP)
- 0/0 -> Default Hop: 10.1.2.11 (Firewall in current VNET)

```

Please note, a routing table is attached to a specific VM in the VNET, not to VNET
itself. The route is a concept of ENI/VNIC, not a VNET (i.e. route table is attached to ENI/VNIC).  
In a VNET the VM functions like a router, to which a routing table is attached.
This makes a difference when plumbing metering.

![dash-dataplane-routing-table-vm](./images/dash-dataplane-routing-table-vm.svg)

<figcaption><i>Figure 1. Routing table per VM</i></figcaption><br/>

## SCENARIOS (these build upon each other)

### Scenario: Explicit LPM 
#### This example is a single VNET with direct traffic between VMs using mappings
#### Customer provides entries, we handle by default

Route Table - attached to VM x.x.x.x
- 10.1.0.0/16 -> VNET (use mappings)
- 0/0 -> Default (Internet)

### Scenario: Peered VNET using Mappings

Route Table - attached to VM x.x.x.x
- 10.1.0.0/16 -> VNET A (use mappings)
- 10.2.0.0/16 -> VNET B (use mappings)
- 10.3.0.0/16 -> VNET C (use mappings)
- 0/0 -> Default (Internet)

### Scenario: Direct communication between subnets w/mapping and addition of next hop (such as a Firewall)
#### Example: Customer wants to filter traffic from subnet 1 to subnet 3 through a FW on subnet 2
#### VNET w/Subnets
VNET: 10.1.0.0/16
- Subnet 1: 10.1.1.0/24
- Subnet 2: 10.1.2.0/24  (VM/NVA: 10.1.2.11 - Firewall) **Customer places FW here**
- Subnet 3: 10.1.3.0/24

#### Add firewall hop to the routes (Communication between subnets w/firewall (NVA) next hop route entry)
Route Table attached to VM x.x.x.x
- 10.1.0.0/16 -> VNET A (use mappings)

**- 10.1.3.0/24 -> Next Hop: (10.1.2.11) - Customer adds Subnet 3 Next Hop through Firewall in Current VNET here**

**- 10.1.3.0/26 -> Next Hop: (10.2.0.88) - Another example, Firewall Next Hop in a Peered VNET**
- 10.2.0.0/16 -> VNET B (use mappings)
- 10.3.0.0/16 -> VNET C (use mappings)
- 0/0 -> Default (Internet)

### Scenario: Customer wants to filter default route (ex: route all Internet destined traffic through a firewall).

The example shows how redirect default route (Internet) traffic to through a firewall.

VNET: 10.1.0.0/16
- Subnet 1: 10.1.1.0/24
- Subnet 2: 10.1.2.0/24  (VM/NVA: 10.1.2.11 - Firewall) **Customer places FW here**
- Subnet 3: 10.1.3.0/24

#### Add firewall hop to the routes (Communication between subnets w/firewall (NVA) next hop route entry)
RouteTable attached to VM 10.1.1.1
- 10.1.0.0/16 -> VNET A (use mappings)
**- 10.1.3.0/26 -> Next Hop: (10.1.2.11) - next hop here from previous example**
- 10.2.0.0/16 -> VNET B (use mappings)
- 10.3.0.0/16 -> VNET C (use mappings)
- 0/0 -> Next Hop: 10.1.2.11 **Customer overrides Default Route with a Next Hop of 10.1.2.11 (firewall in VNET)**

```
```
### Scenario: Trusted Internet-bound traffic is direct, Untrusted transits firewall

The example shows how to set a permit specific Internet destined traffic to transit directly, while the rest transits a firewall.

VNET: 10.1.0.0/16
- Subnet 1: 10.1.1.0/24
- Subnet 2: 10.1.2.0/24  (VM/NVA: 10.1.2.11 - Firewall) **Customer places FW here**
- Subnet 3: 10.1.3.0/24

#### Add firewall hop to the routes (Communication between subnets w/firewall (NVA) next hop route entry)
Route Table attached to VM x.x.x.x
- 10.1.0.0/16 -> VNET A (use mappings)
**- 10.1.3.0/26 -> Next Hop: (10.1.2.11) - next hop here from previous example**
- 10.2.0.0/16 -> VNET B (use mappings)
- 10.3.0.0/16 -> VNET C (use mappings)
- 8.8.0.0/16 -> Internet **(for Trusted traffic) - (can be SNAT to VIP)** 
- 0/0 -> Next Hop: 10.1.2.11 **for Untrusted traffic**

```

### Scenario: Set an on premises route to a express route (ExR) PA 

The example shows how to set an on premises route to an express route (ER) for a
specific private address (PA).

VNET: 10.1.0.0/16
- Subnet 1: 10.1.1.0/24
- Subnet 2: 10.1.2.0/24  (VM/NVA: 10.1.2.11 - Firewall) **Customer places FW here**
- Subnet 3: 10.1.3.0/24

**On-Prem: 50.0.0.0/8 - Customer On Prem space**

Route Table attached to VM x.x.x.x
- 10.1.0.0/16 -> VNET A (use mappings)
- 10.1.3.0/24 -> Next Hop: 10.1.2.11 **(CA -> PA)** - next hop here from previous example**
- 10.1.3.0/26 -> Next Hop: 10.2.0.88 **(CA -> PA)** (firewall in peered VNET)**
- 10.2.0.o/16 -> VNET B (use mappings)
- 10.3.0.o/16 -> VNET C (use mappings)
- 50.1.0.0/16 -> Internet **Used for Intercept**
- 50.0.0.0/8 -> Next Hop: **ER Device PA (100.1.2.3, 100.1.2.4) 2 Endpoints, GRE Key: X**
- 8.8.0.0/16 -> Internet (for Trusted traffic) - (can be SNAT to VIP)
- 0/0 -> Next Hop: 10.1.2.11 for Untrusted traffic

#### Route Table

In the example below the RouteTable (LPM) is attached to VM `10.1.1.1`.

```
- 10.1.0.0/16 -> VNET
- 50.0.0.0/8 -> Hop CISCO Express Route (ER) device PA (100.1.2.3)
```

Where the on premises route: `50.0.0.0/0` is the customer on premises space.

### Set an on premises route to a next hop express route (ExR) PA with two private addresses (usually paired 2 endpoints) w/different GRE key

The example shows how to set an on premises route to an express route (ER) with
two private addresses (end points) and **Generic Routing Encapsulation** (GRE)
key.

#### RouteTable (LPM)

```
- 50.0.0.0/8 -> Hop CISCO Express Route (ER) device PA (100.1.2.3, 100.1.2.4)
- 50.1.0.0/16 -> Internet - This is also supported
```

### Set private links routes using mapping, routes, or peered VNETs
PEs (Private Endpoints) can be /32 routes or mappings

The following example shows how the traffic to private links and VMs can be
directed to a firewall.  

Let’s say we have the following mapping:

```
VNET: 10.1.0.0/16
- Subnet 1: 10.1.1.0/24
- Subnet 2: 10.1.2.0/24 (VM/NVA: 10.1.2.11 - Firewall)
- Subnet 3: 10.1.3.0/24

- Mappings: . VM 1: 10.1.1.1 . VM 2: 10.1.3.2 . Private Link 1: 10.1.3.3 .
    Private Link 2: 10.1.3.4 . VM 3: 10.1.3.5`
```

VM 2, VM 3 and the private links belongs to the Subnet 3: `10.1.3.0/24`. 

The traffic to private links and VMs can be directed to a firewall by adding the
entry shown below to the routing table. 

```

- 10.1.3.0/26 -> Hop: 10.1.2.88 Customer Address (CA) -> Private Address (PA)
  (Firewall in peered VNET)

```

We should also be able to add a private link route to the routing table as shown
below. In this case the routing happens because of the entry in the table not
because of the mapping. 

```

- 10.1.3.3/32 -> Private Link Route (Private Link 1) 

```

> [!NOTE] In the past Microsoft only allowed private links to be added to the
> routing table. But this was not scalable because of the big amount of private
> links. So, the ability was added to use mappings for the routing of the
> private links.  

## Counters

This section briefly introduces the **counters**. A more in depth description
will be found in a document dedicated to this topic.

> [!NOTE] When and how metering is done depends on the way routing is done that is
> statically or via mapping, 

The following applies:

- We need a Counter on both the Route and the Mapping.
- The idea is to treat private endpoints as customer addresses (CA). 
- We are only evaluating private links mappings not using explicit routes. 
- Private endpoints mappings take precedence over everything. 
- If the VMs in a peer VNET have meters, they are going to be used because they
  are attached to the ultimate destination. 
- Because the mapping of the (metering) object is at VNET level, not at VNIC
  level, the metering object means different things depending where the source
  came from.


The question is do you need to specify for each ENI every possible destination
for correct application of the metering (counters)?  
The answer is because VNET is global (there is no VNET for each ENI), those
counters will be global. Otherwise, we have to copy the entire VNET object for
each ENI that would be impossible. But you can get the counters meaning from the
VNET context.  

Different ENI in peered VNET need to have context on the ENI counter for every other NIC. 
Mapping and Peered VNET and statically isolate each value (right now we rely on the fact that the 
mappings are not hit by different ENIs).  
At time of programming of ENI, we now we have to know..?


## Terminlogy

- **GRE**. Generic Routing Encapsulation is a Cisco developed tunneling
protocol. It is a simple IP packet encapsulation protocol used when IP packets
need to be transported from one network to another network, without being
notified as IP packets by any intermediate routers.
  
- **LPM**. LPM or longest prefix match refers to an algorithm used by routers in
Internet Protocol (IP) networking to select an entry from a routing table.
Because each entry in a forwarding table may specify a sub-network, one
destination address may match more than one forwarding table entry. The most
specific of the matching table entries — the one with the longest subnet mask —
is called the longest prefix match. It is called this because it is also the
entry where the largest number of leading address bits of the destination
address match those in the table entry.
- **Routing**. Routing is the process of sending a packet of information from
  one network to another network. Routers build **routing tables** that contain
  the following information:
  - Destination network and subnet mask.
  - Next hop to get to the destination network.
  - Routing metrics.

- **SNAT**. The Source Network Address Translation (SNAT) is typically used when
  an internal/private host needs to initiate a connection to an external/public
  host. The device performing NAT changes the private IP address of the source
  host to public IP address. It may also change the source port in the TCP/UDP
  headers.
  
- **VIP**. The Virtual IP Address (VIP) is a public IP address that may be
  shared by multiple devices connected to the Internet.