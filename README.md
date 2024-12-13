# Home server journey (from scratch using Debian GNU/Linux)

## Goals
1. Local NAS
2. Home assistant
3. Pi-hole
4. Frigate NVR
5. Nextcloud
6. Immich

## OS choice
There are many options here. I decided to go for Debian stable as a base and just use docker to manage container deployments of most of the apps/services. My hardware is a bit different, but this is a pretty detailed guide I found online: https://tongkl.com/building-a-nas-part-1/.

## Hardware

### Processor and motherboard
I initially wanted a system with Intel N100 processor, but couldn't really find the ASUS N100 motherboard, except through shady third parties.
In the end I was able to snatch an Intel i5-14500 processor for half the price (at the time) and decided to get an [ASUS PRIME H610I-PLUS D4-CSM](https://www.asus.com/us/motherboards-components/motherboards/prime/prime-h610i-plus-d4-csm/) to go along with it.

### Memory
TODO

### OS drive
TODO

### Power supply
TODO

### Case
I chose the Jonsbo N3 for its compactness.

### Hard drives
This was a bit of an overkill. I probably should have gotten something smaller. I got 4 x Seagate EXOS 14Tb HDD.

## Power management
I have been unable to get the CPU to get past C3 states.
I have not dug deeper into forcing better power management by the OS and I have been relying on the BIOS settings (this might be the problem).
All the powertop tunings are in place using TLP.
