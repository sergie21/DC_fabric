Spine1
'''
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
'''
