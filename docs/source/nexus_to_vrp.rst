Migrate Nexus to Huawei
=======================

Данная статья описывает миграцию в рамках оборудования с Cisco Nexus на Huawei S/CE

###############
Создание вланов
###############

В формате нексуса всё просто, чтобы обозначить влан достаточно использовать параметр - name.

::

    vlan 10
      name management

В Huawei есть же две возможности - это name и description. Name - это короткое наименование которое можно использовать для работы с вланом в системе. Description же это просто пометка влана, чтобы обозначить его по аналогии в name в Nexus.
Если вы просто зайдете во влан и присвоите ему description, то он не создастся в системе. Его обязательно нужно объявить через batch, чтобы создать его на железке.

::

    vlan batch 10
    vlan 10
      description management
    

Теперь стоит разобраться с тем как происходит отображение прокинутости вланов в Nexus и Huawei.
Nexus показывает то на каких интерфейсах он прокинут и дочерние интерфейсы входящие в Port-Channel.

::
    
    NEXUS# show vlan id 10
    
    VLAN Name                             Status    Ports
    ---- -------------------------------- --------- -------------------------------
    10   management                       active    Po3, Po20, Po33, Po47, Po50
                                                    Po4096, Eth1/3, Eth1/33, Eth1/34
                                                    Eth1/45, Eth1/46, Eth1/47
                                                    Eth1/49, Eth1/50

У Huawei ситуация же несколько иная, он отображает только основные интерфейсы, т.е. если влан подан в Eth-Trunk, он не покажется дочерние интерфейсы Eth-Trunk.

::

    [~HUAWEI]dis vlan 10
    --------------------------------------------------------------------------------
    U: Up;         D: Down;         TG: Tagged;         UT: Untagged;
    MP: Vlan-mapping;               ST: Vlan-stacking;
    #: ProtocolTransparent-vlan;    *: Management-vlan;
    MAC-LRN: MAC-address learning;  STAT: Statistic;
    BC: Broadcast; MC: Multicast;   UC: Unknown-unicast;
    FWD: Forward;  DSD: Discard;
    --------------------------------------------------------------------------------
    
    VID          Ports                                                          
    --------------------------------------------------------------------------------
     10          TG:Eth-Trunk1(U)   Eth-Trunk41(U)  Eth-Trunk61(U)  Eth-Trunk65(U)  
                                                                                    
    VID  Type     Status  Property  MAC-LRN STAT    BC  MC  UC  Description
    --------------------------------------------------------------------------------
     10  common   enable  default   enable  disable FWD FWD FWD management          

###################
Прокидывание вланов
###################

На Nexus же приходится по старой памяти добавлять приставку add при прокидывании влана. У некоторых людей это уже доходит до автоматизма.

::

    switchport trunk allowed vlan add 10
    switchport trunk allowed vlan remove 10

На Huawei же будь то VRP8 или VRP5 добавлять приставку add не надо, потому что просто её там нет. Так же как и при удалении влана с интерфейса.

::

    port trunk allow-pass vlan 10
    undo port trunk allow-pass vlan 10