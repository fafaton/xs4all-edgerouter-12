# Overview

### Links
* [XS4ALL VLAN details](https://www.xs4all.nl/service/installeren/internet/instellingen-andere-modems/)
* [XS4ALL DNS Resolvers](https://www.xs4all.nl/service/installeren/internet/overzicht-serverinstellingen/)

### Items used for fiber connection.
* [1000BASE-BX BiDi SFP 1310nm-TX/1490nm-RX](https://www.fs.com/de-en/products/37922.html)
* [SC/APC to SC/APC Simplex Single Mode Fibre Optic](https://www.fs.com/de-en/products/48491.html)
* [LC UPC to SC APC Simplex OS2 Single Mode PVC](https://www.fs.com/de-en/products/62905.html)


### VLANS
* VLAN 4 (TV)
* VLAN 6 (Internet)

### Interfaces
* switch0 = LAN downstream to your switched network. 
  * used ports eth1-7 as switch port
* eth11 = SFP fiber connection to FTU
* eth11.4 = VLAN for IPTV
* eth11.6 = VLAN for Internet 
* vtun0 = OpenVPN connection

### Static IP Mappings (DHCP)
* 192.168.10.1 - switch0 (LAN GW)
* 192.168.10.20 - cable box 1
* 192168.10.21 - cable box 2
* 192.168.10.4 - Linux box (syslog/tftp)

*I routed netflix to OpenVPN.*

# Script
```bash
sudo vbash -c 'cat > /etc/dhcp3/dhclient-exit-hooks.d/rfc3442-classless-routes' << EOF
# set classless routes based on the format specified in RFC3442
# e.g.:
#   new_rfc3442_classless_static_routes='24 192 168 10 192 168 1 1 8 10 10 17 66 41'
# specifies the routes:
#   192.168.10.0/24 via 192.168.1.1
#   10.0.0.0/8 via 10.10.17.66.41

RUN="yes"


if [ "$RUN" = "yes" ]; then
    if [ -n "$new_rfc3442_classless_static_routes" ]; then
        if [ "$reason" = "BOUND" ] || [ "$reason" = "REBOOT" ]; then

            set -- $new_rfc3442_classless_static_routes

            while [ $# -gt 0 ]; do
                net_length=$1
                via_arg=''

                case $net_length in
                    32|31|30|29|28|27|26|25)
                        if [ $# -lt 9 ]; then
                            return 1
                        fi
                        net_address="${2}.${3}.${4}.${5}"
                        gateway="${6}.${7}.${8}.${9}"
                        shift 9
                        ;;
                    24|23|22|21|20|19|18|17)
                        if [ $# -lt 8 ]; then
                            return 1
                        fi
                        net_address="${2}.${3}.${4}.0"
                        gateway="${5}.${6}.${7}.${8}"
                        shift 8
                        ;;
                    16|15|14|13|12|11|10|9)
                        if [ $# -lt 7 ]; then
                            return 1
                        fi
                        net_address="${2}.${3}.0.0"
                        gateway="${4}.${5}.${6}.${7}"
                        shift 7
                        ;;
                    8|7|6|5|4|3|2|1)
                        if [ $# -lt 6 ]; then
                            return 1
                        fi
                        net_address="${2}.0.0.0"
                        gateway="${3}.${4}.${5}.${6}"
                        shift 6
                        ;;
                    0)    # default route
                        if [ $# -lt 5 ]; then
                            return 1
                        fi
                        net_address="0.0.0.0"
                        gateway="${2}.${3}.${4}.${5}"
                        shift 5
                        ;;
                    *)    # error
                        return 1
                        ;;
                esac

                # take care of link-local routes
                if [ "${gateway}" != '0.0.0.0' ]; then
                    via_arg="via ${gateway}"
                fi

                # set route (ip detects host routes automatically)
                ip -4 route add "${net_address}/${net_length}" \
                    ${via_arg} dev "${interface}" >/dev/null 2>&1
            done
        fi
    fi
fi
EOF
```
```bash
sudo chmod +x /etc/dhcp3/dhclient-exit-hooks.d/rfc3442-classless-routes
```

# Config
```
firewall {
    all-ping enable
    broadcast-ping disable
    group {
        address-group RouteThroughVPN {
        }
    }
    ipv6-name WANv6_IN {
        default-action drop
        description "WAN inbound traffic forwarded to LAN"
        rule 10 {
            action accept
            description "Allow established/related sessions"
            state {
                established enable
                related enable
            }
        }
        rule 20 {
            action drop
            description "Drop invalid state"
            log enable
            state {
                invalid enable
            }
        }
    }
    ipv6-name WANv6_LOCAL {
        default-action drop
        description "WAN inbound traffic to the router"
        rule 10 {
            action accept
            description "Allow established/related sessions"
            state {
                established enable
                related enable
            }
        }
        rule 20 {
            action drop
            description "Drop invalid state"
            state {
                invalid enable
            }
        }
        rule 30 {
            action accept
            description "Allow IPv6 icmp"
            protocol ipv6-icmp
        }
        rule 40 {
            action accept
            description "allow dhcpv6"
            destination {
                port 546
            }
            protocol udp
            source {
                port 547
            }
        }
    }
    ipv6-receive-redirects disable
    ipv6-src-route disable
    ip-src-route disable
    log-martians enable
    modify RouteThroughVPN {
        rule 1 {
            action modify
            destination {
                group {
                    address-group RouteThroughVPN
                }
            }
            modify {
                table 11
            }
            protocol all
        }
    }
    name WAN_IN {
        default-action drop
        description "WAN to Internal"
        rule 10 {
            action accept
            description "Allow established/related"
            log disable
            protocol all
            state {
                established enable
                invalid disable
                new disable
                related enable
            }
        }
        rule 20 {
            action accept
            description "Allow IPTV Multicast Group"
            destination {
                address 239.0.0.0/8
            }
            log disable
            protocol udp
            source {
                address 10.0.0.0/8
            }
        }
        rule 40 {
            action drop
            description "Drop invalid state"
            log enable
            protocol all
            state {
                established disable
                invalid enable
                new disable
                related disable
            }
        }
    }
    name WAN_LOCAL {
        default-action drop
        description "WAN to router"
        rule 10 {
            action accept
            description "Allow established/related"
            log disable
            protocol all
            state {
                established enable
                invalid disable
                new disable
                related enable
            }
        }
        rule 20 {
            action accept
            description "Allow icmp"
            log disable
            protocol icmp
        }
        rule 30 {
            action drop
            description "Drop invalid state"
            log enable
            protocol all
            state {
                established disable
                invalid enable
                new disable
                related disable
            }
        }
    }
    options {
        mss-clamp {
            mss 1412
        }
    }
    receive-redirects disable
    send-redirects enable
    source-validation disable
    syn-cookies enable
}
interfaces {
    ethernet eth0 {
        description Local
        duplex auto
        speed auto
    }
    ethernet eth1 {
        description Local
        duplex auto
        speed auto
    }
    ethernet eth2 {
        description Local
        duplex auto
        speed auto
    }
    ethernet eth3 {
        description Local
        duplex auto
        speed auto
    }
    ethernet eth4 {
        description Local
        duplex auto
        speed auto
    }
    ethernet eth5 {
        description Local
        duplex auto
        speed auto
    }
    ethernet eth6 {
        description Local
        duplex auto
        speed auto
    }
    ethernet eth7 {
        description Local
        duplex auto
        speed auto
    }
    ethernet eth8 {
        description Local
        duplex auto
        speed auto
    }
    ethernet eth9 {
        duplex auto
        poe {
            output off
        }
        speed auto
    }
    ethernet eth10 {
        duplex auto
        speed auto
    }
    ethernet eth11 {
        description "eth11 - FTTH"
        duplex full
        mtu 1512
        speed 1000
        vif 4 {
            address dhcp
            description "eth11.4 - IPTV"
            dhcp-options {
                client-option "send vendor-class-identifier &quot;IPTV_RG&quot;;"
                client-option "request subnet-mask, routers, rfc3442-classless-static-routes;"
                default-route no-update
                default-route-distance 210
                name-server update
            }
        }
        vif 6 {
            description "eth11.6 - Internet"
            mtu 1508
            pppoe 0 {
                default-route auto
                dhcpv6-pd {
                    no-dns
                    pd 0 {
                        interface switch0 {
                            prefix-id :1
                            service slaac
                        }
                        prefix-length /48
                    }
                    prefix-only
                    rapid-commit disable
                }
                firewall {
                    in {
                        ipv6-name WANv6_IN
                        name WAN_IN
                    }
                    local {
                        ipv6-name WANv6_LOCAL
                        name WAN_LOCAL
                    }
                }
                idle-timeout 180
                ipv6 {
                    address {
                        autoconf
                    }
                    dup-addr-detect-transmits 1
                    enable {
                    }
                }
                mtu 1500
                name-server auto
                password xs4all
                user-id FB5490@xs4all.nl
            }
        }
    }
    loopback lo {
    }
    openvpn vtun0 {
        config-file /config/auth/myopenvpn.ovpn
    }
    switch switch0 {
        address 192.168.10.1/24
        description "Local LAN"
        firewall {
            in {
                modify RouteThroughVPN
            }
        }
        ipv6 {
            dup-addr-detect-transmits 1
            router-advert {
                cur-hop-limit 64
                link-mtu 0
                managed-flag false
                max-interval 600
                name-server 2001:4860:4860::8888
                name-server 2001:4860:4860::8844
                other-config-flag false
                prefix ::/64 {
                    autonomous-flag true
                    on-link-flag true
                    valid-lifetime 2592000
                }
                radvd-options "RDNSS 2001:4860:4860::8888 2001:4860:4860::8844 {};"
                reachable-time 0
                retrans-timer 0
                send-advert true
            }
        }
        mtu 1500
        switch-port {
            interface eth1 {
            }
            interface eth2 {
            }
            interface eth3 {
            }
            interface eth4 {
            }
            interface eth5 {
            }
            interface eth6 {
            }
            interface eth7 {
            }
            vlan-aware disable
        }
    }
}
port-forward {
    auto-firewall enable
    hairpin-nat enable
    lan-interface switch0
    rule 1 {
        description https
        forward-to {
            address 192.168.10.4
            port 443
        }
        original-port 443
        protocol tcp
    }
    wan-interface pppoe0
}
protocols {
    igmp-proxy {
        interface eth11.4 {
            alt-subnet 0.0.0.0/0
            role upstream
            threshold 1
        }
        interface switch0 {
            alt-subnet 192.168.10.20/32
            alt-subnet 192.168.10.21/32
            role downstream
            threshold 1
        }
    }
    static {
        interface-route6 ::/0 {
            next-hop-interface pppoe0 {
            }
        }
        route 213.75.112.0/21 {
            next-hop 10.170.160.1 {
            }
        }
        table 11 {
            interface-route 0.0.0.0/0 {
                next-hop-interface vtun0 {
                }
            }
        }
    }
}
service {
    dhcp-server {
        disabled false
        global-parameters "option vendor-class-identifier code 60 = string;"
        global-parameters "option broadcast-address code 28 = ip-address;"
        hostfile-update disable
        shared-network-name LAN {
            authoritative enable
            subnet 192.168.10.0/24 {
                default-router 192.168.10.1
                dns-server 8.8.8.8
                dns-server 8.8.4.4
                lease 86400
                start 192.168.10.50 {
                    stop 192.168.10.200
                }
                static-mapping Cable-TV-Box {
                    ip-address 192.168.10.20
                    mac-address 00:XX:XX:XX:XX:XX
                }
                static-mapping Cable-TV-Box-2 {
                    ip-address 192.168.10.21
                    mac-address 00:XX:XX:XX:XX:XX
                }
            }
        }
        static-arp disable
        use-dnsmasq enable
    }
    dns {
        forwarding {
            cache-size 150
            listen-on eth8
            listen-on switch0
            listen-on eth0
            options listen-address=192.168.10.1
            options ipset=/netflix.com/nflxvideo.net/RouteThroughVPN
        }
    }
    gui {
        http-port 80
        https-port 443
        older-ciphers enable
    }
    nat {
        rule 5000 {
            description IPTV
            destination {
                address 213.75.112.0/21
            }
            log enable
            outbound-interface eth11.4
            protocol all
            type masquerade
        }
        rule 5010 {
            description "XS4ALL Internet"
            log disable
            outbound-interface pppoe0
            protocol all
            source {
                address 192.168.10.0/24
            }
            type masquerade
        }
        rule 5011 {
            description "Masquerade for vtun0"
            log disable
            outbound-interface vtun0
            type masquerade
        }
    }
    ssh {
        port 22
        protocol-version v2
    }
    unms {
        disable
    }
}
system {
    analytics-handler {
        send-analytics-report false
    }
    config-management {
        commit-archive {
            location tftp://192.168.10.4/edgerouter
        }
    }
    crash-handler {
        send-crash-report false
    }
    host-name EdgeRouter-12
    login {
        user fafachu {
            authentication {
                encrypted-password xxxxx
            }
            level admin
        }
    }
    name-server 194.109.6.66
    name-server 194.109.9.99
    name-server 8.8.8.8
    name-server 8.8.4.4
    name-server 2001:888:0:6::66
    name-server 2001:888:0:9::99
    name-server 2001:4860:4860::8888
    name-server 2001:4860:4860::8844
    ntp {
        server 0.ubnt.pool.ntp.org {
        }
        server 1.ubnt.pool.ntp.org {
        }
        server 2.ubnt.pool.ntp.org {
        }
        server 3.ubnt.pool.ntp.org {
        }
    }
    offload {
        hwnat disable
        ipv4 {
            forwarding enable
            pppoe enable
            vlan enable
        }
        ipv6 {
            forwarding enable
            pppoe enable
        }
    }
    package {
        repository stretch {
            components "main contrib non-free"
            distribution stretch
            password ""
            url http://http.us.debian.org/debian
            username ""
        }
    }
    syslog {
        global {
            facility all {
                level notice
            }
            facility protocols {
                level debug
            }
        }
        host 192.168.10.4 {
            facility all {
                level info
            }
        }
    }
    task-scheduler {
        task updateIPTVroute {
            executable {
                path /config/scripts/tvroute.sh
            }
            interval 15m
        }
    }
    time-zone Europe/Amsterdam
    traffic-analysis {
        dpi enable
        export enable
    }
}
```
