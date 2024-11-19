---
title: "Subnetting Guide"
date: "2022-07-24"
categories: 
  - "networking"
tags: 
  - "binary"
  - "decimal"
  - "networking"
  - "subnetting"
coverImage: "global.png"
---

Most technology professionals will stumble across networking at some point. The minefield of subnetworks, CIDRs, broadcast addresses, network addresses and subnets can be overwhelming, but most people manage by using the old' familiar 192.168.0.0/24 network.

Ever wonder why IP addresses have a max value of 255? Or why 192 is so popular? Read on and it will all become clearer.

Whenever i've needed to do actual subnetworking, i always just end up using a [subnet calculator](https://www.subnet-calculator.com/), rather than figure it out by scratch. However, recently i decided to try and remember how it all works and thought i'd write down some of my notes.

# Networking and Subnetworking

An IPv4 address is 32 bits long, split across four octets. What on earth does that mean? It essentially means that each section of an IP address (x.x.x.x) (octet), is 8 bits long. A bit is a binary representation of being present or not being present. 0 means not present, 1 means present. So an octet could look like this:

`11010011`

or any other combination of ones and zeros, as long as there are eight in total. Combine four octets together to get an IPv4 IP address. For example

`11010011.11011011.101011100.00110111`

Now binary isnt very human readable, so we often use decimal notation, like 192.168.0.1. However, binary notation is very useful when it comes to subnetting and understanding how/why IPv4 networking works.

## Converting Binary to Decimal

Within an octet, each bit is either on/off, so the values are either yes/no, one/zero. However, when taken in combination with the entire octet, we can create more complex values. Similar to binary logic gates, we can create n2 combinations of decimal values. For example, `10101010` means 170, `11110000` means `240`, `00000000` means zero.

Starting from the rightmost bit within an octet, the we can associate each position with a value.

![](/images/subnet.png)

`0000000**1**`

In the above example, that first position is set.  That position represents a one. As the value is set but nothing else, that entire octet's value is one.

`000000**1**0`

In the above example, that second position represents a two, As the value is set but nothing else is, that entire octet's value is two.

`000000**11**`

In the above example, both the first and section positions are set. We combine both position values, which gives us a value of three.

`**11111111**`

In this example, every single possible position is set. We add up every single positional value.

128 + 64 + 32 + 16 + 8 + 4 + 2 + 1

And the answer is...

255

The highest possible value we can have in an octet is 255.

`11000000`

This example, has the left two positions set. Those positions are 128 and 64. Adding them together, gives us the value 192! Thats essentially why 192 is so prevalent, its because the last (or first, depending on how you look at it) bit are set.

 

## Converting Decimal to Binary

Converting binary to decimal is pretty straight forward, but how do we do the reverse?

Well its not actually that bad, but requires some basic arithmetic. Lets pick a decimal number for our octet (it needs to be valid, so less than 255). Lets go with 241.

To figure this out in binary, lets go back to our diagram which highlights the positional values.

![](/images/subnet2.png)

Our value, (241) is higher than 128, so we know that bit must be set.

![](/images/subnet3.png)

Next, we minus 128 from our value, to get the remainder. 241-128=113.

Looking at the next position, we know that 113 is greater than 64, so that bit must also be set.

![](/images/subnet4.png)

We repeat the same process. 113-64=49. Looking at the next position, 32, we know that 49 is larger so that bit must also be set.

This process continues until we hit a scenario where the position value is NOT greater than our remainder. In that case, we can assume the bit is zero.

![](/images/bin-to-decimal.png)

So we have now figured out that the binary representation of 241 is

`11110001`

## Subnetting

A CIDR essentially means how many consecutive bits are set in a subnet address. For example, /27 would look like this:

`11111111.11111111.11111111.11100000`

Which is also the same as saying `255.255.255.224` in decimal notation.

Here are four questions, derived from CCNA type mock exam questions which really helped with my understanding.

## How many subnets and hosts per subnet can you get from the network 192.168.51.0/29?

1. Convert both to binary

Network 192 168 51 29
Network 11000000 10101000 00110011 00000000
Subnet  255 255 255 248
Subnet  1111111 11111111 11111111 11111000

2. How many hosts are there? Look at the subnet, see how many host bits are 0. Apply the number in the formula 2**n**-2

Subnet 1111111 11111111 11111111 11111000

Three zeros 2**3**-2 = 6 Hosts

3. How many subnets? Take value of last subnet bit that is ‘1’ and increment by it to find networks

Network 11000000 10101000 00110011 00000000
Subnet  11111111 11111111 11111111 11111**000**
                                 128 64 32 16 8 4 2 1

Network 1 192 168 51 0 +8
Network 2 192 168 51 8 +8
Network 3 192 168 51 16 +8
Network 4 192 168 51 24 +8
...
Network 32 192 168 51 255

Answer is 32 Subnets and 6 Hosts

## What is the broadcast address of the network 172.20.84.0 255.255.255.0?

1. Convert Both to binary

Network 172 20 84 0
Network 10101100 00010100 01010100 00000000
Subnet  255 255 255 0
Subnet  11111111 11111111 11111111 0000000

2. Take value of last subnet bit that is ‘1’ and increment by it to find network

Network 10101100 00010100 01010100 00000000
Subnet  11111111 11111111 1111111**1** 0000000

Network 1 172 20 84 0
Network 2 172 20 85 0
Network 3 172 20 86 0
Network 4 172 20 87 0
...

The broadcast address is the last possible address before going to the next subnet

Network 1 172 20 84 0
Broadcast 172 20 84 255
Network 2 172 20 85 0
Broadcast 172 20 85 255
Network 3 172 20 86 0
Broadcast 172 20 86 255
Network 4 172 20 87 0
Broadcast 172 20 87 255

Therefore the answer is 172.20.84.255

## You are designing a subnet mask for the 192.168.97.0 network. You want 6 subnets with up to 16 hosts on each subnet. What subnet mask should you use?

We want 16 hosts on each subnet. Look at host portion
```
0          2**1**-2 = 0
00         2**2**-2 = 2
000        2**3**-2 = 6
0000       2**4**-2 = 14
00000      2**5**-2 = 30
000000     2**6**-2 = 62
0000000    2**7**-2 = 126
00000000   2**8**-2 = 254
000000000  2**9**-2 = 510
```

The only one that will give us 16 hosts, is 2**5**-2 = 30 which is 00000. So the last octet will look like

11100000

which means the full subnet will look like

11111111.1111111.11111111.11100000

The netmask is /27 which means the address in decimal is 255.255.255.224

Answer is 255.255.255.224

## What valid host range is the IP address 172.31.187.160/23 a part of?

1. Convert both to binary

Network 172 31 187 160
Network 10101100 00011111 10111011 10100000
Subnet  255 255 254 0
Subnet  11111111 11111111 11111110 00000000

2. Find network address Look where network and subnet bits are both 1

Network         **1**0**1**0**11**00 000**11111** **1**0**111**0**1**1 10100000
Subnet          **1**1**1**1**11**11 111**11111** **1**1**111**1**1**0 00000000
Network Address 10101100 00011111 10111010 00000000

So the first network address is

Network Address 10101100.00011111.10111010.00000000
Decimal         172.31.186.0

Now we need to see how much they increment by. Find last active 1 in subnet and increment by that

Network Address 10101100 00011111 10111010 00000000
Subnet          11111111 11111111 11111110 00000000

Network 1 172 31 186 0 +2
Network 2 172 31 188 0 +2
Network 3 172 31 190 0 +2
Network 4 172 31 192 0 +2

## Question is what is the last valid host range for 172.31.187.160?

This node is in the first network. The first host is one after the network address (172.31.186.1) The last host will be Network 2 minus one. (172.31.187.255) Answer is 172.31.186.1 - 172.31.187.255

[Networking icons created by Freepik - Flaticon](https://www.flaticon.com/free-icons/networking "networking icons")
