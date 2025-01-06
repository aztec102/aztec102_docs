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