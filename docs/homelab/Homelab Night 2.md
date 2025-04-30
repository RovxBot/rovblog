# Homelab Configuration Night 2

## The manual setup of each machine
Each unit is a Dell Optiplex 7050 secured from a local business that was upgrading their machines. Its going to be rough for home labbers if the Laptop thing keeps going and businesses just give everyone a laptop and dock.

I guess you could always rack a stack of old laptops, but I realy enjoy the micro form factor desktops. they are small, quiet and powerful enough for most tasks. and probably cool better than a closed stack of laptops.

### metal0
- **IP:** 192.168.1.190
- **Mac:** D8-9E-F3-90-E8-31
- **Secureboot:** Disabled
- **Wake on Lan:** Enabled for LAN
- **Network device name:** enp0
- **Storage device name:** nvme0n1

### metal1
- **IP:** 192.168.1.191
- **Mac:** D8-9E-F3-10-E8-A8
- **Secureboot:** Disabled
- **Wake on Lan:** Enabled for LAN
- **Network device name:** enp0
- **Storage device name:** nvme0n1

### metal2
- **IP:** 192.168.1.192
- **Mac:** D8-9E-F3-90-DD-2B
- **Secureboot:** Disabled
- **Wake on Lan:** Enabled for LAN
- **Network device name:** enp0
- **Storage device name:** nvme0n1

### metal3
- **IP:** 192.168.1.193
- **Mac:** D8-9E-F3-90-E9-54
- **Secureboot:** Disabled
- **Wake on Lan:** Enabled for LAN
- **Network device name:** enp0
- **Storage device name:** nvme0n1


So the super annoying part to get was the network and storage device names. I have no idea what Dell call these, and BIOS has nothing to help. I remembered that when installing Ubuntu it shows up. So I ended up booting one of the units to ubuntu and starting the install off so I can get the info I need.

## More on disks....

So turns out we have to go back into the BIOS on each unit and disable RAID as well swapping it to AHCI.


