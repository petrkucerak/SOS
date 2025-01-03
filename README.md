# Semestrální práce na předmět SOS

## Zadání

Vytvořte Linux image pro Virtualbox který:

- bude mít tři síťová rozhraní (Virtualbox network adapter 1,2 a 3) - dále jen rozhraní, ... první, druhé a třetí v pořadí jak se zobrazují ve VB, toto je důležité dodržet pro následné testování/hodnocení odevzdaného Vašeho image cvičícím

- stroj získá na prvním rozhraní konfiguraci pomocí DHCP, při tom se bude prezentovat předepsanou MAC adresou (individuální adresu vygenerujte jako lokálně spravovanou (https://cs.wikipedia.org/wiki/MAC_adresa) s jinak samými nulami až na to že její poslední číslice budou shodné s Vaším dekadickým userID na serveru k332)
příklad:
moje ID je: 30123
pak adresa je 06:00:00:30:01:23 
Pozor: změnou se rozumí změna MAC adresy pomoci nástrojů OS Linux (ne změna v settings Virtualbox)

- stroj si pravidelně synchronizuje čas z Internetu

- na druhém rozhraní stroj poskytuje DHCP server pro síť 192.168.50.0/24, rozdává adresy v rozsahu 192.168.50.100-200, sám má adresu 192.168.50.1 a je pro tuto síť výchozí branou

- na třetím rozhraní má sám adresu 192.168.55.1 

- stroj dělá router do Internetu (NAT) pro obě ty výše uvedené vnitřní sítě  a směruje také mezi nimi navzájem ovšem s tím, že nová TCP spojení dovoluje pouze ze sítě 50 do sítě 55 nikoliv opačně 

- stroj pravidelně zjišťuje dostupnost vnitřní adresy 192.168.55.100  a pokud je dostupná s otevřeným portem 22, zařídí export tohoto(SSH) portu tak, aby byl dostupny z vnějšího (prvního) rozhraní z portu 8822

- Na portu 80 prvního rozhraní běží web server kde bude jednoduchou formou zobrazována pravidelně aktualizovaná statistika/počty firewallem zahozených paketů nevyhovujících zadání z bodu 6 pro jednotlivé unikátní IP adresy

- **volitelně**: bude port 22 z vnitřního počítače exportován i na veřejnou adresu serveru K332 (máte tam ssh účet) na portu s číslem Vašeho userID nebo prvního vyššího volného. Pozor SSH je z externích sítí mimo ČVUT nově (cca od konce 2023) zcela filtrováno (nejen porty, ale i obsah) je proto třeba testovat "ze školy", nebo s využitím školní VPN. Obejití tohoto (ČVUT SSH filtr) omezení není potřeba vyřešit v rámci generovaného image, jen s ním při případném testování počítat.

### Poznámky:
- zde v TEAMS odevzdáte pouze "reseni.txt" (jen txt,..... NE doc, pdf, ani nic jiného) soubor se stručným popisem co jste udělal a jak ke každému bodu,

- .txt bude obsahovat link na Váš image jako první řádek a také login informace: root/password

- Známka klas. zápočtu bude odvozena on správnosti a kvality provedení výše uvedených úkolů, povinné pro zápočet jsou první 4 body (tedy musí se připojovat sám k Internetu přes DHCP a fungovat alespoň DHCP server pro první vnitřní síť) a zároveň nesmí být Váš .ova image větší než 1G. 

- Celkově se snažte vytvořit co nejmenší image

- pro sdílení image je výhodné využít například službu CESNET FileSender  https://filesender.cesnet.cz/ (pozor na dobu expirace odkazu, je vhodné nastavit co největší)

## Řešení

Výsledný image https://filesender.cesnet.cz/?s=download&token=85e4d608-f632-434e-81e0-dd4d14fdd62c.

1. Stáhnul jsem si image [Apline Linuxu x86_64](https://alpinelinux.org/downloads/) a soustil ve VB s konfigurací popsanou v [dokumentaci](https://wiki.alpinelinux.org/wiki/Installing_Alpine_in_a_virtual_machine).
2. Provedl základní konfiguraci pomocí `setup-alpine`
   ```
   username: root
   heslo: 7777
   ```
3. Zjistil jsem si UserID pomocí commandu
   ```sh
   id -u kucerp28
   ```
   a MAC adresa síťového rozhraní bude vycházet z `1028`, bude tedy `"06:00:00:00:10:28"`.
4. Nastavil jsem rozhraní `eth0` v souboru `/etc/network/interfaces` `pre-up ip link set dev eth0 address 06:00:00:00:10:28` a restartoval network službu `rc-service networking restart`.
5. Pro synchronizaci času jsme zvolil službu `chrony`. Instaloval jsem ji již při konfiguraci Apline, pro jistotu jsme ověřil ale její funkčnost a znovu inicializoval pomocí:
   ```sh
   apk add chrony
   ntpd -qg # set NTP servery
   rc-service ntpd start
   date # validace
   ```
6. V sbouboru `/etc/network/interfaces` jsem donastavil rozhraní `eth1` a `eth2`:
   ```
   auto lo
   iface lo inet loopback

   auto eth0
   iface eth0 inet DHCP
      pre-up ip link set dev eth0 address 06:00:00:00:10:28

   auto eth1
   iface eth1 inet static
      address 192.168.50.1
      netmask 255.255.255.0

   auto eth2
   iface eth2 inet static
      address 192.168.55.1
      netmask 255.255.255.0
   ```
7. Nainstaloval jsem `dnsmasq` a nakonfiguroval DHCP pro druhé rozhraní.
   ```sh
   apk add dnsmasq # instalace dnsmasq
   ```
   Soubor `/etc/dnsmasq.conf`
   ```
   # Custom configuration for SOS semestral work

   interface=eth1
   dhcp-range=192.168.50.100,192.168.50.200,12h
   dhcp-option=option:router,192.168.50.1

   # End of custom SOS semestral DHCP configuration
   ```
8. Pro nastavení nat jsem povolil IP forwarding v soubory `/etc/sysctl.conf` pomocí
   ```
   net.ipv4.ip_forward=1
   ```
   a aplikoval změnu pomocí `sysctl -p`. Nainstaloval jsem `iptables`
   ```sh
   apk add iptables
   rc-update add iptables
   ```
   a následně jsem přidal *iptables pravidla*
   ```sh
   # internal network
   iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

   # routing with restriction
   iptables -A FORWARD -i eth1 -o eth2 -j ACCEPT
   iptables -A FORWARD -i eth2 -o eth1 -j REJECT
   ```
   a uložil konfiguraci.
   ```sh
   /etc/init.d/iptables save
   ```
9. Pro tvorbu serveru jsem zvolil `nginx`. Ten jsem spustil následujícím způsobem:
   ```sh
   apk add nginx # instalace nginx serveru
   mkdir -p /var/www/html # tvorba složky pro html soubor
   ```
   Poté jsem nasměroval port 80 na html soubor. Konfiguraci jsme provedl pomocí souboru `/etc/nginx/http.d/default.conf`.
   ```
   server {
    listen 80;
    server_name localhost;

    root /var/www/html;
    index firewall_stats.html;

    location / {
        try_files $uri $uri/ =404;
    }
   }
   ```
   Úspěšnost konfigurace jsem ověřil pomocí
   ```sh
   nginx -t
   rc-service nginx start # spustil server
   ```
   Html soubory vytvářím automaticky pomocí sh skirtu, který je spouště v periodě jako cron job. Lokace skriptu `/usr/local/bin/update_firewall_stats.sh`.
   ```sh
   #!/bin/sh

   # Generování statistik firewallu
   echo "<html><head><title>Firewall Stats</title></head><body><pre>" > /var/www/html/firewall_stats.html
   iptables -L -v -n | grep DROP >> /var/www/html/firewall_stats.html
   echo "</pre></body></html>" >> /var/www/html/firewall_stats.html
   ```
   Instalce cronu [Alpine package](https://pkgs.alpinelinux.org/package/edge/main/x86_64/busybox-openrc)
   ```sh
   apk add busybox-openrc

   # sputění cronu
   rc-service crond start
   rc-update add crond

   # konfigurace cronu, resp. přidání úlohy do cronu
   crontab -e
   */1 * * * * /usr/local/bin/update_firewall_stats.sh
   ```

   Fungování jsem ověřil pomocí commandů
   ```sh
   # naplnění daty
   iptables -A INPUT -s 192.168.1.1 -j DROP
   iptables -A INPUT -s 192.168.1.2 -j DROP
   iptables -A INPUT -s 192.168.1.1 -j DROP
   
   # stažení dat z web serveru
   wget http://127.0.0.1
   ```
10. Pro exponutí jsme využil `autossh`. Změnil jsem zdroj balíčků na komunitní `/etc/apk/repositories`
    ```sh
    apk update # zaktualizoval
    apk add autossh # nainstaloval
    autossh -M 0 -f -N -R 1028:192.168.55.1:22 kucerp28@k332.feld.cvut.cz # pro trvalé otevření ssh portu
    ```