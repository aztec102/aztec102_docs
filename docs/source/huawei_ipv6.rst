Huawei IPv6
===========

Была хотелка протестить IPv6 на сети оператора, как обычный + так и PD
Схема - два сервера DHCPv6 (Kea DHCP) в кластере, общение происходит по схеме - dhcp relay

Тестируем IPv6 на свитче CloudEngine S6730-H:

::

    dhcpv6 server group OSA_TEST
     dhcpv6-server 2A05:7400:3:406::1
     dhcpv6-server 2A05:7400:3:406::2

    bgp 31257
     ipv6-family vpn-instance inet
     dampening
     import-route direct route-policy RM-IPV6-NET-BGP
     import-route static route-policy RM-IPV6-NET-BGP
     import-route unr
    
    # Терминация сеточек на классическом SVI
    interface Vlanif237
     ip binding vpn-instance inet
     ipv6 enable
     ipv6 address 2A05:7400:100:1::1/64
     undo ipv6 nd ra halt
     ipv6 nd autoconfig managed-address-flag
     ipv6 nd autoconfig other-flag
     dhcpv6 relay server-select OSA_TEST
     dhcpv6 relay advertise prefix-delegation route

    # Терминация сеточек на Sub-interface

    interface XGigabitEthernet0/0/17.1
     dot1q termination vid 237
     ip binding vpn-instance inet
     ipv6 enable
     ipv6 address 2A05:7400:100:1::1/64
     undo ipv6 nd ra halt
     ipv6 nd autoconfig managed-address-flag
     ipv6 nd autoconfig other-flag
     ipv6 nd ns multicast-enable
     dhcpv6 relay server-select OSA_TEST
     dhcpv6 relay advertise prefix-delegation route

Тестируем IPv6 на роутере NetEngine 8000:

::

    interface Eth-Trunk51.237
     vlan-type dot1q 237
     ipv6 enable
     ipv6 address 2A05:7400:100:1::1/64
     undo ipv6 nd ra halt
     ipv6 nd autoconfig managed-address-flag
     ipv6 nd autoconfig other-flag
     dhcpv6 relay source-ip-address 2A05:7400:100:1::1
     dhcpv6 relay binding server group OSA_TEST