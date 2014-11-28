---
layout: post
title: DNS mit nsd und unbound auf OpenBSD
---

Seit einigen Wochen läuft auf meinem Heim-Router wieder <a title="OpenBSD" href="http://www.openbsd.org" target="_blank">OpenBSD</a>. Vor ungefähr 10 Jahren war das schon mal der Fall, dann habe ich mich eine Zeit lang abgewendet und wegen der hübschen Graphen und den Webinterfaces <a title="IPCop" href="http://ipcop.org/" target="_blank">IPCop</a>, <a title="PFSense" href="http://pfsense.com/" target="_blank">Pfsense</a> und <a title="IPFire" href="http://www.ipfire.org/" target="_blank">IPfire</a> verwendet, nun also wieder OpenBSD. (Über die Gründe schreibe ich vielleicht ein anderes Mal, für nun genügt das OpenBSD einfach ne <a title="Gruende 1" href="http://bsdhosting.co.za/openbsd.html" target="_blank">Menge</a> <a title="OpenBSD Talk by Michael W. Lucas" href="http://www.youtube.com/watch?v=BXPV3vJF99k" target="_blank">Appeal</a> hat. )

![Image of an alix](/assets/alix.jpg)

Neben den „normalen“ Routing-Tätigkeiten, dem <a title="DHCP auf OpenBSD" href="http://openbsd.org/faq/faq6.html#DHCP" target="_blank">DHCP-Dienst</a>, dem <a title="Timeserver auf OpenBSD" href="http://openbsd.org/faq/faq6.html#OpenNTPD" target="_blank">Timeserver</a>, der <a title="pf" href="http://openbsd.org/faq/pf/index.html" target="_blank">PF-Firewall</a> usw. dient mein Router auch als DNS-Server. Bislang habe ich <a title="dnsmasq Homepage" href="http://www.thekelleys.org.uk/dnsmasq/doc.html" target="_blank">dnsmasq</a> als Forwarder eingesetzt. Um aber nicht von einem bestimmten externen DNS-Server abhängig zu sein (sogar <a title="OpenDNS" href="http://www.opendns.com/" target="_blank">OpenDNS</a> verwendet nun bei DNS-Fehlern so dämliche „Zwischenschaltseiten“…) sollte nun ein eigenständiger Resolver her, und da greift man heutzutage natürlich zu <a title="Unbound Homepage" href="http://www.unbound.net/index.html" target="_blank">unbound</a>. Da dessen Fähigkeiten, die lokale Domain als authoritativer Nameserver zu bespaßen arg begrenzt sind (z.B. kann unbound keine CNAMEs etc.) übernimmt diesen Part der <a title="NSD" href="http://www.nlnetlabs.nl/projects/nsd/" target="_blank">nsd</a>. Beide Server entstammen den „<a title="NLnet" href="http://www.nlnetlabs.nl/" target="_blank">NLnet Labs</a>“ und befinden sich im OpenBSD Basesystem. Interessanterweise wird unbound nicht standardmäßig gebaut und installiert. Wieso das so ist habe ich auf Anhieb nicht herausfinden können.
<pre>pkg_add -iv unbound</pre>
...löst das aber schnell.

Bevor wir richtig starten noch ein Satz zu „wieso denn nicht <a title="OpenBSD BIND manpage" href="http://www.openbsd.org/cgi-bin/man.cgi?query=named&amp;apropos=0&amp;sektion=0&amp;manpath=OpenBSD+Current&amp;arch=i386&amp;format=html">einfach</a> <a title="BIND" href="https://www.isc.org/downloads/bind/">BIND</a> nehmen?“. Berechtigte Frage. Einfach um was anderes auszuprobieren und vielleicht um ein paar Ressourcen zu sparen (mein Router ist ein PC Engines Alix, nicht das leistungsstärkste). Die Sicherheitsdiskussion war jedenfalls nicht ausschlaggebend: Für einen kleinen Heimrouter ist die Diskussion ob rekursiver Resolver und authoritativer Nameserver aus Sicherheitsgründen ein oder zwei Programme sein sollten rein akademisch.

Also, nachdem unbound installiert ist brauchen wir zunächst mal eine „root.hints“-Datei. Diese verrät unbound wo die DNS-Rootserver zu finden sind.
Diese holen wir so:
<pre>wget ftp://ftp.internic.net/domain/named.cache -O /var/unbound/etc/root.hints</pre>
Nun legen wir das Verzeichnis /var/unbound/etc/root.key an und ändern den Besitzer auf _unbound um: Als dieser Benutzer wird unbound künftig laufen und es muss in dieses Verzeichnis schreiben können. In das Verzeichnis kommt das root.keyfile mit folgendem Inhalt:
<pre>. IN DS 19036 8 2 49AAC11D7B6F6446702E54A1607371607A1A41855200FD2CE1CDDE32F24E8FB5</pre>
Das ist der Key für die DNS-Rootserver, mit dem kann unbound verifizieren ob die gelieferten Ergebnisse „echt“ sind. Den Key kann man sonst auch hier nachschlagen: <a href="https://data.iana.org/root-anchors/">https://data.iana.org/root-anchors/</a> .

Wenn wir das getan haben können wir uns nun der unbound.conf(5) zuwenden. Die manpage ist übrigens sehr zu empfehlen, außerdme ist die Datei gut kommentiert. Hier nur die Optionen die ich geändert habe:
<pre>/var/unbound/etc/unbound.conf</pre>
<pre>server:
 verbosity: 1
 interface: 127.0.0.1
 interface: 10.0.1.1
 port: 53
 do-ip6: no
 access-control: 127.0.0.0/8 allow
 access-control: 10.0.1.0/24 allow
 access-control: 10.0.3.0/24 allow
 root-hints: "/var/unbound/etc/root.hints"
 private-address: 10.0.0.0/8
 private-address: 172.16.0.0/12
 private-address: 192.168.0.0/16
 private-address: 169.254.0.0/16
 private-domain: "home.zone"
 domain-insecure: "home.zone"
 do-not-query-localhost: no
 local-zone: "1.0.10.in-addr.arpa." transparent
 auto-trust-anchor-file: "/var/unbound/etc/root.key/root.keyfile"
 val-clean-additional: yes

# remote control mit unbound-control-setup einrichten!
remote-control:
 control-enable: yes
 control-interface: 127.0.0.1

stub-zone:
 name: "home.zone"
 stub-addr: 127.0.0.1@5053
 stub-prime: no
 stub-first: no
stub-zone:
 name: "1.0.10.in-addr.arpa."
 stub-addr: 127.0.0.1@5053</pre>
Dann noch flux in der /etc/rc.conf.local die Zeile pkg_scripts=„unbound“ anfügen und schon sollte der eigene resolver funktionieren.

Die letzten 5 Zeilen der config-Datei sehen spannend aus: Hier wird unbound angewiesen bei Anfragen, die die Domain „home.zone“ betreffen doch bitte beim DNS-Server unter 127.0.0.1, Port 5053 nachzufragen. Das ist nämlich der nsd(8). Zur Konfiguration hier entlang:

Die /etc/nsd.conf ist denkbar einfach gehalten. Ein allgemeiner Teil unter server:, und dann die Zonen einrichten.
<pre>server:
 hide-version: no
 ip-address: 127.0.0.1
 port: 5053
 server-count: 1
 ip4-only: yes
 zonesdir: "/var/nsd/zones"
## master zone
zone:
 name: home.zone
 zonefile: home.zone.forward
zone:
 name: 1.0.10.in-addr.arpa
 zonefile: home.zone.reverse</pre>
Die Zonendateien sind BIND-kompatibel. Wer also NSD „nur mal ausprobieren“ möchte sollte seine Zonendateien einfach weiterverwenden können. Da ich keine mehr von BIND hatte musste ich meine neu anlegen:
<pre>;## NSD Zonefile - home.zone FORWARD
$ORIGIN home.zone.      ; default zone domain
$TTL 86400              ; default time to live

@ IN SOA gate.home.lan. chris.gate.home.zone. (
 2014012501  ; serial number
 28800       ; Refresh
 7200        ; Retry
 864000      ; Expire
 86400       ; Min TTL
 )

NS      gate.home.zone.
MX      10 gate.home.zone.

gate            IN      A       10.0.1.1
heaven          IN      A       10.0.1.2
file            CNAME   heaven
 …</pre>
<pre>;## NSD home.zone.reverse REVERSE LOOKUP FILE
$ORIGIN home.zone.      ; default zone domain
$TTL    86400           ; default time to live

1.0.10.in-addr.arpa.    IN      SOA gate.home.zone.     chris.gate.home.zone. (
 2014012501      ; serial
 28800           ; refresh
 7200            ; Retry
 864000          ; Expire
 86400           ; TTL
 )

1.1.0.10.in-addr.arpa.          IN      PTR     gate
2.1.0.10.in-addr.arpa.          IN      PTR     heaven
…</pre>
Nun kommt noch was nicht so ganz elegantes: Damit nsd die Zonefiles verwenden kann müssen diese in die nsd-eigene Datenbank übertragen werden. Das erledigt das Kommando nsdc rebuild für uns. Dann startet man das ganze und trägt noch die Zeile nsd_flags=„“ in die rc.conf.local ein damit der daemon beim nächsten reboot auch wieder hoch kommt.

(Update: Mit der Version 4 ist das nicht mehr nötig - Version 4.0.1 ist bei OpenBSD 5.5 mit dabei)

Das war es eigentlich. Der Text hier stellt natürlich einen extremem Schnelldurchlauf dar. Seht euch die man-Pages an, die sind recht hilfreich. Ansonsten: Viel Erfolg beim nachbauen! Natürlich funktionieren unbound und nsd auch prima unter Linux - das Prinzip sollte das gleiche sein, lediglich die Pfade dürften sich unterscheiden.

Und zu guter letzt: meldet euch, wenn etwas unklar geblieben ist oder wenn ich Fehler eingebaut habe :-).

Grüße!