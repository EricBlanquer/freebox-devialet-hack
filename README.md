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

Autres ressources :
- [edl-ng](https://github.com/strongtz/edl-ng) — Modern EDL tool (alternative)
- [XDA Firehose loaders](https://xdaforums.com/t/identifying-edl-firehose-loaders.4525079/)

## Ressources

- [Teardown video](https://www.universfreebox.com/article/61942/decouvrez-le-demontage-total-du-player-devialet-de-la-freebox-delta)
- [homebridge-freebox-player-delta](https://github.com/securechicken/homebridge-freebox-player-delta)
- [CVE Qualcomm](https://www.cvedetails.com/vulnerability-list/vendor_id-153/Qualcomm.html)
- [Bug multiroom Alexa](https://dev.freebox.fr/bugs/task/25569)
- [Bug skill Alexa](https://dev.freebox.fr/bugs/task/33315)