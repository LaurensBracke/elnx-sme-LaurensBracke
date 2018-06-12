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

**Wat testen we nu?**

We proberen om de VM op te starten via Virtualbox.

**Welke commando's heb ik hiervoor nodig?**

Gewoon opstarten via Virtualbox

**Wat verwacht ik van uitkomst?**

Dat hij direct komt tot het loginscherm van de VM

**Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?**


#### Is de datalink-laag correct?

*Wat testen we nu?*

We gaan kijken of we op Datalink niveau de verwachte interfaces krijgen te zien.  

*Welke commando's heb ik hiervoor nodig?*

`ip link`

*Wat verwacht ik van uitkomst?*

Dat de loopback, de enp0s8 en de enp0s3 verschijnen en dat ze op UP staan.

*Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?*

### Phase 2: Internet

```
* ip a (ip adres + netwerkmask)  -> /etc/sysconfig/networkscripts/ifcfg
daarna HERSTARTEN network.service
* ip r (DG) --> 10.0.2.2 en die van de Host-only
* cat /etc/resolv.conf (DNS) -> 10.0.2.3
* dig, nslookup, getent (query DNS)
```

#### Zijn de ip's correct ingesteld intern?

**Wat testen we nu?**

We kijken of net zoals in de opgave de Host-only adapter het ip `192.168.56.42` gekregen, en de nat `10.0.2.15`

**Welke commando's heb ik hiervoor nodig?**

`ip a`

**Wat verwacht ik van uitkomst?**

Dat de interfaces correcte IP's hebben gekregen.

**Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?**

#### Is de ip route correct?

**Wat testen we nu?**

We kijken of onze Nat interface en onze Host-Only Adapter juiste ip's hebben gekregen op vlak van Default Gateway

**Welke commando's heb ik hiervoor nodig?**

`ip r`

mogelijks `sudo cat /etc/sysconfig/network`

**Wat verwacht ik van uitkomst?**

Dat `10.0.2.2` alles naar buiten zal sturen

**Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?**

#### Is de DNS correct?

**Wat testen we nu?**

We kijken of hier de DNS voor de NAT correct is

**Welke commando's heb ik hiervoor nodig?**

`sudo cat /etc/resolv.conf`

**Wat verwacht ik van uitkomst?**

`10.0.2.3` moet verschijnen voor onze NAT-poort

**Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?**

#### Kan er gepingd worden van de host naar de VM?

**Wat testen we nu?**

Via CMD op mijn laptop probeer ik het ip-adres van de Host-Only te bereiken

**Welke commando's heb ik hiervoor nodig?**

`ping 192.168.56.42` op mijn host

**Wat verwacht ik van uitkomst?**

deze ping moet werken!

**Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?**

### Phase 3: Transport

```
* sudo systemctl enable/start/restart
* `sudo systemctl --type=service` / `journalctl -xe`
* `ss -tulpn`
* `sudo firewall-cmd --add-interface=enp0s8 --permanent`
* `sudo firewall-cmd --add-service=https`
* `sudo firewall-cmd --add-service=https --permanent`
* `sudo systemctl restart firewalld-service`

```

#### Welke services draaien momenteel en zijn deze juist?

**Wat testen we nu?**

We testen of de nodige services voor deze DNS-server draaien.

**Welke commando's heb ik hiervoor nodig?**

`sudo systemctl status named/network/firewalld`

**Wat verwacht ik van uitkomst?**

Al deze services moeten draaien en starten bij het opstarten.

**Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?**


#### Welke poorten staan momenteel open?

**Wat testen we nu?**

We testen welke poorten allemaal momenteel openstaan nu alle nodige services runnen.

**Welke commando's heb ik hiervoor nodig?**

`sudo ss -tulpn`

**Wat verwacht ik van uitkomst?**

De poorten 53 (of mogelijks 953) moeten zeker openstaan, ook kijken of poort voor ssh er bij staat

**Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?**


#### Wat is juist ingesteld op de firewall?

**Wat testen we nu?**

We testen of de juiste interfaces zijn toegekend aan de firewall

**Welke commando's heb ik hiervoor nodig?**

`sudo firewalld-cmd --list-all`

**Wat verwacht ik van uitkomst?**

De 2 interfaces van onze VM moeten er instaal + dns

**Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?**


### Phase 4: Application

```
* dig, nslookup, getent (query DNS)
```

#### Welke error's verschijnen in journalctl?

**Wat testen we nu?**

We houden de hele tijd een extra venster open om te testen waar de fouten liggen in onze configuratie

**Welke commando's heb ik hiervoor nodig?**

`sudo journalctl -l -f`

**Wat verwacht ik van uitkomst?**

Normaal mogen er geen errors meer overblijven

**Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?**


### SELinux (hoort deels bij de Applicatie-laag)

```
* `ls -Z` / `sudo restorecon -R <directory>` --> service herstarten hierna van dns
```

#### Testen of SELINUX aanstaat ?

**Wat testen we nu?**

We testen of de beveiligde laag die over CENTOS ligt, werkelijk aanstaat.

**Welke commando's heb ik hiervoor nodig?**

`sudo sestatus`

**Wat verwacht ik van uitkomst?**

Dat SELINUX enabled is, en dat deze service op enforcing staat.

**Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?**

#### Testen of juiste context is ingesteld?

**Wat testen we nu?**

We testen of de context van de files dezelfde is als de bovenliggende mappen waarin deze files horen te draaien.

**Welke commando's heb ik hiervoor nodig?**

`ls -Z` / `sudo restorecon -R <directory>`

**Wat verwacht ik van uitkomst?**

Ik verwacht dat de files dezelfde (en de correcte) context hebben als de mappen waarin ze staan.

**Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?**



## End result

- Kopie van de geslaagde testen:


- Aantonen dat DNS requests vanop het hostsysteem naar de VM sturen en antwoord krijgen mogelijk is:

Screenshot: https://gyazo.com/c983e46c8c58572f0a9a68f9df5bf4a7


- Beschrijving van de errors die ik nog steeds tegenkom:

Normaal gezien zie ik nu geen errors meer verschijnen in mijn `journalctl`



## Resources

* Mijn eigen boekje met linux-commando's

* https://github.com/bertvv/cheat-sheets/blob/master/docs/NetworkTroubleshooting.md

* https://github.com/bertvv/cheat-sheets/blob/master/docs/EL7.md

* https://www.slideshare.net/bertvanvreckem/linux-troubleshooting-tips

* http://www.zytrax.com/books/dns/

* RHEL 7 Networking Guide, Hoofdstuk 11 DNS
