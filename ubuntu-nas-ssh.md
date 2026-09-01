# SSH eleres az Ubuntu NAS-hoz

Ez a dokumentum az Ubuntu NAS SSH elereset irja le. Az SSH lehetove teszi, hogy egy masik gep terminaljabol biztonsagosan kezeld a NAS-t, peldaul backupot indits, naplokat nezz vagy a torrent klienst vezereld.

## Jelenlegi allapot

- Az SSH szerver csomag: `openssh-server`
- Szolgaltatas: `ssh.service`
- Port: `22`
- Allapot: engedelyezett es aktiv
- Jelenlegi helyi IPv4 cim: `192.168.0.69`
- Belepesi felhasznalo: `afk`
- Jelszavas es kulcsos belepes is engedelyezett.
- Root felhasznalokent jelszavas belepes nem engedelyezett.

Az IP-cim DHCP-t hasznalva megvaltozhat. A router adminfeluleten kesobb erdemes DHCP-foglalast beallitani a NAS halozati kartyanak, hogy mindig ugyanazt a helyi IP-cimet kapja.

## 1. Allapot ellenorzese a NAS-on

```bash
systemctl status ssh
```

Rovid ellenorzesek:

```bash
systemctl is-enabled ssh
systemctl is-active ssh
sudo ss -ltnp | grep ':22'
```

Az elvart eredmeny, hogy a szolgaltatas `enabled` es `active`, a 22-es port pedig figyeljen.

## 2. Csatlakozas masik Linux vagy macOS gepbol

```bash
ssh afk@192.168.0.69
```

Elso csatlakozasnal az SSH rakerdez a szerver ujjlenyomatanak elfogadasara. Ellenorizd, hogy tenyleg a sajat NAS-od IP-cime szerepel a parancsban, majd ird be:

```text
yes
```

Ezutan az `afk` Ubuntu-felhasznalo jelszavat keri. Ez nem a Samba-jelszo es nem a Transmission jelszo.

Kilepes a tavoli kapcsolatbol:

```bash
exit
```

## 3. Csatlakozas Windowsrol

Windows Terminalban vagy PowerShellben:

```powershell
ssh afk@192.168.0.69
```

Windows 10/11 rendszereken az OpenSSH kliens altalaban elore telepitve van. Ha a `ssh` parancs nem talalhato, telepitsd a Windows Opcionális szolgaltatasok kozul az OpenSSH Client komponenst.

## 4. SSH kulcs beallitasa

Az SSH kulcs biztonsagosabb es kenyelmesebb, mint a jelszo. A kliensgepen hozz letre kulcsot:

```bash
ssh-keygen -t ed25519 -C "sajat-gep-nas"
```

Fogadd el az alapertelmezett fajlnevet. Jelszofrazist erosen ajanlott megadni.

A publikus kulcs felmasolasa Linuxrol vagy macOS-rol:

```bash
ssh-copy-id afk@192.168.0.69
```

Ezutan teszteld uj terminalablakban:

```bash
ssh afk@192.168.0.69
```

Windowsrol a publikus kulcsod tartalmat a NAS-on az alabbi fajlba kell helyezni, egy kulcs soronkent:

```text
/home/afk/.ssh/authorized_keys
```

A NAS-on a jogosultsagok legyenek:

```bash
chmod 700 /home/afk/.ssh
chmod 600 /home/afk/.ssh/authorized_keys
chown -R afk:afk /home/afk/.ssh
```

Csak akkor kapcsold ki a jelszavas belepest, ha a kulcsos belepes mar egy uj terminalbol biztosan mukodik.

## 5. Jelszavas belepes kikapcsolasa kulcs utan

Szerkeszd az SSH szerver beallitasat:

```bash
sudo nano /etc/ssh/sshd_config
```

Allitsd be vagy vedd fel ezt a sort:

```text
PasswordAuthentication no
```

Ellenorizd a konfiguraciot, majd toltsd ujra az SSH-t:

```bash
sudo sshd -t
sudo systemctl reload ssh
```

Tarts nyitva egy mar mukodo SSH kapcsolatot, ameddig egy masodik terminalbol nem igazoltad, hogy a kulcsos belepes mukodik. Igy egy eliras nem zar ki a NAS-bol.

## 6. Tuzfal

A jelenlegi UFW tuzfal inaktiv. Amikor bekapcsolod, az SSH-t elobb engedelyezd a sajat helyi halozatodrol, kulonben sajat magadat is kizathatod.

Pelda a `192.168.0.x` helyi halozathoz:

```bash
sudo ufw default deny incoming
sudo ufw allow from 192.168.0.0/24 to any port 22 proto tcp
sudo ufw enable
sudo ufw status numbered
```

Az `192.168.0.0/24` a jelenlegi halozatodhoz illo minta. Ha a router mas alhalozatot hasznal, elobb ellenorizd:

```bash
ip route
```

Az SSH portot ne iranyitsd tovabb a routerben az internet fele. Tavoli elereshez kesobb Tailscale vagy WireGuard VPN hasznalhato.

## 7. Hasznos SSH parancsok

Ezeket tavolrol, az SSH kapcsolatban futtathatod:

```bash
# NAS IP-cimek
hostname -I

# SSH naploja
sudo journalctl -u ssh -n 50 --no-pager

# Backup idozito es utolso mentési naplo
systemctl list-timers nas-backup.timer
journalctl -u nas-backup.service -n 50 --no-pager

# Torrent kliens allapota, ha telepitve van
systemctl status transmission-daemon
```

## 8. Hibakereses

Ha a kliensrol nem erheto el az SSH:

1. Ellenorizd a NAS-on: `systemctl is-active ssh`.
2. Ellenorizd a NAS aktualis IP-cimet: `hostname -I`.
3. Ellenorizd, hogy mindket gep ugyanazon a helyi halozaton van.
4. Ellenorizd a tuzfalat: `sudo ufw status verbose`.
5. A kliensrol reszletes hibakereseshez futtasd:

```bash
ssh -v afk@192.168.0.69
```

Ha a NAS IP-cime megvaltozott, a parancsban az uj `hostname -I` altal mutatott IPv4 cimet hasznald.