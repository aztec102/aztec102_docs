Huawei MPLS
===========

###############
Топология с IGP
###############

.. image:: images/huawei_vpls.png

###########################
LDP прямые без лишних хопов
###########################

.. image:: images/huawei_vpls_no_igp.png


DC

::


    mpls ldp remote-peer 10.232.255.111
     remote-ip 10.232.255.111

    mpls ldp remote-peer 10.232.255.112
     remote-ip 10.232.255.112

    vsi 111
     pwsignal ldp
      vsi-id 111
      peer 10.232.255.111
      peer 10.232.255.112
      protect-group CITY
       protect-mode pw-redundancy master
       reroute delay 300
       peer 10.232.255.111 preference 5
       peer 10.232.255.112 preference 10
     encapsulation ethernet
     ignore-ac-state

BR1/BR2

::


    mpls ldp remote-peer 10.124.254.32
     remote-ip 10.124.254.32

    vsi 111
     pwsignal ldp
      vsi-id 111
      peer 10.124.254.32
     encapsulation ethernet
     ignore-ac-state
