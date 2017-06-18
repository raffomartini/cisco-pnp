# ISIS configuration for ECMP

The purpose is to configure ISIS for ECMP.
The c3650 connects back to the lab using EIGRP.
My topology is very simple:
```
(LAB)---|c3650| --- |c3850-1|
           |
       |c3850-2|
```

I've decided to use /30 networks for the p2p links, in the range 192.168.254.0 - 255
All devices have a loopback, I am using the subnet 10.102.0.0/24 for the loopbacks address.

The configuration is done in 2 steps:
  * router configuration
  * interface configuration
  
  
### Router configuration:
```
router isis
 net 49.0000.0101.0200.0003.00
 is-type level-2-only
 metric-style transition
 log-adjacency-changes
 bfd all-interfaces
```

### Interface configuration
```
 ip router isis 
 logging event link-status
 load-interval 30
 carrier-delay msec 0
 bfd interval 300 min_rx 300 multiplier 3
 no bfd echo
```
**Note:** without the bfd command the adjacency will never go up on a cat3k switch.

## Redistribuite the default route
As this is a stub network, I am happy with just redistributing the default route (again, in the interest of not cluttering the route table with pointless entries.
```
router isis
 default-information originate
```

## Advertise back to the LAB only the loopback subnet
I use some eigrp route summarisation here.
It is done in 2 steps: 
  1. first you define the summary route in the network statement for the router
  2. you then use the summary route feature on the interface
**router configuration**
```
router eigrp 100
 network 10.102.0.0 0.0.0.255
 eigrp stub summary
```
Note I have also configured the router as stub summar to only advertise the redistributed route back to the lab.

**interface configuration**
```
interface Vlan1
 ip summary-address eigrp 100 10.102.0.0 255.255.255.0
```



## Filter ECMP link
Normally I would see  all the point to point links in the routing table (ipv6 woud solve this with link local addresses).
This is cluttering my routing table with destination I don't want to reach (I'm ok to go to the loopback if I have to):
```

      10.0.0.0/32 is subnetted, 2 subnets
i L2     10.102.0.1 
           [115/20] via 192.168.254.13, 00:08:24, GigabitEthernet1/0/24
           [115/20] via 192.168.254.9, 00:08:24, GigabitEthernet1/0/23
C        10.102.0.3 is directly connected, Loopback0
      192.168.254.0/24 is variably subnetted, 6 subnets, 2 masks
i L2     192.168.254.0/30 
           [115/20] via 192.168.254.13, 00:00:03, GigabitEthernet1/0/24
           [115/20] via 192.168.254.9, 00:00:03, GigabitEthernet1/0/23
i L2     192.168.254.4/30 
           [115/20] via 192.168.254.13, 00:00:03, GigabitEthernet1/0/24
           [115/20] via 192.168.254.9, 00:00:03, GigabitEthernet1/0/23
C        192.168.254.8/30 is directly connected, GigabitEthernet1/0/23
L        192.168.254.10/32 is directly connected, GigabitEthernet1/0/23
C        192.168.254.12/30 is directly connected, GigabitEthernet1/0/24
L        192.168.254.14/32 is directly connected, GigabitEthernet1/0/24
```
The solution is to do inbound filtering:
```
ip access-list standard filter-isis-ecmp-address
 deny   192.168.254.0 0.0.0.255
 permit any
router isis
 distribute-list filter-isis-ecmp-address in
```
Much better now:
```
      10.0.0.0/32 is subnetted, 2 subnets
i L2     10.102.0.1 
           [115/20] via 192.168.254.13, 00:05:08, GigabitEthernet1/0/24
           [115/20] via 192.168.254.9, 00:05:08, GigabitEthernet1/0/23
C        10.102.0.3 is directly connected, Loopback0
      192.168.254.0/24 is variably subnetted, 4 subnets, 2 masks
C        192.168.254.8/30 is directly connected, GigabitEthernet1/0/23
L        192.168.254.10/32 is directly connected, GigabitEthernet1/0/23
C        192.168.254.12/30 is directly connected, GigabitEthernet1/0/24
L        192.168.254.14/32 is directly connected, GigabitEthernet1/0/24
```

