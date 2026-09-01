# Transmission torrent kliens az Ubuntu NAS-on

Ez a leiras a NAS torrent klienset allitja be es kezeli SSH terminalbol. A kliens a Transmission daemon es a `transmission-remote` parancs.

## Cel es mukodes

- A torrent kliens szolgaltataskent fut, ujrainditas utan is elindul.
- A felkesz letoltesek nem jelennek meg a media konyvtarban: `/srv/nas/_torrents/incomplete` alatt vannak.
- A kesz torrentet minden hozzaadasnal egy valasztott, vagy ujonnan letrehozott `/srv/nas/Homeflix` alatti mappaba lehet tenni.
- Az RPC vezerles csak a NAS sajat gepen, `127.0.0.1:9091` cimen figyel. Ezert SSH-n belepve hasznalhato, de nem nyilik meg a helyi halozat vagy az internet fele.
- A kesz torrentek 2.0 feltoltesi aranyig seedelnek, aztan leallnak.
- Kezdetben nincs globalis savszelesseg-korlatozas.

Csak olyan torrentet adj hozza, amelynek letoltesere es megosztasara jogod van.

## 1. Telepitett komponensek ellenorzese

A jelenlegi gepen a `transmission-daemon` es a `transmission-cli` mar telepitve van. Ellenorzes:

```bash
dpkg -l transmission-daemon transmission-cli
```

A fo szolgaltatas neve:

```text
transmission-daemon.service
```

Allapot:

```bash
systemctl status transmission-daemon
systemctl is-enabled transmission-daemon
systemctl is-active transmission-daemon
```

Ha a csomagok meg nem lennenek telepitve egy masik Ubuntu gepen:

```bash
sudo apt update
sudo apt install transmission-daemon transmission-cli
```

## 2. Konyvtarak es jogosultsagok

A konyvtarstruktura:

```text
/srv/nas/
  Homeflix/
    films/
    series/
    nCORE_files/
    ...
  _torrents/
    incomplete/
```

A `/srv/nas/_torrents/incomplete` csak a felkesz letoltesek munkakonyvtara. A kesz torrenteket a Transmission a hozzaadaskor megadott `Homeflix` celmappaba helyezi.

Ha a munkakonyvtar meg nem letezne, hozd letre:

```bash
sudo mkdir -p /srv/nas/_torrents/incomplete
sudo chown -R debian-transmission:debian-transmission /srv/nas/_torrents
sudo chmod -R 770 /srv/nas/_torrents
```

A daemon `debian-transmission` felhasznalokent fut. Irasi joga kell legyen a `Homeflix` megfelelo celmappajaban is. A jelenlegi Homeflix fa `777` jogosultsagu, ezert mukodni fog, de ez minden helyi felhasznalonak teljes hozzaferest ad.

Ellenorzes:

```bash
sudo -u debian-transmission test -w /srv/nas/_torrents/incomplete
sudo -u debian-transmission test -w /srv/nas/Homeflix/films
```

Ha a parancsok nem adnak hibat, a daemon irni tud ezekbe a mappakba.

## 3. Transmission konfiguracio

Fontos: a szolgaltatast mindig allitsd le a `settings.json` szerkesztese elott. Kulonben leallaskor felulirhatja a kezzel modositott fajlt.

```bash
sudo systemctl stop transmission-daemon
sudo nano /etc/transmission-daemon/settings.json
```

A JSON-fajlban keresd meg es modositsd ezeket az ertekeket. Ha uj sort adsz hozza, figyelj a vesszokre: az utolso elem utan nincs vesszo.

```json
"download-dir": "/srv/nas/Homeflix",
"incomplete-dir": "/srv/nas/_torrents/incomplete",
"incomplete-dir-enabled": true,

"ratio-limit-enabled": true,
"ratio-limit": 2.0000,

"rpc-bind-address": "127.0.0.1",
"rpc-port": 9091,
"rpc-authentication-required": true,
"rpc-username": "torrentadmin",
"rpc-password": "VALASSZ_EGY_HOSSZU_EGYEDI_JELSZOT",
"rpc-whitelist-enabled": true,
"rpc-whitelist": "127.0.0.1,::1"
```

Ne irj bele valodi jelszot ebbe a dokumentumba, a shell history-ba, aliasba vagy megosztott jegyzetbe. A Transmission az elso sikeres inditas utan hash-elt formara csereli a jelszot a `settings.json`-ban; ez normalis, es nem visszafejtheto jelszoformatum.

Inditas es automatikus indulas beallitasa:

```bash
sudo systemctl enable --now transmission-daemon
sudo systemctl status transmission-daemon --no-pager
```

Ellenorizd, hogy az RPC csak localhoston figyel:

```bash
sudo ss -ltnp | grep ':9091'
```

Helyes esetben `127.0.0.1:9091` szerepel. Ha `0.0.0.0:9091` szerepel, allitsd le a szolgaltatast, ellenorizd a `rpc-bind-address` erteket, majd inditsd ujra.

Az RPC-hez ne adj UFW szabalyokat es ne nyiss routeres porttovabbitast. SSH-n belepve helyi kapcsolatkent mukodik.

## 4. Parancssoros belepes beallitasa

SSH-n belepve a kliens a NAS-on fut, igy a localhosthoz kapcsolodik. A parancs jelszot ker, ha nem adod meg a `-n` opcioban:

```bash
transmission-remote 127.0.0.1:9091 -n torrentadmin
```

Torrentlista tesztje:

```bash
transmission-remote 127.0.0.1:9091 -n torrentadmin -l
```

Ne tarold a jelszot `~/.bashrc` aliasban, mert sima szovegkent ott marad. Keszits inkabb egy rovid shell fuggvenyt, amely futasnal bekéri:

```bash
nano ~/.bashrc
```

A fajl vegere:

```bash
torrent() {
    transmission-remote 127.0.0.1:9091 -n torrentadmin "$@"
}
```

Ezutan toltsd ujra:

```bash
source ~/.bashrc
```

Hasznalat:

```bash
torrent -l
```

## 5. Interaktiv torrent-hozzaado script

Ez a script felsorolja a `Homeflix` kozvetlen almappait. Valaszthatsz egyet sorszammal, vagy letrehozhatsz egy uj, Homeflix alatti mappat. A celutvonal nem lehet abszolut es nem tartalmazhat `..` elemet.

Nyisd meg a scriptet:

```bash
sudo nano /usr/local/bin/torrent-add-homeflix
```

Illeszd be ezt:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

HOMEFLIX="/srv/nas/Homeflix"
HOST="127.0.0.1:9091"
USER="torrentadmin"

if [[ ! -d "$HOMEFLIX" ]]; then
    echo "Hiba: nem letezik: $HOMEFLIX" >&2
    exit 1
fi

mapfile -t folders < <(
    find "$HOMEFLIX" -mindepth 1 -maxdepth 1 -type d -printf '%f\n' | sort
)

echo "Cel mappa valasztasa:"
echo "0) Uj mappa letrehozasa"

for index in "${!folders[@]}"; do
    printf '%d) %s\n' "$((index + 1))" "${folders[$index]}"
done

read -r -p "Valasztas: " choice

if [[ "$choice" == "0" ]]; then
    read -r -p "Uj mappa neve vagy relativ utvonala: " relative_path

    if [[ -z "$relative_path" || "$relative_path" == /* || "$relative_path" == *".."* ]]; then
        echo "Hiba: csak Homeflix alatti relativ utvonal adhato meg." >&2
        exit 1
    fi

    destination="$HOMEFLIX/$relative_path"
    sudo mkdir -p "$destination"
    sudo chown nasadmin:nas "$destination"
    sudo chmod 777 "$destination"
elif [[ "$choice" =~ ^[1-9][0-9]*$ ]] && (( choice <= ${#folders[@]} )); then
    destination="$HOMEFLIX/${folders[$((choice - 1))]}"
else
    echo "Ervenytelen valasztas." >&2
    exit 1
fi

read -r -p "Magnet link vagy .torrent fajl utvonala: " torrent_source

if [[ -z "$torrent_source" ]]; then
    echo "Hiba: nincs megadott torrent." >&2
    exit 1
fi

transmission-remote "$HOST" -n "$USER" \
    -a "$torrent_source" \
    -w "$destination"

echo "Hozzaadva: $destination"
```

Futtatasi jog:

```bash
sudo chmod 755 /usr/local/bin/torrent-add-homeflix
```

Futtatas SSH-n belul:

```bash
torrent-add-homeflix
```

Pelda egy uj celra: valaszd a `0`-t, majd irj be `films/Film-cime`. A script ezt a konyvtarat a `/srv/nas/Homeflix/films/Film-cime` helyen hozza letre.

## 6. Torrentek kezelese SSH-n

Az azonosito (`ID`) a lista elso oszlopa. Minden muvelet elott listazd ki:

```bash
torrent -l
```

Reszletes informacio egy torrentre:

```bash
torrent -t ID --info
```

Inditas:

```bash
torrent -t ID --start
```

Leallitas:

```bash
torrent -t ID --stop
```

Torles a letoltott fajlok megtartasaval:

```bash
torrent -t ID --remove
```

Torles a letoltott fajlokkal egyutt:

```bash
torrent -t ID --remove-and-delete
```

Az utobbi veglegesen torli a celmappabol a torrenthez tartozo adatot. Csak akkor hasznald, ha erre biztosan szukseg van.

Minden torrent leallitasa:

```bash
torrent -t all --stop
```

Egy torrent jelenlegi letoltesi helyenek modositasa:

```bash
torrent -t ID --move /srv/nas/Homeflix/films
```

## 7. Sebessegkorlatok kesobbi beallitasa

Jelenleg nincs beallitott globalis korlat. Eloszor merd fel, hogy a letoltes mennyire zavarja az otthoni halozatot.

Globalis letoltesi es feltoltesi korlatok beallitasa KB/s egysegben:

```bash
torrent --downlimit 20000
torrent --uplimit 2000
```

A korlat kikapcsolasa:

```bash
torrent --no-downlimit
torrent --no-uplimit
```

Mivel a pontos Transmission CLI kapcsolok verziotol fugghetnek, elobb ellenorizd a telepitett valtozat sogojet:

```bash
transmission-remote --help | less
```

## 8. Teszteles es hibakereses

Eloszor jogszeruen megoszthato teszt torrenttel vagy Ubuntu ISO-val probald ki.

1. Lepj be SSH-n: `ssh afk@192.168.0.69`.
2. Ellenorizd a szolgaltatast: `systemctl is-active transmission-daemon`.
3. Futtasd a `torrent-add-homeflix` scriptet.
4. Valassz celmappat.
5. Add meg a magnet linket vagy egy `.torrent` fajl utvonalat.
6. Figyeld: `torrent -l` es `torrent -t ID --info`.
7. Ellenorizd, hogy a felkesz adat az `/srv/nas/_torrents/incomplete` alatt van.
8. Befejezes utan ellenorizd a kivalasztott Homeflix celmappat.

Hasznos naplok es ellenorzesek:

```bash
systemctl status transmission-daemon --no-pager
journalctl -u transmission-daemon -n 100 --no-pager
sudo ss -ltnp | grep ':9091'
sudo -u debian-transmission test -w /srv/nas/Homeflix/films
```

Ha a `transmission-remote` nem tud kapcsolodni, ellenorizd, hogy a szolgaltatas fut-e, es hogy `127.0.0.1:9091` hallgat-e. SSH-n a parancsot a NAS-on kell futtatni, nem a sajat kliensgepen.

Ha a torrent elindul, de irasi hibaval leall, ellenorizd a `debian-transmission` irasi jogat a munkakonyvtarban es a valasztott Homeflix celmappaban.