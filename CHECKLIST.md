# Freebox Player Devialet — Hardware Hacking Checklist

Voir [README.md](README.md) pour le contexte, le hardware et le materiel.

## Logiciels (tous installes)

Voir README.md section "Outils installes" pour les details.

---

## Phase 0 — Reparation prealable

### 0.1 — Ressouder le connecteur USB-C
- [ ] Ressouder le connecteur USB-C (dessoude a la reception)
- Methode : air chaud (~350-380°C) + flux + pate a braser (ou pads pre-etames au fer)
- Fixer les 4 pattes structurelles d'abord, puis chauffer les pins signal
- Verifier au multimetre : pas de court entre pins adjacents, continuite OK

## Phase 1 — Reconnaissance non-destructive (sans demonter)

### 1.1 — Branchement initial
- [ ] Brancher le player normalement (alimentation + HDMI)
- [ ] Le laisser booter completement
- [ ] Noter la version firmware dans Parametres > A propos

### 1.2 — Reseau
- [ ] Connecter le player en Ethernet (ou WiFi)
- [ ] Scanner les ports ouverts depuis le PC :
```bash
# Trouver l'IP du player (scanner le reseau)
nmap -sn 192.168.1.0/24

# Scanner tous les ports du player
nmap -sV -p- <IP_PLAYER>
```
- [ ] Noter tous les ports ouverts (ADB sur 5555 ? telnet ? SSH ?)

### 1.3 — ADB
- [ ] Tester si ADB est actif :
```bash
adb connect <IP_PLAYER>:5555
adb devices
```
- [ ] Si connecte : `adb shell` → noter si root (#) ou user ($)
- [ ] Si ADB refuse : activer le mode developpeur (Parametres > A propos > taper 7x sur "Build number")
- [ ] Re-tester ADB apres activation

### 1.4 — Fastboot
- [ ] Eteindre le player
- [ ] Essayer de booter en fastboot : maintenir Volume Down + Power (ou autre combo)
- [ ] Tester via USB-C :
```bash
fastboot devices
fastboot getvar all
fastboot oem device-info   # verifie si bootloader unlock
```

**RESULTAT Phase 1 :**
- ADB root → TERMINE, on a le controle total
- ADB user → Phase 3 (escalade)
- Fastboot accessible → Phase 4 (bootloader unlock)
- Rien → Phase 2 (UART)

---

## Phase 2 — UART (demontage)

### 2.1 — Demontage
- [ ] Debrancher completement le player
- [ ] Retirer les vis Torx du chassis
- [ ] Ouvrir delicatement (attention aux nappes haut-parleurs)
- [ ] Photographier la carte mere (recto/verso) en haute resolution

### 2.2 — Identification des pads UART
- [ ] Chercher un groupe de 3-4 pads/pins alignes marques "UART", "DBG", "DEBUG", "CON", "J-quelquechose"
- [ ] Souvent pres du SoC ou en bordure de PCB
- [ ] Identifier les pins avec le multimetre :

| Pin | Comment identifier |
|-----|-------------------|
| GND | Continuite avec le blindage/masse du chassis |
| VCC | 3.3V constant quand le player est allume |
| TX  | Tension qui oscille au boot (le player envoie des donnees) |
| RX  | Tension stable, flottante |

### 2.3 — Connexion UART
- [ ] NE PAS connecter VCC (alimenter l'UART depuis l'adaptateur peut endommager)
- [ ] Connecter : GND ↔ GND, TX player ↔ RX adaptateur, RX player ↔ TX adaptateur
- [ ] Ouvrir la console serie :
```bash
screen /dev/ttyUSB0 115200
# ou
minicom -D /dev/ttyUSB0 -b 115200
```
- [ ] Allumer le player et observer la sortie

### 2.4 — Analyse de la sortie UART
- [ ] Si pas de sortie : essayer d'autres bauds (9600, 38400, 57600, 921600)
- [ ] Sauvegarder TOUT le log de boot :
```bash
screen -L /dev/ttyUSB0 115200
# Le log sera dans ~/screenlog.0
```

**Chercher dans les logs :**
- [ ] `Hit any key to stop autoboot` → appuyer pour acceder au bootloader U-Boot/LK
- [ ] `login:` → essayer root/root, root/(vide), admin/admin, freebox/freebox
- [ ] `#` (shell root) → JACKPOT
- [ ] Lignes `[cmdline]` ou `bootargs` → parametres kernel (utile pour phase 3)
- [ ] `dm-verity` ou `verified boot` → indique la protection anti-modification
- [ ] `fastboot` dans les logs → comment entrer en mode fastboot

**RESULTAT Phase 2 :**
- Shell root via UART → TERMINE
- Shell user → Phase 3
- Bootloader interactif → Phase 4
- Logs seulement, pas de shell → Phase 5 (EDL)

---

## Phase 3 — Escalade de privileges (si shell user)

### 3.1 — Reconnaissance Android
```bash
# Version Android
getprop ro.build.version.release
getprop ro.build.version.sdk

# Build info
getprop ro.build.display.id
getprop ro.build.description

# Security patch level (important pour choisir les exploits)
getprop ro.build.version.security_patch

# SELinux status
getenforce

# Architecture
uname -a
cat /proc/cpuinfo
```

### 3.2 — Exploits kernel connus
Selon le security patch level, chercher les CVE applicables :
- [ ] CVE-2020-0041 (Binder)
- [ ] CVE-2019-2215 (Use-after-free Binder, "Bad Binder")
- [ ] CVE-2020-0069 (MediaTek — pas applicable ici mais verifier)
- [ ] Dirty Pipe (CVE-2022-0847) si kernel >= 5.8
- [ ] Dirty COW (CVE-2016-5195) si kernel ancien

Outils :
```bash
# Sur le PC, compiler un exploit pour arm64
# Exemple avec Dirty Pipe
git clone https://github.com/polygraphene/DirtyPipe-Android.git
# Cross-compiler pour aarch64
```

### 3.3 — Autres pistes
- [ ] Verifier si `su` existe quelque part : `find / -name su 2>/dev/null`
- [ ] Tester Magisk via ADB sideload si recovery accessible
- [ ] Chercher des binaires SUID : `find / -perm -4000 2>/dev/null`

---

## Phase 4 — Bootloader unlock

### 4.1 — Depuis fastboot
```bash
# Verifier l'etat
fastboot oem device-info

# Tenter unlock (probablement refuse sans code OEM)
fastboot flashing unlock
fastboot oem unlock
```

### 4.2 — Si refuse
- [ ] Chercher si Free a un outil de unlock (probablement non)
- [ ] Tenter l'exploit GBL si applicable (Snapdragon 835 peut etre vulnerable)
- [ ] Passer en Phase 5 (EDL)

---

## Phase 5 — Mode EDL (Emergency Download)

### 5.1 — Entrer en mode EDL
Methode 1 — Via ADB/fastboot (si accessible) :
```bash
adb reboot edl
# ou
fastboot oem edl
# ou
fastboot reboot emergency
```

Methode 2 — Hardware (test points) :
- [ ] Identifier les test points EDL sur le PCB (souvent 2 pads a shorter)
- [ ] Shorter les pads avec un fil/pince AVANT de brancher l'USB
- [ ] Brancher l'USB-C au PC en maintenant le short
- [ ] Verifier sur le PC :
```bash
lsusb | grep 05c6:9008
# Doit afficher "Qualcomm, Inc. Gobi Wireless Modem (QDL mode)"
```

Methode 3 — Interruption de boot :
- [ ] Corrompre volontairement une partition (derniere option)

### 5.2 — Communication EDL
```bash
# Avec l'outil edl
edl printgpt         # lire la table de partitions
edl rl dumps --skip  # dump complet de toutes les partitions
```

### 5.3 — Le probleme du firehose
Si edl demande un firehose programmer :
- [ ] Chercher dans les collections XDA : https://xdaforums.com/t/4525079/
- [ ] Chercher "prog_ufs_firehose_8998" ou "prog_emmc_firehose_8998"
- [ ] Tester les firehose d'autres appareils Snapdragon 835 (OnePlus 5, Essential PH-1, Pixel 2)
  - Peu de chances que ca marche (signature differente) mais tenter ne coute rien
```bash
edl --loader=prog_ufs_firehose_8998.mbn printgpt
```

---

## Phase 6 — Post-exploitation (si root obtenu)

### 6.1 — Dump complet
```bash
# Depuis ADB root
adb root
adb pull /dev/block/bootdevice/by-name/ /tmp/devialet-dump/

# Ou partition par partition
for part in boot recovery system vendor; do
    adb shell "dd if=/dev/block/bootdevice/by-name/$part" | \
        dd of=/tmp/devialet-dump/$part.img
done
```

### 6.2 — Desactiver dm-verity
```bash
adb disable-verity
adb reboot
```

### 6.3 — Installer un recovery custom (TWRP)
- [ ] Pas de TWRP officiel pour le Player Devialet
- [ ] Compiler TWRP depuis les sources avec le device tree extrait
- [ ] Flasher via fastboot :
```bash
fastboot flash recovery twrp-devialet.img
```

### 6.4 — Installer Magisk (root permanent)
```bash
# Patcher boot.img avec Magisk
# Puis flasher
fastboot flash boot magisk_patched_boot.img
```

### 6.5 — Objectif final : custom ROM / correction Alexa multiroom
Options :
- [ ] Patcher le skill Alexa directement dans /system
- [ ] Installer Android TV pour un meilleur ecosysteme d'apps
- [ ] Compiler LineageOS pour le device (gros projet)

---

## Ressources

- [bkerler/edl](https://github.com/bkerler/edl) — Qualcomm EDL tool
- [edl-ng](https://github.com/strongtz/edl-ng) — Modern EDL tool
- [XDA Firehose loaders](https://xdaforums.com/t/identifying-edl-firehose-loaders.4525079/)
- [Teardown video](https://www.universfreebox.com/article/61942/decouvrez-le-demontage-total-du-player-devialet-de-la-freebox-delta)
- [homebridge-freebox-player-delta](https://github.com/securechicken/homebridge-freebox-player-delta)
- [CVE Qualcomm](https://www.cvedetails.com/vulnerability-list/vendor_id-153/Qualcomm.html)

## Notes

- TOUJOURS travailler sur un player d'occasion, jamais sur le principal
- Photographier chaque etape du demontage pour pouvoir remonter
- Ne JAMAIS shorter des pins au hasard — identifier d'abord avec le multimetre
- Si brick logiciel : le mode EDL est dans la ROM silicium, toujours accessible
