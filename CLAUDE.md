# Freebox Player Devialet — Hardware Hacking Project

## Context

This project is about rooting a second-hand Freebox Player Devialet to install
a custom firmware. The player uses a Qualcomm APQ8098 (Snapdragon 835) SoC
running a custom Android OS by Free.

## Current status

Player received and opened (2026-03-24). PCB photographed (photos/).
Phase 1 (reconnaissance non-destructive) not yet done — need to power on and test ADB/fastboot/nmap.
Phase 2 (UART) — demontage done, test points identified, need multimeter testing.
Next step: test TP5/TP6/TP7 with multimeter to identify UART pads.
If successful, document and publish the method.

## Important

- The player bought is a test unit ("pour pieces") — it's OK to brick it
- The user's main Freebox Player is NOT to be touched
- The user's alim (20V 6.5A) will be shared between both players
- Modifying owned hardware is legal in France/EU (no DRM circumvention, no piracy)
- User wants to publish the documentation if the hack succeeds
