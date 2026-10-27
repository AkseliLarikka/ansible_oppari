# ansible_oppari

Repositorio Akselin opinnäytetyön Ansible skriptien versionhallintaa varten.

# Automaation kohteiden valinta ja rajaukset

Lynis-auditoinnin ja CIS-CAT Lite -skannauksen havaintojen perusteella opinnäytetyön Ansible-automaation piiriin valikoitui joukko ensisijaisia kovennuksia. Valintaperusteina painotettiin toimenpiteiden standardinmukaisuutta, soveltuvuutta koodiksi kääntämiseen (Infrastructure as Code) sekä niiden tuottamaa merkittävää parannusta järjestelmän tietoturvaan.

Ohjelmallisten auditointien lisäksi työssä kartoitettiin koti- ja laboratorioympäristöille suunnattuja yleisiä tietoturvasuosituksia. Nämä suositukset korostavat tyypillisesti hyökkäyspinta-alan minimointia, verkon segmentointia sekä pääsynhallinnan tiukentamista (Cybersecurity and Infrastructure Security Agency, ei pvm.). Kartoituksella ei kuitenkaan löydetty uutuusarvoa omaavia suosituksia verrattuna automaattisten auditointien tarjoamiin tuloksiin.

## Toteutettavat tietoturvakovennukset

Automaatiolla toteutettaviksi kohteiksi valittiin seuraavat ydinosa-alueet, jotka voidaan viedä tuotantoon luotettavasti Ansiblen sisäänrakennettuja moduuleja hyödyntäen:

*   **SSH-palvelimen konfiguraatio (SSH-7408):** Etähallinnan turvallisuuden perustana toteutetaan pääkäyttäjän (root) kirjautumisen estäminen, oletusportin vaihtaminen, X11-forwardingin poistaminen käytöstä sekä idle-timeout-arvojen määrittäminen.
*   **Ytimen parametrien optimointi (KRNL-6000):** Verkkotason hyökkäyksiltä suojaudutaan estämällä IP-reititys (forwarding), kytkemällä rp_filter päälle ja rajoittamalla core dump -tiedostojen luomista. Nämä toimenpiteet toteutetaan Ansiblen sysctl-moduulilla.
*   **Tarpeettomien verkkoprotokollien esto (NETW-3200):** DCCP-, SCTP-, RDS- ja TIPC-protokollien latautuminen estetään luomalla erillinen asetustiedosto (`/etc/modprobe.d/cis.conf`) ytimen moduulitasolla.
*   **PAM-moduulit ja käyttöoikeudet (AUTH-9230, AUTH-9328):** CIS-CAT-skannauksen pohjalta määritetään tiedostojen oletusoikeuksien tiukentaminen (umask 027), väliaikaishakemistojen käyttöoikeuksien rajoittaminen ja sudo-oikeuksien konfigurointi.
*   **Palomuuri ja aktiivinen torjunta (DEB-0880):** UFW-palomuurin oletussääntöjen määrittämisen lisäksi asennetaan Fail2ban-palvelu suojaamaan SSH-porttia väsytyshyökkäyksiltä.
*   **Haittaohjelmaskannaus (HRDN-7230) ja päivitykset (PKGS-7392):** Järjestelmään asennetaan haittaohjelmaskanneri (esim. rkhunter tai chkrootkit) automaattisia tarkistuksia varten, ja määritetään haavoittuvien pakettien päivitykset apt-moduulilla.
*   **Kirjautumisbannerit (BANN-7126, 7130):** Varoitustekstien (`/etc/issue` ja `/etc/issue.net`) asettaminen kirjautumisen yhteyteen luo visuaalisesti todennettavan muutoksen opinnäytetyön tuloksiin.

## Työn ulkopuolelle rajatut toimenpiteet

Osa auditointityökalujen ehdottamista toimenpiteistä jätettiin automaation ja opinnäytetyön laajuuden ulkopuolelle. Rajaukset tehtiin toimenpiteiden aiheuttamien riskien, käytettävyyden heikkenemisen tai kohdeympäristön ominaisuuksien vuoksi:

*   **Salasanojen vanhenemiskäytännöt:** Auditointityökalut suosittelivat salasanojen maksimi- ja minimi-iän määrittämistä, mutta tästä luovuttiin. Yhdysvaltain standardisoimisviraston (NIST) SP 800-63B -ohjeistuksen mukaan pakotetut salasananvaihdot heikentävät salasanojen laatua, sillä käyttäjät luovat usein helposti arvattavia muunnelmia. Salasana tulisi vaihtaa ainoastaan silloin, kun sen epäillään vaarantuneen (Temoshok ym., 2025).
*   **Levyosioiden eriyttäminen (FILE-6310):** Vaatimus `/home`- ja `/var`-hakemistojen siirtämisestä omille osioilleen rajattiin pois. Tämä toimenpide vaatisi käyttöjärjestelmän uudelleenasennuksen tai riskialtista fyysistä uudelleenpartitiointia lennosta, mikä johtaa herkästi datan tuhoutumiseen.
*   **GRUB-salasana (BOOT-5122):** Ominaisuus on kriittinen riski ilman näyttöä (headless) toimivassa palvelimessa. Virtakatkoksen jälkeen järjestelmä vaatisi manuaalista salasanan syöttöä fyysiseltä näppäimistöltä käynnistyäkseen.
*   **Palvelutilien lukitukset (AUTH-9284):** Lynisin antamat huomautukset lukituista tileistä (kuten Minecraft-palvelimen `crafty`-käyttäjä) sivuutettiin. Palvelutilien kuuluukin olla lukittuja interaktiivisen kirjautumisen estämiseksi.
*   **Ulkoinen lokitus (LOGG-2154):** Erillisen Syslog-palvelimen (esim. rsyslog tai Splunk) pystyttäminen toiselle koneelle keskitettyä lokienhallintaa varten rajattiin laajuuden ulkopuolelle.
*   **Pakettien interaktiiviset tarkistukset (DEB-0810, 0811):** apt-listbugs- tai apt-listchanges-pakettien asennus estäisi täysin automaattiset taustapäivitykset (unattended-upgrades), sillä ne pysäyttävät suorituksen ja vaativat käyttäjältä manuaalista hyväksyntää, jos bugeja löytyy.
*   **Harkinnanvaraiset toimenpiteet (PRNT-2307, USB-1000):** Tulostinpalvelimen (CUPS) asetusten koventamisen sijaan koko tarpeeton paketti poistetaan Ansiblella. USB-porttien massamuistien estämistä ei otettu käyttöön, jotta ulkoisten varmuuskopiolevyjen kytkeminen palvelimeen säilyy jatkossakin mahdollisena.