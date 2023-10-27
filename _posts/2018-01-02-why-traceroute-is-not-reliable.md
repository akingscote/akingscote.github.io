---
title: "Why traceroute isnt that great..."
date: "2018-01-02"
categories: 
  - "networking"
coverImage: "traceroute.png"
---

Traceroute is a tool that shows the hops that **network** traffic has taken to reach its destination - this can be extremely useful for network diagnostics. As an administrator can _trace_ the _route_ of a packet and potentially identify any blockers (firewalls, routing, connection issues etc...) as well as view the delay at each hop.

This post is to advise not to use traceroute as an absolute source of truth. It is incredibly useful but it is worth noting that traceroute is not 100% reliable.

Traceroute uses Internet Control Message Protocol (ICMP) which is the same as what the ping utility uses. With ICMP, each packet is sent with a Time To Live (TTL) value which gets decremented by 1. If the TTL reaches 0, then the packet is dropped and a Time Exceeded in Transit message is returned back to the source node.

With traceroute, the round trip time (RTT) is recorded at each hop along the route. The **total of the averages** at each hop is used to calculate the time taken to establish the connection. Traceroute attempts to send three packets to each hop, if 2/3 are lost then the connection is dropped and a destination unreachable message is returned to the source node. This differs from ping where the round trip time is calculated from the end node as opposed to an the sum of the average (three packets) from each node on the route.

Traceroute sends the first packet with a TTL of 1 and increases the TTL with each packet (three packets sent for each node). If the packet reaches a router, the router will then decrease the TTL value by 1. Typically traceroute uses UDP at port 33434.

![](/images/traceroute.png)

Traceroute discovers paths at the interface level rather than the router level. On most occasions with a network you will want two way communication, as traceroute requires a reply, it will fail if there is not a return route or if there is a firewall blocking the return. As traceroute uses User Datagram Protocol, it is inherently unreliable as UDP is a best effort transport protocol. Some routes down prioritise ICMP packets, which can add delay. As well as this, with traceroute equal cost routes vary meaning you may get different results each time the utility is used. Traceroute only shows how many layer 3 hops you are away (switches are ignored). This means that your data could be traversing a huge MPLS cloud and not know it (suprisingly this isnt common knowledge about traceroute).  Finally any device that dosent decrease the TTL won't show on the traceroute like the Cisco ASA firewall.

Basically, its important to know how traceroute actually works in order to help dianose network issues.
