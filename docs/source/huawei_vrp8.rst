Huawei VRP8
===========

Сама по себе операционная система VRP8 имеет приятные бонусы для работы:

#. Auto-save config
#. Commit/Rollback
#. Если это маршрутизатор то конечно же - XPL

Подключение по протоколу BGP используя возможности XPL:

::

    # Наш ASP
    xpl as-path-list ASP-AS9000
     regular ^$
     end-list

    # ASP клиентов
    xpl as-path-list ASP-AS9000-CLIENTS
     ! Example Client
     regular ^9001_
     end-list

    # Главный PL advertise который будет учавствовать в политиках маршрутизации
    xpl ip-prefix-list PL-ADV-AS9000
     ! -- All AS9000 prefixes --
     212.50.32.0 24
     end-list

    # Давайте сначала определить те AS приватные, которые мы не должны принимать от FV
    xpl as-path-list ASP-BOGONS
     ! -- bogons AS --
     regular _0_,
     regular _23456_,
     regular _((6449[6-9])|(64[5-9][0-9][0-9]))_,
     regular _(6[5-9][0-9][0-9][0-9])_,
     regular _([7-9][0-9][0-9][0-9][0-9])_,
     regular _((1[0-2][0-9][0-9][0-9][0-9])|(130[0-9][0-9][0-9]))_,
     regular _((1310[0-6][0-9])|(13107[0-1]))_,
     regular _(42[0-8][0-9][0-9][0-9][0-9][0-9][0-9][0-9])_,
     regular _(429[0-3][0-9][0-9][0-9][0-9][0-9][0-9])_,
     regular _(4294[0-8][0-9][0-9][0-9][0-9][0-9])_,
     regular _((42949[0-5][0-9][0-9][0-9][0-9])|(429496[0-6][0-9][0-9][0-9]))_,
     regular _((4294967[0-1][0-9][0-9])|(42949672[0-8][0-9])|(429496729[0-5]))_
     end-list

    # Далее определим префикс лист который будет фильтровать BOGON-сети принимаемые от FV
    xpl ip-prefix-list PL-IPV4-BOGON
     0.0.0.0 8 le 32,
     10.0.0.0 8 le 32,
     100.64.0.0 10 le 32,
     127.0.0.0 8 le 32,
     169.254.0.0 16 le 32,
     172.16.0.0 12 le 32,
     192.0.0.0 24 le 32,
     192.0.2.0 24 le 32,
     192.88.99.0 24 le 32,
     192.168.0.0 16 le 32,
     198.18.0.0 15 le 32,
     198.51.100.0 24 le 32,
     203.0.113.0 24 le 32,
     224.0.0.0 4 le 32,
     240.0.0.0 4 le 32
     end-list

    xpl ip-prefix-list PL-IPV4-LONG
     0.0.0.0 0 ge 25 le 32
     end-list

    # C основе IN - Политики есть понятие as-path-list по которым выполняется изменение local preference атрибута BGP на IN
    # На больший
    xpl as-path-list ASP-UPSTR-RETN-HIGH
     ! ER-Telecom Holding
     origin '9049'
     end-list
    # На меньший
    xpl as-path-list ASP-UPSTR-RETN-LOW
     end-list

    # Теперь переходим к IN FV политике
    xpl route-filter RP-UPSTR-RETN-IN
     ! -- Upstream RETN AS9002 inbound --
     ! Prefix classification community
     apply community {9000:1001} additive
     if ip route-destination in PL-IPV4-BOGON then
      ! Do not accept bogons
      refuse
     elseif ip route-destination in PL-IPV4-LONG then
      ! Do not accept longer 25 to 32
      refuse
     elseif as-path in ASP-BOGONS then
      ! Do not accept AS BOGONS
      refuse
     elseif as-path in ASP-UPSTR-RETN-HIGH then
      ! More preffered AS
      apply local-preference 120
      approve
     elseif as-path in ASP-UPSTR-RETN-LOW then
      ! Less preffered AS
      apply local-preference 90
      approve
     else
      ! Everything else
      apply local-preference 110
      approve
     endif
     end-filter
    
    # Общий роут фильтр которые выполняет если есть общее UPSTREAM-community
    xpl route-filter RP-UPSTR-OUT-COMMUNITY
     ! -- Upstream outbound community policy --
     ! AS prepend control communities
     if community matches-any {9000:1990} then
      refuse
     elseif community matches-any {9000:1991} then
      apply as-path 9000 1 additive
      approve
     elseif community matches-any {9000:1992} then
      apply as-path 9000 2 additive
      approve
     elseif community matches-any {9000:1994} then
      apply as-path 9000 4 additive
      approve
     elseif community matches-any {9000:1996} then
      apply as-path 9000 6 additive
      approve
     else
      approve
     endif
     end-filter

    # Сформируем персональный RF для конкретного FV-аплинка и реакции на BGP Community в сторону него.
    xpl route-filter RP-UPSTR-RETN-OUT-COMMUNITY
     ! -- Upstream RETN outbound community policy --
     call route-filter RP-UPSTR-OUT-COMMUNITY
     ! AS prepend control communities
     if community matches-any {9000:1010} then
      refuse
     elseif community matches-any {9000:1011} then
      apply as-path 9000 1 additive
      approve
     elseif community matches-any {9000:1012} then
      apply as-path 9000 2 additive
      approve
     elseif community matches-any {9000:1013} then
      apply as-path 9000 3 additive
      approve
     elseif community matches-any {9000:1014} then
      apply as-path 9000 4 additive
      approve
     elseif community matches-any {9000:1015} then
      apply as-path 9000 5 additive
      approve
     elseif community matches-any {9000:1016} then
      apply as-path 9000 6 additive
      approve
     else
      approve
     endif
     end-filter

    # Теперь сама политика OUT FV
    xpl route-filter RP-UPSTR-RETN-OUT
     ! -- Upstream RETN AS9002 outbound --
     if ip route-destination in PL-IPV4-BLACKHOLE or community matches-any {9000:666} then
      ! Tell uplink to blackhole some prefixes
      apply community {9002:666} additive
     elseif ip route-destination in PL-IPV4-BOGON then
      ! Do not advertise bogons
      refuse
     elseif as-path in ASP-AS9000 and ip route-destination in PL-ADV-AS9000 then
      approve
     elseif as-path in ASP-AS9000-CLIENTS then
      apply as-path 9000 1 additive
      approve
     else
     ! Drop everything else
      refuse
     endif
     end-filter
    

QinQ-терминация интерфейсов:

::

    # Router A
    interface Eth-Trunk51.99999999
     ip address 10.119.0.1 255.255.255.252
     encapsulation qinq-termination rt-protocol
     qinq termination pe-vid 272 ce-vid 100
     arp broadcast enable

    # Router B
    interface Eth-Trunk51.99999999
     ip address 10.119.0.2 255.255.255.252
     encapsulation qinq-termination rt-protocol
     qinq termination pe-vid 272 ce-vid 100
     arp broadcast enable

VRRP:

::

    # Router A
    interface Eth-Trunk51.597
     vlan-type dot1q 597
     mtu 9198
     ip address 10.119.0.1 255.255.255.248
     statistic enable
     vrrp vrid 1 virtual-ip 10.119.0.6

    # Router B - master
    interface Eth-Trunk51.597
     vlan-type dot1q 597
     mtu 9198
     ip address 10.119.0.2 255.255.255.248
     statistic enable
     vrrp vrid 1 virtual-ip 10.119.0.6
     vrrp vrid 1 priority 120
     vrrp vrid 1 preempt-mode timer delay 20
     vrrp recover-delay 20

Коммутаторы CloudEngine некоторые тоже имеют VRP8, данный кейс был описан используя коммутаторы Huawei CE8861-4C-EI.
Покажу процесс сборки M-LAG:

::
    
    # Создаем VPN-instance
    ip vpn-instance management
     ipv4-family
     description OoBM
    quit
    # Настраиваем MGMT-интерфейс для подключения
    interface MEth0/0/0
     description << OoBM >>
     ip binding vpn-instance management
     ip address 10.10.255.1 255.255.255.0
    quit
    # Создаем статический маршрут для полного доступа к сети управления
    ip route-static vpn-instance management 10.10.192.0 255.255.192.0 10.10.255.254

    # M-LAG
    stp bridge-address 0001-0001-0001
    stp mode rstp
    stp v-stp enable
    # Со стороны коммутатор master указывается priority, по умолчанию 100
    dfs-group 1
     source ip 10.10.255.1 vpn-instance management peer 10.10.255.2
     priority 200
    quit
    # Собираем peer-link
    interface Eth-Trunk1
     trunkport 100GE 1/3/8
     trunkport 100GE 1/4/8
     mode lacp-static
     peer-link 1

    # Подключение устройства в рамках M-LAG
    interface Eth-Trunk2
     port link-type trunk
     port trunk allow-pass vlan 5
     stp bpdu-filter enable
     mode lacp-static
     dfs-group 1 m-lag 2