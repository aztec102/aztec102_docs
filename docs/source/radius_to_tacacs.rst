Меняем обувь - Radius -> Tacacs
===============================

############################################################
Шаг 0: Необходимо изучить в каком виде живет пароль в Radius
############################################################

В моем случае использовался хэширование без соли. Примерно так - SHA2-Password := hash

Генерация пароля имеет вид

:: 

    import hashlib
    password = "pass"
    hashlib.sha512(password.encode()).hexdigest()
    
Как выясняется далее данный пароль в том виде в котором он указан не перенести в Tacacs. 

###################################
Шаг 1: Генерируем пароль для Tacacs
###################################

::

    # Поскольку старый crypt скоро deprecated - используем новую либу
    import crypt_r
    password = "pass"
    crypt_r.crypt(password, "$6$" + "salt")

#############################################
Шаг 2: Как правильно готовить такакс под ключ
#############################################

Посолили, поперчили, едем дальше. Будем билдить такакс используя к окружении дистрибутив Debian 12.

::

    cd /opt/
    git clone https://github.com/MarcJHuber/event-driven-servers.git
    apt install gcc libpcre2-dev libfreeradius-dev libradcli-dev libssl-dev curl libc-ares-dev libcurl4-openssl-dev libldap-dev zlib1g-dev libpam0g-dev libsctp-dev libnet-ip-perl libauthen-tacacsplus-perl
    cd ./event-driven-servers
    configure
    make
    make install

################################
Шаг 3: Готовим конфиг для такакс
################################

Учитывая зоопарк железа, готовим конфиг для Cisco IOS, Cisco Nexus, JunOS, Huawei VRP. Нексус будем обманывать за счет отдельного списка с IP-адресами, т.к. если передавать атрибут в IOS - авторизация ломается.

::

    #!/usr/local/sbin/tac_plus
    id = spawnd {
        listen = { port = 49 }
    }

    id = tac_plus {
        authentication log = /var/log/tac_plus/authc/%Y/%m/%d.log
        authorization log = /var/log/tac_plus/authz/%Y/%m/%d.log
        accounting log = /var/log/tac_plus/acct/%Y/%m/%d.log
        retire limit = 1000
        mavis module = external {
            exec = /usr/local/lib/mavis/mavis_tacplus_passwd.pl
        }
        login backend = mavis
        pap backend = mavis
        host = nexus {
            address = 10.10.10.1
            address = 10.10.10.2
        }
        host = world {
            key = keystring
            address = 10.0.0.0/8
            address = 192.168.0.0/16
        }
        group = engineer {
            default service = permit
            service = shell {
                default command = permit
                default attribute = permit
                set priv-lvl = 15
            }
            service = junos-exec {
                set local-user-name = admin
            }
            member = nexus-engineer@nexus
        }
        group = nexus-engineer {
            default service = permit
            service = shell {
                default command = permit
                default attribute = permit
                set priv-lvl = 15
                set shell:roles="\"network-admin\""
            }
        }
        user = admin {
            password {
                login = crypt $6$salt$hashpassword
            }
            member = engineer@world
        }
    }

######################################
Шаг 4: Ограничения для учетных записей
######################################

Рассмотрим высокий уровень привелегий и ограничения команд к выполнению

::

    group = senior {
      default service = permit
      service = shell {
        set priv-lvl = 10
        default command = permit
        cmd = reload { deny .* }
        cmd = reboot { deny .* }
        cmd = isis { deny .* }
        cmd = bgp { deny .* }
        cmd = router { deny .* }
        cmd = clear { deny /^(access-list|ip\saccess-list)/ }
        cmd = access-list { deny .* }
        cmd = acl { deny .* }
        cmd = aaa { deny .* }
        cmd = ip { deny /^access-list/ }
        message deny = "denied by t+"
      }
    }

Рассмотрим пример создания сервисной учетной записи, которой разрешены только определенные команды

::

    group = service {
      default service = permit
      service = shell {
        set priv-lvl = 5
        cmd = show { permit .* }
        cmd = display { permit .* }
        cmd = screen-length { permit .* }
        cmd = terminal { permit .* }
        cmd = ping { permit .* }
        cmd = traceroute { permit .* }
        cmd = tracert { permit .* }
        message deny = "denied by t+"
      }
      service = junos-exec {
            set local-user-name = service
      }
      member = nexus-service@nexus
    }

####################################
Шаг 5: Конфигурации для оборудования
####################################

Huawei CE VRP8

::

    hwtacacs server template tacacs-oob
     hwtacacs server authentication 10.226.255.229 vpn-instance management
     hwtacacs server authentication 10.226.255.230 vpn-instance management secondary
     hwtacacs server authorization 10.226.255.229 vpn-instance management
     hwtacacs server authorization 10.226.255.230 vpn-instance management secondary
     hwtacacs server accounting 10.226.255.229 vpn-instance management
     hwtacacs server accounting 10.226.255.230 vpn-instance management secondary
     hwtacacs server shared-key cipher keystring
     hwtacacs server user-name domain-excluded
    quit
 
    aaa
     authentication-scheme default
      authentication-mode local hwtacacs
     authorization-scheme tacacs-oob
      authorization-mode local hwtacacs
      authorization-cmd 5 hwtacacs local
      authorization-cmd 10 hwtacacs local
      authorization-cmd 15 hwtacacs local
     accounting-scheme default
      accounting-mode hwtacacs
     domain default_admin
      authorization-scheme tacacs-oob
      hwtacacs server tacacs-oob
     recording-scheme tacacs-oob
      recording-mode hwtacacs tacacs-oob
     #
     cmd recording-scheme tacacs-oob

Huawei S VRP2

::

    hwtacacs-server template tacacs-oob
     hwtacacs-server authentication 10.226.255.229 vpn-instance management
     hwtacacs-server authentication 10.226.255.230 vpn-instance management secondary
     hwtacacs-server authorization 10.226.255.229 vpn-instance management
     hwtacacs-server authorization 10.226.255.230 vpn-instance management secondary
     hwtacacs-server accounting 10.226.255.229 vpn-instance management
     hwtacacs-server accounting 10.226.255.230 vpn-instance management secondary
     hwtacacs-server source-ip 10.226.255.232
     hwtacacs-server shared-key cipher keystring
     hwtacacs-server timer response-timeout 10
     undo hwtacacs-server user-name domain-included
    
    aaa
     authentication-scheme default
      authentication-mode local hwtacacs
     authentication-scheme tacacs-oob
      authentication-mode local hwtacacs
     authorization-scheme tacacs-oob
      authorization-mode local hwtacacs
      authorization-cmd 3 hwtacacs local
      authorization-cmd 5 hwtacacs local
      authorization-cmd 7 hwtacacs local
      authorization-cmd 10 hwtacacs local
      authorization-cmd 15 hwtacacs local
     accounting-scheme tacacs-oob
      accounting-mode hwtacacs
      accounting start-fail online
     recording-scheme tacacs-oob
      recording-mode hwtacacs tacacs-oob
     cmd recording-scheme tacacs-oob
     domain default_admin
      authentication-scheme tacacs-oob
      accounting-scheme tacacs-oob
      authorization-scheme tacacs-oob
      hwtacacs-server tacacs-oob

Huawei NE VRP8

::
    
    hwtacacs-server template tacacs-oob
     hwtacacs-server authentication 10.226.255.229 vpn-instance management
     hwtacacs-server authentication 10.226.255.230 vpn-instance management secondary
     hwtacacs-server authorization 10.226.255.229 vpn-instance management
     hwtacacs-server authorization 10.226.255.230 vpn-instance management secondary
     hwtacacs-server accounting 10.226.255.229 vpn-instance management
     hwtacacs-server accounting 10.226.255.230 vpn-instance management secondary
     hwtacacs-server shared-key cipher keystring
     hwtacacs-server user-name original
    quit

    aaa
     authentication-scheme default
      authentication-mode local hwtacacs
     authorization-scheme tacacs-oob
      authorization-mode local hwtacacs
      authorization-cmd 5 hwtacacs local
      authorization-cmd 10 hwtacacs local
      authorization-cmd 15 hwtacacs local
     accounting-scheme default
      accounting-mode hwtacacs
      accounting start-fail online
     domain default_admin
      authorization-scheme tacacs-oob
      accounting-scheme default
      hwtacacs-server tacacs-oob
     recording-scheme tacacs-oob
      recording-mode hwtacacs tacacs-oob
     #
     cmd recording-scheme tacacs-oob

Cisco IOS

::

    tacacs server tacacs1-krk
     address ipv4 1.1.1.1
     key 7 keystring
     timeout 3
    tacacs server tacacs2-krk
     address ipv4 1.1.1.2
     key 7 keystring
     timeout 3

    aaa group server tacacs+ TACACS-GLOBAL
     server name tacacs1-krk
     server name tacacs2-krk
     ip tacacs source-interface Loopback0
    
    aaa authentication login default local group TACACS-GLOBAL
    aaa authentication enable default group TACACS-GLOBAL
    aaa authorization console
    aaa authorization exec default local group TACACS-GLOBAL if-authenticated 
    aaa authorization commands 3 default local group TACACS-GLOBAL 
    aaa authorization commands 5 default local group TACACS-GLOBAL 
    aaa authorization commands 7 default local group TACACS-GLOBAL 
    aaa authorization commands 10 default local group TACACS-GLOBAL 
    aaa authorization commands 15 default local group TACACS-GLOBAL 
    aaa accounting update newinfo
    aaa accounting commands 3 default start-stop group TACACS-GLOBAL
    aaa accounting commands 5 default start-stop group TACACS-GLOBAL
    aaa accounting commands 7 default start-stop group TACACS-GLOBAL
    aaa accounting commands 10 default start-stop group TACACS-GLOBAL
    aaa accounting commands 15 default start-stop group TACACS-GLOBAL

Cisco IOS - old version software

::

    tacacs-server host 1.1.1.1 key 7 keystring
    tacacs-server host 1.1.1.2 key 7 keystring
    
    aaa group server tacacs+ TACACS-GLOBAL
     server 1.1.1.1
     server 1.1.1.2
     ip tacacs source-interface Loopback0

    aaa authentication login default local group TACACS-GLOBAL
    aaa authentication enable default enable group TACACS-GLOBAL
    aaa authorization console
    aaa authorization exec default local group TACACS-GLOBAL if-authenticated 
    aaa authorization commands 3 default local group TACACS-GLOBAL 
    aaa authorization commands 5 default local group TACACS-GLOBAL 
    aaa authorization commands 7 default local group TACACS-GLOBAL 
    aaa authorization commands 10 default local group TACACS-GLOBAL 
    aaa authorization commands 15 default local group TACACS-GLOBAL 
    aaa accounting commands 3 default start-stop group TACACS-GLOBAL
    aaa accounting commands 5 default start-stop group TACACS-GLOBAL
    aaa accounting commands 7 default start-stop group TACACS-GLOBAL
    aaa accounting commands 10 default start-stop group TACACS-GLOBAL
    aaa accounting commands 15 default start-stop group TACACS-GLOBAL

D-Link - universal DES-series

::

    create authen server_host 10.226.255.229 protocol tacacs+ port 49 key "keystring" timeout 5 retransmit 1
    create authen server_host 10.226.255.230 protocol tacacs+ port 49 key "keystring" timeout 5 retransmit 1
    config authen server_group tacacs+ delete server_host 10.226.255.229 protocol tacacs+
    config authen server_group tacacs+ add server_host 10.226.255.229 protocol tacacs+
    config authen server_group tacacs+ delete server_host 10.226.255.230 protocol tacacs+
    config authen server_group tacacs+ add server_host 10.226.255.230 protocol tacacs+
    config authen_login default method tacacs+ local
    config authen_enable default method local_enable
    config authen application telnet login default
    config authen application telnet enable default
    config authen application ssh login default
    config authen application ssh enable default
    config authen parameter response_timeout 30
    config authen parameter attempt 3
    enable authen_policy

###################################
Шаг 6: Централизованное логирование
###################################

::

    authentication log = /var/log/tac_plus/authc/%Y/%m/%d.log
    authentication log = 10.10.10.254:514
    authorization log = /var/log/tac_plus/authz/%Y/%m/%d.log
    authorization log = 10.10.10.254:514
    accounting log = /var/log/tac_plus/acct/%Y/%m/%d.log
    accounting log = 10.10.10.254:514

    # OR
    log = mylog {
        destination = 169.254.0.23:514
    }
    authentication log = /var/log/tac_plus/authc/%Y/%m/%d.log
    authentication log = mylog
    authorization log = /var/log/tac_plus/authz/%Y/%m/%d.log
    authorization log = mylog
    accounting log = /var/log/tac_plus/acct/%Y/%m/%d.log
    accounting log = mylog


########################
Успех достигнут, балдеем
########################

.. image:: images/luck.jpg
    :width: 850