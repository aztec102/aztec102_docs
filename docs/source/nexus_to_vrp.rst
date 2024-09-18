Migrate Nexus to Huawei
=======================

Данная статья описывает миграцию в рамках оборудования с Cisco Nexus на Huawei S/CE

#############
Создание VLAN
#############

В формате Nexus всё просто, чтобы обозначить VLAN достаточно использовать параметр - name.

::

    vlan 10
      name management

В Huawei есть же две возможности - это name и description. Name - это короткое наименование которое можно использовать для работы с VLAN в системе. Description же это просто пометка VLAN, чтобы обозначить его по аналогии в name в Nexus.
Если вы просто зайдете во VLAN и присвоите ему description, то он не создастся в системе. Его обязательно нужно объявить через batch, чтобы создать его на железке.

.. note::

   Если у Вас используется M-LAG, то при создании VLAN на коммутаторе он автоматически подается в peer-link.

::

    vlan batch 10
    vlan 10
      description management
    

Теперь стоит разобраться с тем как происходит отображение прокинутости VLAN в Nexus и Huawei.
Nexus показывает то на каких интерфейсах он прокинут и дочерние интерфейсы входящие в Port-Channel.

::
    
    NEXUS# show vlan id 10
    
    VLAN Name                             Status    Ports
    ---- -------------------------------- --------- -------------------------------
    10   management                       active    Po3, Po20, Po33, Po47, Po50
                                                    Po4096, Eth1/3, Eth1/33, Eth1/34
                                                    Eth1/45, Eth1/46, Eth1/47
                                                    Eth1/49, Eth1/50

У Huawei ситуация же несколько иная, он отображает только основные интерфейсы, т.е. если VLAN подан в Eth-Trunk, он не покажется дочерние интерфейсы Eth-Trunk.

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

#################
Прокидывание VLAN
#################

На Nexus же приходится по старой памяти добавлять приставку add при прокидывании VLAN. У некоторых людей это уже доходит до автоматизма.

::

    switchport trunk allowed vlan add 10
    switchport trunk allowed vlan remove 10

На Huawei же будь то VRP8 или VRP5 добавлять приставку add не надо, потому что просто её там нет. Так же как и при удалении VLAN с интерфейса.

::

    port trunk allow-pass vlan 10
    undo port trunk allow-pass vlan 10


###########
Чекаем маки
###########

::

    NEXUS# show mac address-table vlan 10
    Legend: 
            * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
            age - seconds since last seen,+ - primary entry using vPC Peer-Link,
            (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
       VLAN     MAC Address      Type      age     Secure NTFY Ports
    ---------+-----------------+--------+---------+------+----+------------------
    +   10     0003.b988.da72   dynamic  0         F      F    Po3
    +   10     0003.ea0e.5d06   dynamic  0         F      F    Po3
    +   10     0003.ea13.b420   dynamic  0         F      F    Po3
    +   10     0010.7468.4046   dynamic  0         F      F    Po3
    +   10     0013.4666.5405   dynamic  0         F      F    Po3
    +   10     001c.a300.3d10   dynamic  0         F      F    Po3
    +   10     001c.f0d3.d74b   dynamic  0         F      F    Po3
    +   10     001e.5870.5b00   dynamic  0         F      F    Po3
    +   10     001e.5898.fe39   dynamic  0         F      F    Po3
    

::

    # VRP5
    <HUAWEI>dis mac-address vlan 142
    -------------------------------------------------------------------------------
    MAC Address    VLAN/VSI/BD                       Learned-From        Type      
    -------------------------------------------------------------------------------
    0001-e88b-d8fd 142/-/-                           Eth-Trunk4          dynamic   
    0001-e8d8-4c7e 142/-/-                           Eth-Trunk4          dynamic   
    0004-968f-aa0d 142/-/-                           Eth-Trunk4          dynamic   
    0004-968f-aa20 142/-/-                           Eth-Trunk4          dynamic   
    000c-2971-0ad2 142/-/-                           Eth-Trunk4          dynamic   
    000c-29e4-7418 142/-/-                           Eth-Trunk4          dynamic   
    0018-18b8-4aff 142/-/-                           Eth-Trunk4          dynamic   
    
    # VRP8
    <HUAWEI>dis mac-address vlan 1488
    Flags: * - Backup  
           # - forwarding logical interface, operations cannot be performed based 
               on the interface.
    BD   : bridge-domain   Age : dynamic MAC learned time in seconds
    -------------------------------------------------------------------------------
    MAC Address    VLAN/VSI/BD   Learned-From        Type                Age
    -------------------------------------------------------------------------------
    0000-5e00-0101 1488/-/-      Eth-Trunk64         dynamic             718122
    000b-829b-8315 1488/-/-      Eth-Trunk64         dynamic                665
    000c-29a2-0c5c 1488/-/-      Eth-Trunk64         dynamic             718122
    0015-5d21-e60e 1488/-/-      Eth-Trunk64         dynamic                889
    0015-5d21-e61b 1488/-/-      Eth-Trunk64         dynamic                723
    0015-5d43-0201 1488/-/-      Eth-Trunk64         dynamic             718122
    0015-5d43-0213 1488/-/-      Eth-Trunk64         dynamic                924


##################
Диагностика портов
##################

::

    NEXUS# show int ethernet1/9 transceiver details 
    Ethernet1/9
        transceiver is present
        type is 10Gbase-ER
        name is OEM
        part number is SFP+ CWDM-70
        revision is A
        serial number is SD1D460005
        nominal bitrate is 10300 MBit/sec
        Link length supported for 9/125um fiber is 70 km
        cisco id is 3
        cisco extended id number is 4
    
               SFP Detail Diagnostics Information (internal calibration)
      ----------------------------------------------------------------------------
                    Current              Alarms                  Warnings
                    Measurement     High        Low         High          Low
      ----------------------------------------------------------------------------
      Temperature   53.08 C        90.00 C    -15.00 C     85.00 C      -10.00 C
      Voltage        3.23 V         3.59 V      3.00 V      3.50 V        3.04 V
      Current       78.23 mA      120.00 mA    30.00 mA   110.00 mA      35.00 mA
      Tx Power       0.63 dBm       6.99 dBm   -1.99 dBm    5.99 dBm     -1.00 dBm
      Rx Power     -13.01 dBm      -8.01 dBm  -30.00 dBm   -9.03 dBm    -30.00 dBm
      Transmit Fault Count = 0
      ----------------------------------------------------------------------------
      Note: ++  high-alarm; +  high-warning; --  low-alarm; -  low-warning


::

    <HUAWEI>dis int 25GE 1/1/6 transceiver verbose 
    
     25GE1/1/6 transceiver information:
    -------------------------------------------------------------------
     Common information:
       Transceiver Type                      :10GBASE_LR
       Connector Type                        :LC
       Wavelength (nm)                       :1330
       Transfer Distance (m)                 :20000(9um/125um SMF)
       Digital Diagnostic Monitoring         :YES
       Vendor Name                           :OEM
       Vendor Part Number                    :SFP+ BIDI-20
    -------------------------------------------------------------------
     Manufacture information:
       Manu. Serial Number                   :PJBN780037
       Manufacturing Date                    :2018-11-27
       Vendor Name                           :OEM
    -------------------------------------------------------------------
     Alarm information:
    -------------------------------------------------------------------
     Diagnostic information: 
       Temperature (Celsius)                 :17.96
       Voltage (V)                           :3.26
       Bias Current (mA)                     :23.31
       Bias High Threshold (mA)              :90.00
       Bias Low Threshold (mA)               :1.00
       Current RX Power (dBm)                :-12.49
       Default RX Power High Threshold (dBm) :0.00
       Default RX Power Low Threshold (dBm)  :-16.00
       Current TX Power (dBm)                :0.85
       Default TX Power High Threshold (dBm) :5.50
       Default TX Power Low Threshold (dBm)  :-5.50
    -------------------------------------------------------------------
