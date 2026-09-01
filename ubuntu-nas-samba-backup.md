# Ubuntu NAS Samba es napi HDD-mentes

Ez a leiras a jelenlegi gep beallitasat dokumentalja:

- Ubuntu fut a 500 GB-os SSD-n.
- A NAS aktiv adatai az SSD-n, a `/srv/nas` alatt vannak.
- A `Homeflix` megosztas helye: `/srv/nas/Homeflix`.
- A csak mentendo adatok helye: `/srv/nas/saveForLater`.
- Az 1 TB-os HDD a `/srv/backup` alatt van csatolva.
- A mento script csak a `saveForLater` tartalmat menti napi snapshotokba.

## Fontos alapelvek

Az SSD az elsoleges tarhely. A HDD nem RAID-par, hanem biztonsagi mentesi cel. Ezert kulonbozo napi pillanatkepek keszulnek ra, hogy veletlen torles vagy hibas modositas utan korabbi fajlverzio is visszaallithato legyen.

A helyi HDD-mentes nem ved tuz, lopas, tulfeszultseg vagy mindket lemezt erinto hiba ellen. Potolhatatlan adatokrol legyen harmadik, fizikailag elkulonitett masolat is, peldaul kulso USB-lemezen vagy titkositott felhotarhelyen.

## 1. Lemezek azonositasa

Mielott barmilyen lemezt formaznal vagy csatolnal, mindig ellenorizd a meretet es a modellt:

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,TRAN,FSTYPE,MOUNTPOINTS
```

A jelenlegi mentesi lemez a rendszer szerint `/dev/sdb1`, es ez van `/srv/backup` ala csatolva. Ezt ellenorizheted:

```bash
findmnt /srv/backup
```

Soha ne feltetelezz eszkozbetut: uj USB-lemez csatlakoztatasakor a betujelek valtozhatnak.

## 2. HDD tartos csatolasa

A HDD-t ext4 fajlrendszerrel erdemes hasznalni. A stabil automatikus csatolashoz UUID kell, nem `/dev/sdb1`.

Az UUID lekerdezese:

```bash
sudo blkid /dev/sdb1
```

Az `/etc/fstab` fajlba felveendo sor, a sajat UUID-ddel:

```fstab
UUID=IDE-KERUL-A-HDD-UUID /srv/backup ext4 defaults,nofail,noatime,errors=remount-ro 0 2
```

Letre kell hozni a csatolasi pontot, ha meg nincs:

```bash
sudo mkdir -p /srv/backup
```

Az `fstab` modositas utan azonnal ellenorizd ujrainditas nelkul:

```bash
sudo mount -a
findmnt /srv/backup
```

Csak akkor inditsd ujra a gepet, ha a `findmnt` egyertelmuen a HDD-t mutatja a `/srv/backup` melle.

## 3. NAS mappak es jogosultsagok

A NAS gyokerkonyvtara nem kulon SSD-particio, hanem az Ubuntu rendszerparticiojan levo konyvtar. Ez most szandekosan egyszeru elrendezes:

```text
SSD (Ubuntu rendszerparticio)
  /srv/nas
    /Homeflix
    /saveForLater

HDD
  /srv/backup
    /snapshots
```

A `nas` csoport es a `nasadmin` helyi felhasznalo letrehozasa:

```bash
sudo addgroup nas
sudo adduser --disabled-password --gecos "" nasadmin
sudo adduser nasadmin nas
```

Ha a csoport vagy a tagsag mar letezik, az olyan uzenet, mint `already exists` vagy `usermod: no changes`, nem hiba.

Az aktiv NAS-mappa jogosultsagai:

```bash
sudo mkdir -p /srv/nas/saveForLater
sudo chown -R nasadmin:nas /srv/nas
sudo chmod 2775 /srv/nas
sudo chown nasadmin:nas /srv/nas/saveForLater
sudo chmod 2775 /srv/nas/saveForLater
```

A `2` a `2775` elejen setgid bit: az uj fajlok es mappak alapertelmezetten a `nas` csoporthoz fognak tartozni.

Ellenorzes:

```bash
id nasadmin
ls -ld /srv/nas /srv/nas/saveForLater
```

## 4. Homeflix athelyezese a NAS-ra

Ha a mappa meg a felhasznaloi Dokumentumokban van, az athelyezes:

```bash
sudo mv /home/afk/Documents/Homeflix /srv/nas/
sudo chown -R nasadmin:nas /srv/nas/Homeflix
sudo chmod -R g+rwX /srv/nas/Homeflix
```

Az `mv` athelyez, nem masol: a regi `/home/afk/Documents/Homeflix` utvonal megszunik. Elobb ellenorizd, hogy a celmappa nem letezik-e mar.

## 5. Samba telepitese es megosztasai

Telepites:

```bash
sudo apt update
sudo apt install samba
```

A Samba fo konfiguracioja az `/etc/samba/smb.conf`. A jelenlegi megosztasok:

```ini
[Homeflix]
    path = /srv/nas/Homeflix
    browsable = yes
    read only = no
    guest ok = yes
    guest only = yes
    force user = afk
    writable = yes
    create mask = 0664
    directory mask = 0775

[data]
    path = /srv/nas
    browseable = yes
    read only = no
    guest ok = yes
    force user = nasadmin
    force group = nas
    create mask = 0664
    directory mask = 0775
```

Konfiguracio ellenorzese es a Samba ujratoltese:

```bash
sudo testparm
sudo systemctl restart smbd
```

Windowsrol a megosztasok igy erhetoek el:

```text
\\NAS-IP\Homeflix
\\NAS-IP\data
```

Az `NAS-IP` helyett a gep helyi IP-cimet kell irni, amit igy nezhetsz meg:

```bash
hostname -I
```

### Samba felhasznalok es jelszavak

A Linux- es a Samba-felhasznalok kulon adatbazisban vannak. A Samba-felhasznalok listaja:

```bash
sudo pdbedit -L
```

A `nasadmin` Samba-felhasznalo letrehozasa vagy jelszavanak ujrabeallitasa:

```bash
sudo smbpasswd -a nasadmin
```

A Samba jelszavak nem olvashatok vissza, csak cserelhetok:

```bash
sudo smbpasswd nasadmin
```

A jelenlegi `guest ok = yes` beallitas mellett a megosztasokhoz nem kell jelszo. Ez csak megbizhato, zar t helyi halozaton vallalhato. Kesobb biztonsagosabb beallitas:

```ini
guest ok = no
valid users = nasadmin
```

Ekkor Samba ujrainditas szukseges:

```bash
sudo testparm && sudo systemctl restart smbd
```

## 6. Napi snapshot-mentes script

A script helye:

```text
/usr/local/sbin/nas-backup
```

Csak a `/srv/nas/saveForLater` tartalmat menti. A HDD-n a pillanatkepek igy neznek ki:

```text
/srv/backup/snapshots/2026-09-01
/srv/backup/snapshots/2026-09-02
/srv/backup/snapshots/latest
```

A `latest` az utolso sikeres mentest jelolo szimbolikus link. A hard linkes `--link-dest` miatt a valtozatlan fajlok nem foglalnak el ujra teljes helyet minden napi snapshotban.

A script teljes, helyes valtozata:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

SOURCE="/srv/nas/saveForLater"
BACKUP_ROOT="/srv/backup"
SNAPSHOTS="${BACKUP_ROOT}/snapshots"
DATE="$(date +%F)"
NEW_SNAPSHOT="${SNAPSHOTS}/${DATE}"
LATEST="${SNAPSHOTS}/latest"
RETENTION_DAYS=30

if [[ ! -d "$SOURCE" ]]; then
    echo "Hiba: a forraskonyvtar nem letezik: $SOURCE" >&2
    exit 1
fi

if ! mountpoint -q "$BACKUP_ROOT"; then
    echo "Hiba: a mentesi HDD nincs csatolva: $BACKUP_ROOT" >&2
    exit 1
fi

mkdir -p "$SNAPSHOTS"

if [[ -e "$NEW_SNAPSHOT" ]]; then
    echo "A mai mentes mar letezik: $NEW_SNAPSHOT"
    exit 0
fi

LINK_DEST=()
if [[ -L "$LATEST" ]] && [[ -d "$(readlink -f "$LATEST")" ]]; then
    LINK_DEST=(--link-dest="$(readlink -f "$LATEST")")
fi

rsync -aHAX --delete --numeric-ids \
    "${LINK_DEST[@]}" \
    "${SOURCE}/" "${NEW_SNAPSHOT}/"

ln -sfn "$DATE" "$LATEST"

find "$SNAPSHOTS" -mindepth 1 -maxdepth 1 -type d -mtime +"$RETENTION_DAYS" -exec rm -rf -- {} +

sync
echo "Sikeres mentes: $NEW_SNAPSHOT"
```

A script futtatasi joga:

```bash
sudo chmod 750 /usr/local/sbin/nas-backup
```

Kezdo teszt:

```bash
sudo /usr/local/sbin/nas-backup
ls -la /srv/backup/snapshots
```

Egy napon belul a script szandekosan nem hoz letre masodik snapshotot.

## 7. Automatikus napi futtatas systemd-vel

A service fajl: `/etc/systemd/system/nas-backup.service`

```ini
[Unit]
Description=NAS daily snapshot backup
RequiresMountsFor=/srv/backup
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/nas-backup
```

Az idozito fajl: `/etc/systemd/system/nas-backup.timer`

```ini
[Unit]
Description=Run NAS backup every day at 02:30

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
Unit=nas-backup.service

[Install]
WantedBy=timers.target
```

A `Persistent=true` azt jelenti, hogy ha a gep 02:30-kor ki volt kapcsolva, akkor a systemd a kovetkezo inditas utan potolja a kimaradt futast.

Aktivalas:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nas-backup.timer
```

Allapot es kovetkezo futas ellenorzese:

```bash
systemctl list-timers nas-backup.timer
systemctl status nas-backup.timer
```

Az automatikus futas tesztje:

```bash
sudo systemctl start nas-backup.service
journalctl -u nas-backup.service -n 50 --no-pager
```

## 8. Visszaallitas tesztelese

A mentes akkor er valamit, ha a visszaallitas is mukodik. Hozz letre egy tesztfajlt a `saveForLater` mappaban, mentsd, majd modositsd vagy torold, es keszits masnapi snapshotot.

Egy regi fajl visszaallitasa peldaul:

```bash
sudo cp -a /srv/backup/snapshots/2026-09-01/proba.txt /srv/nas/saveForLater/proba.txt
```

Elobb ellenorizd a snapshot nevét:

```bash
ls -la /srv/backup/snapshots
```

Havonta egyszer probalj ki tenyleges visszaallitast.

## 9. Halozati biztonsag

Az SMB portot soha ne nyisd ki kozvetlenul az internetre. A NAS csak a helyi halozaton legyen elerheto.

Az alhalozat ellenorzese:

```bash
ip route
```

Pelda UFW szabalyokra `192.168.1.0/24` helyi halozat eseten:

```bash
sudo ufw default deny incoming
sudo ufw allow from 192.168.1.0/24 to any port 445 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
sudo ufw enable
sudo ufw status numbered
```

Az `192.168.1.0/24` csak pelda. A sajat routered halozatat hasznald. Tavoli elereshez kesobb Tailscale vagy WireGuard VPN kell, nem nyitott SMB-port.

Automatikus biztonsagi frissitesek:

```bash
sudo apt install unattended-upgrades
```

## 10. Lemezallapot figyelese

Telepites es aktivitas:

```bash
sudo apt install smartmontools
sudo systemctl enable --now smartd
```

Allapot lekerdezese a sajat eszkozazonositokkal:

```bash
sudo smartctl -a /dev/sda
sudo smartctl -a /dev/sdb
```

Hetente rovid, havonta hosszu SMART-tesztet erdemes futtatni. Hibas SMART-ertek vagy I/O hiba eseten az erintett lemezt mielobb cserelni kell.

## 11. Gyors hibakereses

```bash
# Lemezek, fajlrendszerek, csatolasok
lsblk -f
findmnt /srv/backup

# Samba konfiguracio es szolgaltatas
sudo testparm
sudo systemctl status smbd

# Samba felhasznalok
sudo pdbedit -L

# Backup timer es utolso futas naploja
systemctl list-timers nas-backup.timer
journalctl -u nas-backup.service -n 50 --no-pager

# Backup scripted szintaktikai ellenorzese
sudo bash -n /usr/local/sbin/nas-backup
```

Ha a script azt irja, hogy a mentesi HDD nincs csatolva, ne keruld meg az ellenorzest. Elobb allitsd helyre a `/srv/backup` HDD-csatolast; ez akadalyozza meg, hogy a mentes veletlenul az SSD-re keruljon.