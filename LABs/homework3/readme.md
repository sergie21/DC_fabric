Spine1
```
enable
configure terminal

router isis UNDERLAY
   net 49.0001.0100.0000.1000.00
   is-type level-1
   address-family ipv4 unicast

interface Ethernet1
   description TO_LEAF1
   mtu 9000
   no switchport
   ip address 10.2.1.0/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet2
   description TO_LEAF2
   mtu 9000
   no switchport
   ip address 10.2.1.2/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet3
   description TO_LEAF3
   mtu 9000
   no switchport
   ip address 10.2.1.4/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Loopback0
   description UNDERLAY_ROUTER_ID
   ip address 10.0.1.0/32
   isis enable UNDERLAY
   isis passive

ip routing

end
```

Spine2
```
enable
configure terminal

router isis UNDERLAY
   net 49.0001.0100.0000.2000.00
   is-type level-1
   address-family ipv4 unicast

interface Ethernet1
   description TO_LEAF1
   mtu 9000
   no switchport
   ip address 10.2.2.0/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet2
   description TO_LEAF2
   mtu 9000
   no switchport
   ip address 10.2.2.2/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet3
   description TO_LEAF3
   mtu 9000
   no switchport
   ip address 10.2.2.4/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Loopback0
   description UNDERLAY_ROUTER_ID
   ip address 10.0.2.0/32
   isis enable UNDERLAY
   isis passive

ip routing

end
```

Leaf1
```
enable
configure terminal

router isis UNDERLAY
   net 49.0001.0100.0010.1000.00
   is-type level-1
   address-family ipv4 unicast

interface Ethernet1
   description TO_SPINE1
   mtu 9000
   no switchport
   ip address 10.2.1.1/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet2
   description TO_SPINE2
   mtu 9000
   no switchport
   ip address 10.2.2.1/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Loopback0
   description UNDERLAY_ROUTER_ID
   ip address 10.0.101.0/32
   isis enable UNDERLAY
   isis passive

interface Loopback1
   description VTEP
   ip address 10.1.101.0/32
   isis enable UNDERLAY
   isis passive

ip routing

end
```

Leaf2
```
enable
configure terminal

router isis UNDERLAY
   net 49.0001.0100.0010.2000.00
   is-type level-1
   address-family ipv4 unicast

interface Ethernet1
   description TO_SPINE1
   mtu 9000
   no switchport
   ip address 10.2.1.3/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet2
   description TO_SPINE2
   mtu 9000
   no switchport
   ip address 10.2.2.3/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Loopback0
   description UNDERLAY_ROUTER_ID
   ip address 10.0.102.0/32
   isis enable UNDERLAY
   isis passive

interface Loopback1
   description VTEP
   ip address 10.1.102.0/32
   isis enable UNDERLAY
   isis passive

ip routing

end
```

Leaf3
```
enable
configure terminal

router isis UNDERLAY
   net 49.0001.0100.0010.3000.00
   is-type level-1
   address-family ipv4 unicast

interface Ethernet1
   description TO_SPINE1
   mtu 9000
   no switchport
   ip address 10.2.1.5/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet2
   description TO_SPINE2
   mtu 9000
   no switchport
   ip address 10.2.2.5/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Loopback0
   description UNDERLAY_ROUTER_ID
   ip address 10.0.103.0/32
   isis enable UNDERLAY
   isis passive

interface Loopback1
   description VTEP
   ip address 10.1.103.0/32
   isis enable UNDERLAY
   isis passive

ip routing

end
```
