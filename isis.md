# ISIS configuration for ECMP

THe purpose is to configure ISIS for ECMP.
My topology is very simple:
```
(LAB)---|c3650| --- |c3850-1|
           |
       |c3850-2|
```

I've decided to use /30 networks for the p2p links, in the range 192.168.254.0 - 255

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

### Filter ECMP link
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

