# Router hinter Router ohne NAT

Wenn man einen eigenen Router hinter einem Provider Router betreibt sieht das
oft in etwa so aus:

    .---------------.    .---------------------------.---.    .-------------------------------.---.
    |      LAN      |    |          OpenWrt          |   |    |        Provider Router        |   |
    |---------------|    |---------------------------|   |    |-------------------------------|   |
    |               |    |                           | N |    |               ..PUBLIC IPv4/6 | N |    ************
    | 192.168.0.170 |--->| 192.168.0.1/24 (DHCP)     | A |--->|               ...IPv4 in IPv6 | A |--->  INTERNET
    |               |    | (LAN)                     | T |    |                               | T |    ************
    |               |    |           192.168.128.136 |   |    | 192.168.128.1/24 (DHCP)       |   |
    |               |    |                     (WAN) |   |    |                               |   |
    '---------------'    '---------------------------'---'    '-------------------------------'---'

Da das zweite NAT unnötig ist und eventuell auch Hardwarebeschleunigungen verhindert
ist folgendes Setup wünschenswert:

    .---------------.    .---------------------------.    .-------------------------------.---.
    |      LAN      |    |          OpenWrt          |    |        Provider Router        |   |
    |---------------|    |---------------------------|    |-------------------------------|   |
    |               |    |                           |    |               ..PUBLIC IPv4/6 | N |        ************
    | 192.168.0.170 |--->| 192.168.0.1/24 (DHCP)     |--->|               ...IPv4 in IPv6 | A |------->  INTERNET
    |               |    | (LAN)                     |    |                               | T |        ************
    |               |    |               192.168.0.2 |    | 192.168.0.1/24 (DHCP)         |   |
    |               |    |                     (WAN) |    |                               |   |
    '---------------'    '---------------------------'    '-------------------------------'---'

Das ganze funktioniert relativ einfach. Als erstes gilt es, dass gemeinsame
Netz, in diesem Beispiel 192.168.0.0/24 in 2 Teile zu splitten.

Auf der LAN Seite ist dies einfach:

    A1: 192.168.0.1/24
    R1: 192.168.128.0/24 dev LAN

Auf der WAN Seite sieht es etwas komplizierter aus:

    A2: 192.168.0.2/32
    R2: 192.168.0.1 dev WAN
    R3: default via 192.168.0.1 dev WAN

Pakete an den Provider Router und an die default Route gehen über Route R2 bzw.
R3 raus, da entweder die spezifischere Route oder die einzige Route ist. Die IP
Konfiguration auf der WAN Seite sollte statisch erfolgen, das sonst eine
192.168.0.0/24 gesetzt wird und man sich so elegant aussperrt.

Ohne weitere Änderungen wird das ganze noch nicht funktionieren, da der
Provider Router nicht per ARP die MAC Adressen der Clients im LAN Netz auflösen
kann.  Hier kommt nur `proxy_arp` ins Spiel. Dieses wird, wie im Skipt weiter
unten gezeigt, für das `WAN` und `LAN` Interface aktiviert. Fragt nun der
Provider Router nach der MAC Addresse de IP `192.168.0.170`, so antwortet der
OpenWrt Router an seiner Stelle mit seiner eigenen MAC Adresse. Somit bekommt
er das Pakete geschickt und kann es entsprechend weiterleiten. So können nun
die Unicast Pakete zwischen beiden Netzteilen geroutet werden.

# Anhang
## Script zum aktivieren von `proxy_arp` in OpenWrt

Diese Skript muss nach `/etc/hotplug.d/iface/`

    #!/bin/sh
  
    [ "$ACTION" = ifup -o "$ACTION" = ifupdate ] || exit 0
    [ "$DEVICE" != "" ] || exit 0
    
    case $DEVICE in
      eth0|br-lan)
        /sbin/sysctl net.ipv4.conf.${DEVICE}.proxy_arp=1 ;;
      *)
        exit 0 ;;
    esac

## Statische Routen als `uci show`

Die Reihenfolge ist wichtig da `ip` ohne R2 R3 nicht anlegen kann.

    # R2
    network.@route[0].interface='wan'
    network.@route[0].netmask='255.255.255.255'
    network.@route[0].target='192.168.0.1'
    network.@route[0]=route

    # R3
    network.@route[1].gateway='192.168.0.1'
    network.@route[1].interface='wan'
    network.@route[1].netmask='0.0.0.0'
    network.@route[1].target='0.0.0.0'
    network.@route[1]=route
