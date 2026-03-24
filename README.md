# Freebox Player Devialet — Hardware Hacking

## Objectif

Rooter un Player Devialet (Freebox Delta) pour pouvoir installer un firmware custom
et corriger les limitations d'Alexa (notamment le smart home par piece qui ne fonctionne
pas sur la Freebox — bug Free ouvert depuis 2019, jamais resolu).

## Contexte

- La Freebox Delta ne supporte pas les groupes smart home Alexa correctement
- Bug [FS#25569](https://dev.freebox.fr/bugs/task/25569) ouvert depuis 2019, priorite "tres basse", aucune reponse de Free
- Le bootloader est verrouille, Free refuse de le debloquer ([FS#23024](https://dev.freebox.fr/bugs/task/23024))
- Le firmware est proprietaire, aucune contribution possible

## Hardware

| Spec | Detail |
|------|--------|
| SoC | Qualcomm APQ8098 (Snapdragon 835 / MSM8998) |
| RAM | 2 Go |
| Flash | 32 Go |
| OS | Android custom Free |
| Alimentation | 20V 6.5A (130W) barrel jack |

## Player d'occasion (cobaye)

- Achete sur LeBonCoin le 2026-03-18
- 35 EUR + ~5 EUR livraison
- Etat "pour pieces" (vendu sans alim, sans cable, sans telecommande)
- Utiliser l'alim de la Freebox principale pour tester
- QR code / serial : KM71180803792500439R02205256
- Recu et demonte le 2026-03-24
- **Connecteur USB-C dessoude a la reception** — a ressouder (air chaud + flux + pate a braser ou pre-etamage)

## Materiel necessaire

- [x] Player Devialet d'occasion (achete 2026-03-18, ~40 EUR livré)
- [x] Adaptateur USB-UART 3.3V CP2102 (commande 2026-03-18)
- [x] Pinces croco (pour se clipser sur les pads PCB sans souder)
- [x] Multimetre
- [x] Kit tournevis Torx (T6, T8, T10)

## Quoi faire a la reception

Suivre la checklist dans [CHECKLIST.md](CHECKLIST.md) — 6 phases progressives :

1. **Reconnaissance sans demonter** — brancher, scanner les ports (nmap), tester ADB/fastboot
2. **UART** — demonter, trouver les pads serie, connecter l'adaptateur USB-UART
3. **Escalade de privileges** — si shell user, exploits kernel selon le security patch level
4. **Bootloader unlock** — via fastboot
5. **Mode EDL** — acces bas niveau Qualcomm (ROM silicium, toujours accessible)
6. **Post-exploitation** — dump firmware, root permanent, custom ROM

## Outils installes

| Outil | Usage | Installation |
|-------|-------|-------------|
| `edl` | Qualcomm EDL (flash/dump bas niveau) | `~/edl/venv/`, symlink dans PATH |
| `minicom` | Console serie UART | apt |
| `screen` | Console serie UART (alternative) | apt |
| `adb` | Android Debug Bridge | apt |
| `fastboot` | Flash Android bootloader | apt |
| `nmap` | Scan ports reseau | apt |
| `binwalk` | Analyse firmware | apt |

Groupe `dialout` ajoute pour acces `/dev/ttyUSB0`.

## Composants identifies (photos 2026-03-24)

### Module SoM (carte rouge, ref E04)

| Ref | Composant | Role |
|-----|-----------|------|
| U1 | Qualcomm APQ8098 (Snapdragon 835) | SoC principal |
| U260 | SK hynix H28S6D3D2BMR (UFS 801A) | Flash UFS 32 Go |
| U1 (verso) | SK hynix H9HKNNNCRM MU | RAM LPDDR4 |
| U190 | Qualcomm PM8978 | PMIC (gestion alimentation) |

### Carte mere (PCB vert)

| Ref | Composant | Role |
|-----|-----------|------|
| — | Realtek RTL8353 | Controleur Ethernet |
| — | Pulse HC5008ANL / HD6008NL | Magnetics Ethernet |
| NH245 (x4) | TI SN74LV245 | Buffers I2S audio |
| U1000 | VIA T240 MP89 / MR804G | Codec audio |
| U2000 | A8066N01 | WiFi/BT ou peripherique (a confirmer) |
| — | Module blinde 25K-400-0784R | WiFi/BT (probablement QCA6174) |

### Test points — candidats UART

| Priorite | Test Points | Description |
|----------|------------|-------------|
| **#1** | **TP5, TP6, TP7** | 3 pads dores alignes — candidat UART principal (TX, RX, GND) |
| #2 | TP101, TP102 | 2 pads cote a cote — possiblement JTAG/debug |
| #3 | TP28, TP29 | 2 pads isoles |
| Autres | TP14, TP16, TP22, TP24, TP25, TP26, TP98, TP100, TP104, TP106, TP108 | Signaux individuels (test/mesure) |

**Prochaine etape :** tester TP5/TP6/TP7 au multimetre (GND = continuite masse, TX = oscille au boot, RX = flottant).

Autres ressources :
- [edl-ng](https://github.com/strongtz/edl-ng) — Modern EDL tool (alternative)
- [XDA Firehose loaders](https://xdaforums.com/t/identifying-edl-firehose-loaders.4525079/)

## Ressources

- [Teardown video](https://www.universfreebox.com/article/61942/decouvrez-le-demontage-total-du-player-devialet-de-la-freebox-delta)
- [homebridge-freebox-player-delta](https://github.com/securechicken/homebridge-freebox-player-delta)
- [CVE Qualcomm](https://www.cvedetails.com/vulnerability-list/vendor_id-153/Qualcomm.html)
- [Bug multiroom Alexa](https://dev.freebox.fr/bugs/task/25569)
- [Bug skill Alexa](https://dev.freebox.fr/bugs/task/33315)