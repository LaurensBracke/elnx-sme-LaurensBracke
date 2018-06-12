# Enterprise Linux Lab Report - Troubleshooting

- Student name: Laurens Bracke
- Class/group: TIN-TI-3B (Gent)

## Instructions

- Write a detailed report in the "Report" section below (in Dutch or English)
- Use correct Markdown! Use fenced code blocks for commands and their output, terminal transcripts, ...
- The different phases in the bottom-up troubleshooting process are described in their own subsections (heading of level 3, i.e. starting with `###`) with the name of the phase as title.
- Every step is described in detail:
    - describe what is being tested
    - give the command, including options and arguments, needed to execute the test, or the absolute path to the configuration file to be verified
    - give the expected output of the command or content of the configuration file (only the relevant parts are sufficient!)
    - if the actual output is different from the one expected, explain the cause and describe how you fixed this by giving the exact commands or necessary changes to configuration files
- In the section "End result", describe the final state of the service:
    - copy/paste a transcript of running the acceptance tests
    - describe the result of accessing the service from the host system
    - describe any error messages that still remain

## Report

We laden eerst het ova-bestand in op onze pc.

We werken volgens het principe van het TCP/IP model om het op te lossen.

### Phase 1: Network Access

#### VM Opstarten

*Wat testen we nu?*

We proberen om de VM op te starten via Virtualbox.

*Welke commando's heb ik hiervoor nodig?*

Gewoon opstarten via Virtualbox

*Wat verwacht ik van uitkomst?*

Dat hij direct komt tot het loginscherm van de VM

*Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?*

We krijgen de VM niet direct opgestart, aangezien de 2de netwerkadapter niet blijkt te werken. De VM wijst hier naar de naam 'vboxnet0', maar deze bestaat niet op mijn machine. We koppelen onze machine aan de naam 'Virtualbox Host-Only Ethernet Adapter' in de plaats, deze werkt met het ip-adres dat we nodig hebben voor enp0s8. We konden ook een Host-Only netwerk nog speciaal aanmaken, maar dat is niet overbodig vind ik als het netwerk 'Virtualbox Host-Only Ethernet Adapter' al voldoet.

#### Is de datalink-laag correct?

*Wat testen we nu?*

We gaan kijken of we op Datalink niveau de verwachte interfaces krijgen te zien.  

*Welke commando's heb ik hiervoor nodig?*

`ip link`

*Wat verwacht ik van uitkomst?*

Dat de loopback, de enp0s8 en de enp0s3 verschijnen en dat ze op UP staan.

*Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?*

De Host-only staat op down. Deze moeten we dus opnieuw zien op te starten. We gaan eens kijken naar de config hiervan. We voeren uit:

```
cd /etc/sysconfig/network-scripts/
cat ifcfg-enp0s8

```
Hier zien we dat voor deze kaar staat 'ONBOOT = no', deze wordt dus niet opgestart bij het opstarten, we gaan dit dus aanpassen met het VI commando.

```
sudo vi ifcfg-enp0s8
```

Daarin zetten we ONBOOT=yes.

Vervolgens laten we de machine herstarten via het commando `reboot`.

### Phase 2: Internet

#### Zijn de ip's correct ingesteld intern?

*Wat testen we nu?*

We kijken of net zoals in de opgave de Host-only adapter het ip `192.168.56.42` gekregen, en de nat `10.0.2.15`

*Welke commando's heb ik hiervoor nodig?*

`ip a`

*Wat verwacht ik van uitkomst?*

Dat de interfaces correcte IP's hebben gekregen.

*Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?*

Nu krijgen we voor loopback `127.0.0.1`, wat correct is. Voor enp0s3 krijgen we 10.0.2.15, wat ook correct is en voor enp0s8 krijgen we ook het correcte ip-adres.

#### Is de ip route correct?

*Wat testen we nu?*

We kijken of onze Nat interface en onze Host-Only Adapter juiste ip's hebben gekregen op vlak van Default Gateway

*Welke commando's heb ik hiervoor nodig?*

`ip r`

mogelijks `sudo cat /etc/sysconfig/network`

*Wat verwacht ik van uitkomst?*

Dat `10.0.2.2` alles naar buiten zal sturen

*Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?*

We krijgen te zien dat default alles wordt verstuurd via 10.0.2.2, wat ook zeer correct is.

#### Is de DNS correct?

*Wat testen we nu?*

We kijken of hier de DNS voor de NAT correct is

*Welke commando's heb ik hiervoor nodig?*

`sudo cat /etc/resolv.conf`

*Wat verwacht ik van uitkomst?*

`10.0.2.3` moet verschijnen voor onze NAT-poort

*Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?*

We krijgen ook 10.0.2.3 te zien als nameserver in deze file.

#### Kan er gepingd worden van de host naar de VM?

*Wat testen we nu?*

Via CMD op mijn laptop probeer ik het ip-adres van de Host-Only te bereiken

*Welke commando's heb ik hiervoor nodig?*

`ping 192.168.56.42` op mijn host

*Wat verwacht ik van uitkomst?*

deze ping moet werken!

*Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?*

we krijgen volgende uitvoer vanop de hostmachine naar enp0s8.

```
laure@LaurensBracke MINGW64 ~
$ ping 192.168.56.42

Pinging 192.168.56.42 with 32 bytes of data:
Reply from 192.168.56.42: bytes=32 time<1ms TTL=64
Reply from 192.168.56.42: bytes=32 time<1ms TTL=64
Reply from 192.168.56.42: bytes=32 time<1ms TTL=64
Reply from 192.168.56.42: bytes=32 time<1ms TTL=64

Ping statistics for 192.168.56.42:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms

```

SSH werkte nog niet vanop mijn host (ssh vagrant@192.168.56.42) door de vorige troubleshoot, heb dus eerst uit mijn `known_hosts` op mijn machine de vorige opgeslagen sleutel eruit gehaald. Daarna werkt wel het commando.

### Phase 3: Transport

#### Welke services draaien momenteel en zijn deze juist?

*Wat testen we nu?*

We testen of de nodige services voor deze DNS-server draaien.

*Welke commando's heb ik hiervoor nodig?*

`sudo systemctl status named/network/firewalld`

*Wat verwacht ik van uitkomst?*

Al deze services moeten draaien en starten bij het opstarten.

*Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?*

We voeren eerst volgend commando uit om te kijken op de service named draait.

`sudo systemctl status named`

We krijgen volgend uitvoer:

```
[vagrant@golbat ~]$ sudo systemctl -l status named
named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled)
   Active: failed (Result: exit-code) since Fri 2017-12-08 08:35:35 UTC; 10min ago
  Process: 1311 ExecStartPre=/usr/sbin/named-checkconf -z /etc/named.conf (code=exited, status=1/FAILURE)
```
enz...

Dit toont aan dat in /etc/named.conf of in de zonebestanden nog fouten staan qua syntax.

De oplossing hiervoor kan u terugvinden in het APPLICATIE-gedeelte.

Na het oplossen van de configuratiefiles (/etc/named.conf en /var/named/....):

Normaal moet nu alles kloppen en kunnen we de service (her)starten.
```

[vagrant@golbat ~]$ sudo systemctl restart named
[vagrant@golbat ~]$ sudo systemctl status named
named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled)
   Active: active (running) since Fri 2017-12-08 09:39:03 UTC; 32s ago
  Process: 3857 ExecStart=/usr/sbin/named -u named $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 3855 ExecStartPre=/usr/sbin/named-checkconf -z /etc/named.conf (code=exited, status=0/SUCCESS)
 Main PID: 3859 (named)
   CGroup: /system.slice/named.service
           └─3859 /usr/sbin/named -u named

```

#### Welke poorten staan momenteel open?

*Wat testen we nu?*

We testen welke poorten allemaal momenteel openstaan nu alle nodige services runnen.

*Welke commando's heb ik hiervoor nodig?*

`sudo ss -tulpn`

*Wat verwacht ik van uitkomst?*

De poorten 53 moet zeker openstaan, ook kijken of poort voor ssh er bij staat

*Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?*

Voor het opstarten van de service `named` zien we nog geen DNS verschijnen, dus ging ik eens kijken bij de firewall-instellingen hieronder.

#### Wat is juist ingesteld op de firewall?

*Wat testen we nu?*

We testen of de juiste interfaces zijn toegekend aan de firewall

*Welke commando's heb ik hiervoor nodig?*

`sudo firewalld-cmd --list-all`

*Wat verwacht ik van uitkomst?*

De 2 interfaces van onze VM moeten er instaan + dns (53/tcp)

*Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?*

DNS staat wel open met zijn poort, maar **ik persoonlijk** werk liever met de juiste service openzetten in de firewall. We gaan deze service dus permanent toevoegen.

```
[vagrant@golbat ~]$ sudo firewall-cmd --list-all
public (default, active)
  interfaces: enp0s3 enp0s8
  sources:
  services: dhcpv6-client ssh
  ports: 53/tcp
  masquerade: no
  forward-ports:
  icmp-blocks:
  rich rules:
```

We voeren volgende commando's uit.

```
sudo firewall-cmd --add-service=dns
sudo firewall-cmd --add-service=dns --permanent
sudo firewall-cmd --remove-port=53/tcp
sudo firewall-cmd --remove-port=53/tcp --permanent
sudo systemctl restart firewalld
```

Nu staat de service DNS wel erbij in de config van de firewall, en als we kijken naar de poorten via `sudo ss -tulpn`, dan staat nu de poort 53 er wel bij (service moet wel opgestart zijn), zelfs meerdere keren omdat dns moet antwoorden op verschillende netwerken.

### Phase 4: Application

#### Welke error's verschijnen in journalctl?

*Wat testen we nu?*

We houden de hele tijd een extra venster open om te testen waar de fouten liggen in onze configuratie

*Welke commando's heb ik hiervoor nodig?*

`sudo rndc querylog on`

`sudo journalctl -l -f (-u named)`

*Wat verwacht ik van uitkomst?*

Normaal mogen er geen errors meer overblijven

*Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?*

Wanneer we proberen om de service op te starten van BIND, krijgen we errors die zich precies vooral voordoen in de config van de zonebestanden, maar ook in het hoofdconfiguratiebestand.

* We kijken eens in het bestand door `sudo cat /etc/named.conf`. We zoeken nu naar de fout.

`listen-on port 53 { 127.0.0.1; };` moet `listen-on port 53 { any; };` worden.

*We kunnen ook nog `recursion yes;` zetten, maar het heeft geen effect op de uitslagen van het testscript.*

Ook is er een zone die fout gedefinieerd staat. Deze moet aangepast worden naar het volgende:

```
zone "56.168.192.in-addr.arpa" IN {
  type master;
  file "192.168.56.in-addr.arpa";
  notify yes;
  allow-update { none; };
};
zone "2.0.192.in-addr.arpa" IN {
  type master;
  file "2.0.192.in-addr.arpa";
  notify yes;
  allow-update { none; };
};

```

Dit passen we aan met het commando `sudo vi /etc/named.conf`

Daarna voeren we `sudo named-checkconf` uit op dit bestand ==> geen problemen.

* via `sudo vi /var/named/cynalco.com` passen we vervolgens het zonebestand aan.

We voegen een lijn erbij omdat er 2 Nameservers moeten zijn.

`IN    NS    tamatama.cynalco.com.`

Daarnaast zijn butterfree en beedle hun IP's omgewisseld. Dit moet het zijn:

```
butterfree           IN  A      192.168.56.12
db1                  IN  CNAME  butterfree
beedle               IN  A      192.168.56.13
db2                  IN  CNAME  beedle
```

Vervolgens moet er nog een naam bijkomen voor host mankey ==> files.

`files                IN  CNAME  mankey`

Hierna slaan we het bestand op met het commando `:wq`.

* Nu gaan we de file `/var/named/192.168.56.in-addr.arpa` aanpassen met VI. We houden de filenaam hetzelfde, hoewel deze eigenlijk fout is, maar we gaan de config erin aanpassen zodat deze klopt voor de aangepaste zone in `/etc/named.conf`.

We doen volgende aanpassingen (origin aanpassen en butterfree en beedle omwisselen):

`$ORIGIN 56.168.192.in-addr.arpa.`

```
12       IN  PTR  butterfree.cynalco.com.
13       IN  PTR  beedle.cynalco.com.
```

* Als laatste gaan we de file `/var/named/2.0.192.in-addr.arpa` aanpassen met VI. Hier is alles correct, enkel dat overal op het einde van de naam een punt is vergeten.

```
$TTL 1W
$ORIGIN 2.0.192.in-addr.arpa.

@ IN SOA golbat.cynalco.com. hostmaster.cynalco.com. (
  15081921
  1D
  1H
  1W
  1D )

                     IN  NS     golbat.cynalco.com.

2        IN  PTR  tamatama.cynalco.com.
4        IN  PTR  karakara.cynalco.com.
6        IN  PTR  sawamular.cynalco.com.
```

Nu kunnen we kort terug naar de Transport-laag op de service op te starten. Dit Werkt!

### SELinux (hoort deels bij de Applicatie-laag)

Er zijn geen SELINUX-problemen.

## End result

- Kopie van de geslaagde testen:

```
[vagrant@golbat ~]$ ./golbat_test.bats
 ✓ Forward lookups private servers
 ✓ Forward lookups public servers
 ✓ Reverse lookups private servers
 ✓ Reverse lookups public servers
 ✓ Alias lookups private servers
 ✓ Alias lookups public servers
 ✓ NS record lookup
 ✓ Mail server lookup

8 tests, 0 failures

```

- Aantonen dat DNS requests vanop het hostsysteem naar de VM sturen en antwoord krijgen mogelijk is:

```
laure@LaurensBracke MINGW64 ~
$ nslookup www.cynalco.com 192.168.56.42
Server:  golbat.cynalco.com
Address:  192.168.56.42

Name:    karakara.cynalco.com
Address:  192.0.2.4
Aliases:  www.cynalco.com

```

Ook in Screenshot-vorm: https://gyazo.com/1094642c6a453d7a1dae7f90d6f88776

## Resources

* Mijn eigen boekje met linux-commando's

* https://github.com/bertvv/cheat-sheets/blob/master/docs/NetworkTroubleshooting.md

* https://github.com/bertvv/cheat-sheets/blob/master/docs/EL7.md

* https://www.slideshare.net/bertvanvreckem/linux-troubleshooting-tips

* http://www.zytrax.com/books/dns/

* RHEL 7 Networking Guide, Hoofdstuk 11 DNS
