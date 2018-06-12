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

We laden eerst het ova-bestand in op onze pc. We loggen in met de login `vagrant/vagrant`. De home-directory bevat een testscript.

Nginx: nginx.service / firewall: http/https

The default server root directory (top level directory containing configuration files): /etc/nginx

The main Nginx configuration file: /etc/nginx/nginx.conf (kijk of hij luistert naar poort 80)

Server block (virtual hosts) configurations can be added in: /etc/nginx/conf.d

The default server document root directory (contains web files): /usr/share/nginx/html

We werken volgens het principe van het TCP/IP model om het op te lossen.

### Phase 1: Network Access

#### VM Opstarten

- Wat testen we nu?

We proberen om de VM op te starten via Virtualbox.

- Welke commando's heb ik hiervoor nodig?

Gewoon opstarten via Virtualbox

- Wat verwacht ik van uitkomst?

Dat hij direct komt tot het loginscherm van de VM

- Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?

Ik krijg van Virtualbox de foutmelding dat er iets foutloopt met vboxnet0 (adapter 2). Ik ga dus naar de netwerkinterface van deze VM en ik zie dat voor deze Host-only adapter de kabel niet inzit. Ik steek hem in en de VM start nu normaal op. we komen op het login scherm.

#### Is de datalink-laag correct?

- Wat testen we nu?

We gaan kijken of we op Datalink niveau de verwachte interfaces krijgen te zien.  

- Welke commando's heb ik hiervoor nodig?

`ip link`

- Wat verwacht ik van uitkomst?

Dat de loopback, de enp0s8 en de enp0s3 verschijnen en dat ze op UP staan.


- Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?

Dit lijkt in orde.

### Phase 2: Internet
* ip a (ip adres + netwerkmask)  -> /etc/sysconfig/networkscripts/ifcfg
daarna HERSTARTEN network.service
* ip r (DG) --> 10.0.2.2 en die van de Host-only
* cat /etc/resolv.conf (DNS) -> 10.0.2.3
* dig, nslookup, getent (query DNS)

#### Zijn de ip's correct ingesteld intern?

- Wat testen we nu?

We kijken of net zoals in de opgave de Host-only adapter het ip `192.168.56.42` gekregen, en de nat `10.0.2.15`

- Welke commando's heb ik hiervoor nodig?

`ip a`

- Wat verwacht ik van uitkomst?

Dat de interfaces correcte IP's hebben gekregen.

- Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?

Dit lijkt in orde.

#### Is de ip route correct?

- Wat testen we nu?

We kijken of onze Nat interface en onze Host-Only Adapter juiste Ip's hebben toegekregen op vlak van Default Gateway

- Welke commando's heb ik hiervoor nodig?

`ip r`

- Wat verwacht ik van uitkomst?

Dat `10.0.2.2` alles naar buiten zal sturen

- Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?

Het enige wat raar is is dat ook de `169.254.0.0` hier bij staat voor de Host-only adapter, wat niet correct is, deze moet uit de tabel verdwijnen. We gaan kijken in het bestand via `sudo cat /etc/sysconfig/network` maar hier is niets te zien van instellingen. Dit is gewoon het APIPA-adres en zorgt ervoor dat hosts zonder toegang tot het internet toch met elkaar kunnen communiceren.

#### Is de DNS correct?

- Wat testen we nu?

We kijken of hier de DNS voor de NAT correct is

- Welke commando's heb ik hiervoor nodig?

`sudo cat /etc/resolv.conf`

- Wat verwacht ik van uitkomst?

`10.0.2.3` moet verschijnen voor onze NAT-poort

- Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?

Dit lijkt in orde.

#### Kan er gepingd worden van de host naar de VM?

- Wat testen we nu?

Via CMD op mijn laptop probeer ik het ip-adres van de Host-Only te bereiken

- Welke commando's heb ik hiervoor nodig?

`ping 192.168.56.42` op mijn host

- Wat verwacht ik van uitkomst?

deze ping moet werken!

- Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?

Dit werkt!

### Phase 3: Transport

#### Welke services draaien momenteel en zijn deze juist?

- Wat testen we nu?

We testen of de nodige services voor deze LEMP-stack draaien.

- Welke commando's heb ik hiervoor nodig?

`sudo systemctl status nginx/network/firewalld`

- Wat verwacht ik van uitkomst?

Al deze services moeten draaien en starten bij het opstarten.

- Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?

De network.service is gestart bij het opstarten.
De firewalld.service is gestart bij het opstarten.
De nginx.service is NIET gestart bij het opstarten en staat niet aan.
--> hiervoor voeren we volgende commando's uit:
```
sudo systemctl enable nginx.service
sudo systemctl start nginx.service --> ERROR
sudo systemctl restart nginx.service
```

Na de enable en de start krijgen we in de status een error dat de nginx service niet is kunnen starten. We kijken eens wat de oplossing kan zijn via het commando `sudo systemctl --type=service`. Daar vinden we niet dat Apache nog aanstaat ofzo...

Via `journalctl -xe` vinden we dat het een error op komt door een Unregistered authentication Agent.

We kijken eens naar de poorten die open staan via het commando
`ss -tulpn`
--> DE POORTEN VOOR HTTP EN HTTPS staan niet open op dit moment, enkel voor ssh.


DIT IS EIGENLIJK AL OP HET NIVEAU VAN DE APPLICATIE-LAAG
--> We gaan naar de config file voor de service van nginx, die we kunnen vinden via `vi /etc/nginx/nginx.conf`

Daar vinden we terug dat de poort 8443 is ingesteld, dit moet 443 worden voor https.

Daarnaast vinden we ook dat het SSL certificate niet correct is ingesteld, er staat:

`ssl_certificate /etc/pki/tls/certs/nigxn.pem;`

dit moet worden:

`ssl_certificate /etc/pki/tls/certs/nginx.pem;`

We slaan alles op in dit bestand en proberen terug het commando `sudo systemctl start nginx.service` uit te voeren. Deze wordt nu opgestart inderdaad.

Nu laten we hem nog eens restarten via `sudo systemctl restart nginx.service`, de service blijft nog steeds draaien!

#### Welke poorten staan momenteel open?

- Wat testen we nu?

We testen welke poorten allemaal momenteel openstaan nu alle nodige services runnen.

- Welke commando's heb ik hiervoor nodig?

`sudo ss -tulpn`

- Wat verwacht ik van uitkomst?

De poorten 80,443 moeten zeker openstaan, ook kijken of poort voor ssh er bij staat

- Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?

De eerste keer dat ik dit commando uitvoerde hierboven, kreeg ik het niet te zien spijtig genoeg. Maar door de nginx service eindelijk werkende te krijgen, verschijnen nu zoals verwacht de poorten 80,443 en 22.


#### Wat is juist ingesteld op de firewall?

- Wat testen we nu?

We testen of de juiste interfaces zijn toegekend aan de firewall

- Welke commando's heb ik hiervoor nodig?

`sudo firewalld-cmd --list-all`

- Wat verwacht ik van uitkomst?

De 2 interfaces van onze VM moeten er instaal + http en https

- Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?

De interface `enp0s8` staat er niet bij, deze voegen we toe via het commando:

`sudo firewall-cmd --add-interface=enp0s8 --permanent`

Daarnaast wordt ook de `https` niet weergeven in de firewall. Deze voegen we toe via het commando:

`sudo firewall-cmd --add-service=https`
`sudo firewall-cmd --add-service=https --permanent`

Daarna starten we de service terug op (`sudo systemctl restart firewalld-service`) en dan zien we normaal dat alles is toegevoegd wat nodig is.


### Phase 4: Application

#### Welke error's verschijnen in journalctl?

- Wat testen we nu?

We houden de hele tijd een extra venster open om te testen waar de fouten liggen in onze configuratie

- Welke commando's heb ik hiervoor nodig?

`sudo journalctl -l -f`

- Wat verwacht ik van uitkomst?

Normaal mogen er geen errors meer overblijven

- Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?

De eerste keer dat ik fouten zag, was omdat de service van Nginx niet wou opstarten

--> oplossing: zie de Transport-laag en uitvoeren van `sudo nginx -t` om het configbestand te testen op syntax-fouten.

Nu krijg ik een 2de error wanneer ik probeer om de webpagina te openen op mijn host, die gaat over het feit dat de context van mijn php-files niet de juiste is, deze moet dezelfde worden als de map waarin deze staat.

--> oplossing: zie hieronder in mijn stukje over SELINUX



### SELinux

#### Testen of SELINUX aanstaat ?

- Wat testen we nu?

We testen of de beveiligde laag die over CENTOS ligt, werkelijk aanstaat.

- Welke commando's heb ik hiervoor nodig?

`sudo sestatus`

- Wat verwacht ik van uitkomst?

Dat SELINUX enabled is, en dat deze service op enforcing staat.

- Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?

Dit lijkt te kloppen, geen fouten.

#### Testen of juiste context is ingesteld voor de webfiles?

- Wat testen we nu?

We testen of de context van de php-files dezelfde is als de bovenliggende mappen waarin deze php-files horen te draaien.

- Welke commando's heb ik hiervoor nodig?

`ls -Z` / `sudo restorecon -R <directory>` /

- Wat verwacht ik van uitkomst?

Ik verwacht dat de PHP-files dezelfde context hebben als de mappen waarin ze staan.

- Wat krijg ik werkelijk te zien? Hoe zal ik het oplossen?

Dit klopt niet!

het php-bestand heeft als context `system_u:object_r:user_home_t:s0`, waar de map de context `system_u:object_r:usr_t:s0` heeft. Dit moet veranderen.

Dit doen we via het commando `sudo restorecon -R /usr/share/nginx/html`, daarna zou de context van de file in orde moeten zijn

Hierna herstarten we de services nginx nog eens.

Als we nu de browser openen, krijgen we de message te zien 'Keep calm and vagrant up'

...

## End result

- Kopie van de geslaagde testen:

```
[vagrant@nginx ~]$ ./runbats.sh
Running test /home/vagrant/01-whitebox.bats
 ✓ The SELinux status should be ‘enforcing’
 ✓ The firewall should be running
 ✓ Apache should not be installed
 ✓ Apache should not be running

4 tests, 0 failures
Running test /home/vagrant/02-blackbox.bats
 ✓ The web server host should be reachable
 ✓ The website should be accessible through HTTP
 ✓ The website should be accessible through HTTPS

3 tests, 0 failures
```

- Beschrijving van het resultaat als we kijken naar de webserver vanop de host:

Wanneer we surfen naar https://192.168.56.42, dan krijgen we simpelweg de boodschap te zien `keep calm and vagrant up`


- Beschrijving van de errors die ik nog steeds tegenkom:

Normaal gezien zie ik nu geen errors meer verschijnen in mijn `journalctl`



## Resources

* Mijn eigen boekje met linux-commando's

* https://github.com/bertvv/cheat-sheets/blob/master/docs/NetworkTroubleshooting.md

* https://github.com/bertvv/cheat-sheets/blob/master/docs/EL7.md

* https://bertvv.github.io/presentation-el7-basics/

* https://www.slideshare.net/bertvanvreckem/linux-troubleshooting-tips

* https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-centos-7

* https://www.digitalocean.com/community/tutorials/how-to-install-linux-nginx-mysql-php-lemp-stack-on-centos-7
