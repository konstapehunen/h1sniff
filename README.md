## Wireshark ja Network Interface Names artikkelien tiivistys
-Wireshark on verkkoliikenteen kaapaustyökalu
-Verkkoliikennettä saadaan kaapattua valitsemalla verkkorajapinta ja käynnistämällä kaappaus
-Wiresharkin tilastotyökaluilla voidaan analysoida tietoja kuten liikenteen määrä ja protokollat
-Verkkoliitäntä ei ole aina fyysinen verkkokortti
-localhost verkkoliitäntä ei ole fyysinen verkkokortti
-systemd-pohjaiset järjestelmät nimeävät verkkoliitännät: en=Ethernet, wl=Wlan, lo=loopback
-Omat verkkoliitännät saa tulostettua komennoilla ip a ja ip route

### Virtuaalikoneen internetyhteyden katkaisu ja palautus
Aioin toteuttaa verkkoyhteyden katkaisun ja paluautuksen komentoriviltä, joten etsin tähän liittyvät komennot Googlettamalla ja löysin GeeksforGeekin artikkelin, jossa käsitellään verkon sammuttamista ja palauttamista komentokehotteelta.
Artikkelissa mainitaan, että yksi tapa sammuttaa verkkoyhteys on käyttää komentoa sudo ip link set [interface name] down (GeeksforGeeks s.a).
Katsoin verkkorajapintani nimen Wiresharkin yläpalkista, jossa keskellä lukee Capturing from eth0 ja ajoin komennon sudo ip link set eth0 down. Tämän johdosta virtuaalikoneeni verkkoyhteys lakkasi toimimasta.

TCP/IP-mallin neljä kerrosta Wiresharkilla analysoidusta paketista
Sieppasin Wiresharkilla komennon ping wiki.archlinux.org aikana tapahtuvaa liikennettä. Analysoitavaksi paketiksi valitsin DNS-kyselypaketin.

![image](https://github.com/user-attachments/assets/a027ed4b-8de0-419c-a277-0a2d3628253b)
Kuvassa laatikoituna TCP/IP-pinon neljä kerrosta

Ensimmäinen laatikoitu kerros, merkattu numerolla 1. on linkkikerros. 
Numerolla 2. merkattu laatikko on verkkokerros, jossa näkyy, että käytössä on IP-protokollan versio 4. 
Kolmantena on laatikoitu kuljetuskerros, jossa protokollana UDP sekä viimeisessä laatikoituna on sovelluskerros.

### Mitäs tuli surffattua?
surfing-secure.pcap tiedostossa on siepattu verkkoliikennettä. Paketteja on siepattu 283 kappaletta sekä kaappauksen kokonaiskesto on noin 7,5 sekuntia.
Sieppauksen koko on 122445 tavua, eli 122,4kB sekä protokollia löytyy DNS, QUIC, TCP ja TLSv1.3.
Avatessani Wiresharkin yläpalkista Statistics > Ethernet, löysin kaksi MAC-osoitetta ja Ipv4 lehdeltä löytyy 7 ip-osoitetta.

### Verkkokortin merkin tunnistus
Verkkokortin valmistajan saa tietoon MAC-osoitteen kuudesta ensimmäisestä merkistä. Näitä kuutta ensimmäistä kutsutaan nimellä OUI, joka tulee sanoista Organizational Unique Identifier. (GeeksforGeeks 2024).
Otin Wiresharkilla DNS-kysely -paketista Source MAC-osoitteen OUI:n ja syötin sen Wiresharkin verkkosivuilta löytyvään OUI hakutyökaluun, enkä saanut haulle tuloksia.
Googlaamalla kyseisen OUI:n päädyin GitHub repositorioon, jossa oli listattu erinäisille valmistajille kuuluvia OUI:ta. Linkki repositorioon.
Repositoriosta löytyi OUI 52:45:00, joka on Qemu Virtual NIC. Käyttäjä on siis käyttänyt virtuaalikonetta, koska verkkorajapinta on virtualisoitu.

Selainten tunnistus
Siepattu verkkoliikenne alkaa DNS-kyselyllä ja -vastauksella Google.Com:iin, jonka jälkeen joukko paketteja käyttävät QUIC-protokollaa.
QUIC on Googlen kehittämä kuljetuskerroksen protokolla, jota käytetään muun muassa asiakkaan ja palvelimen välisessä salatun yhteyden muodostamisessa. QUIC on oletus kuljetusprotokolla Chromium-pohjaisissa selaimissa. (Google s.a.)

Ensimmäisen QUIC-paketin Crypto -kehyksestä löytyy Extension: server_name (len=19) name=www.google.com, jonka perusteella päättelin, että kyseessä on selaimesta lähtöisin oleva paketti palvelimelle yhteyden muodostamisen aloittamista varten.
Edellämainittujen tietojen perusteella sanoisin, että käytössä on todennäköisesti Chromium-pohjainen selain.

Mielestäni käyttäjällä on käytössä myös toinen selain, sillä havaitsin paketteja, jotka käyttävät TLS-protokollaa ja niiden info-kentässä on merkattu verkkotunnus.
TLS-yhteyden muodostaminen alkaa TLS-kättelyllä, jossa asiakas lähettää Client hello-viestin palvelimelle. 

TLS:sä palvelin todentaa itsensä asiakkaalle sekä asiakas myös pyydettäessä todentaa identiteettinsä palvelimelle (Microsoft 2025).
Näiden tietojen perusteella oletan, että kyseessä on selain, joka aloittaa kommunikoinnin palvelimen kanssa.

Vielä täytyi kuitenkin selvittää mikä selain on käytössä, joten silmäilin paketin sisältöä tarkemmin.
Kaikki tieto tuntui olevan salattua, kuitenkin paketin Handshake Protocol-kehyksen lopusta löytyi Avain-arvo pareja, jotka pistivät silmääni. 

**JA3** on MD5 hashattu merkkijono, johon on sisälletty tietoja asiakkaasta. **Selaimilla** usein on **tunnistettavissa** tai tiedossa oleva **JA3 fingerprint** (Peakhour s.a).
Edellämainitsemani Avain-arvo parit ovat siis JA3 fingerprint:ejä.

Etsin työkalun, jolla voi hakea tunnettuja JA3 fingerprintejä ja haun tulokseksi sain tiedot Firefox-selaimen eri versioita. Tämän tiedon perusteella oletan käyttäjän käyttäneen myös Firefox-selainta.

![image](https://github.com/user-attachments/assets/478bdfa4-6a73-4e26-9434-f3cb19c006fc)
 Kuvakaappaus sivulta https://ja3.zone, jossa suoritettu haku edellämainitulle JA3 fingerprint:ille

### Oman verkkoliikenteen analysointi

Käynnistin Wiresharkissa pakettien sieppauksen ja ajoin komentorivillä komennon ping google.com, jonka jälkeen lopetin pakettien sieppauksen. Rajoitin kaappauksen kahdeksaan pakettiin demonstroidakseni ping-komennosta johtuvaa liikennettä.

![image](https://github.com/user-attachments/assets/39da58eb-86d5-4248-9076-2ec95de8ac01)
Kuvassa sieppaamani verkkoliikenne

Verkkoliikenne alkaa kahdella DNS-kyselyllä. Ensimmäinen paketti kysyy kohteen google.com IPv4 osoitetta sillä pakein DNS-tietue on tyyppiä A.
Tietuetyyppi on merkitty Wiresharkissa paketin info-kenttään.

![image](https://github.com/user-attachments/assets/faedb17e-4c33-4892-9bc1-d66daa0d4456)
Kuvassa korostettu DNS-tietuetyypi A ja AAAA

A-tyyppinen DNS-tietue sisältää IPv4-osoitteen ja sitä käytetään IP-osoitehakuihin (cloudflare).
Paketti numero 2 kysyy kohteen Ipv6-osoitetta, sillä sen DNS-tietue tyyppi on AAAA. Kuten myös tätäkin tietuetyyppiä käytetään IP-osoitehakuihin.
DNS-kyselyiden jälkeen verkkoliikenteessä näkyy vastaukset DNS-kyselyille. Verkkoliikenne jatkuu kahdella paketilla ICMP-liikennennettä, joka on peräisin ping komennosta. Näissä paketeissa lähetetään viesti palvelimelle ja palvelin vastaa takaisin. ICMP-protokollaa käytetään verkon tilan tarkastuksessa (MOOC s.a). 

### Lähteet
Cloudflare - What happens in a TLS handshake? | SSL handshake https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/
GeeksforGeeks – How To Disable A Network Interface https://www.geeksforgeeks.org/disabling-and-enabling-an-interface-on-linux-system/#how-to-disable-a-network-interface
GeeksforGeeks – What Is MAC Address 
https://www.geeksforgeeks.org/mac-address-in-computer-network/#format-of-mac-address
Github.com/joemiller - https://github.com/joemiller/mac-to-vendor/blob/main/db/mac-vendor.txt
JA3 - https://ja3.zone/ käytetty JA3 fingerprint haussa
Microsoft – TLS Handshake Protocol https://learn.microsoft.com/en-us/windows/win32/secauthn/tls-handshake-protocol
MOOC - Muita verkkokerroksen protokollia https://tietoliikenteen-perusteet-2-21.mooc.fi/osa-4/5-muita-verkkokerroksen-protokollia.md
Peakhour - What Is JA3 Fingerprinting https://www.peakhour.io/learning/fingerprinting/what-is-ja3-fingerprinting/
