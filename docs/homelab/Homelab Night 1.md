# Homelab Configuration Night 1

## Windows desktop setup for deployment

- install WSL (Windows Subsystem for Linux) to run Linux commands and scripts on Windows. This is useful for running scripts that are not available in PowerShell or for running Linux-based tools.

```powershell
wsl --install
```
- Install Ubuntu 24.04 LTS

```powershell
wsl --install Ubuntu-24.04
```
- Start Ubuntu 24.04 LTS

```powershell
wsl -d Ubuntu-24.04
```

- Get Determinate for WSL

```powershell
curl -fsSL https://install.determinate.systems/nix | sh -s -- install --determinate
```wsl

- cd to Git repo

```bash
cd /mnt/d/Git/homelab
```

- cd to linux home directory
    
```bash
cd ~
```

- make a new git directory
    
```bash
mkdir git
```
- cd to git directory
    
```bash
cd git
```
- clone the homelab repo
    
```bash
git clone https://github.com/RovxBot/homelab.git

```
- cd to the homelab repo
    
```bash
cd homelab
```
- start nix develop
            
```bash
nix develop
```
- run configuration

```bash
make configure
```

## Configuration

- Text editor: nano
- Domain: cooked.beer
- Seed repo: https://github.com/RovxBot/homelab
- Timezone: Australia/Adelaide
- IP range for load balancer: 10.0.0.0/24


- Server list:
```yaml
 all:
  vars:
    control_plane_endpoint: 192.168.1.180
    load_balancer_ip_pool:
      - 10.0.0.0/24
metal:
  children:
    masters:
      hosts:
        metal0: {ansible_host: 192.168.1.190, mac: '00:23:24:d1:f5:69', disk: sda, network_interface: eno1}
        metal1: {ansible_host: 192.168.1.191, mac: '00:23:24:d1:f3:f0', disk: sda, network_interface: eno1}
    workers:
      hosts:
```

- Terraform
    - Added teraform workspace as terraform ID: ws-yM6CtwT9NWtgTug3
    - Im unsure if this is what was required, but we will see I guess.

- git commit and push!
- Homelab token for github: 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

You need to use a token for login for CLI. Using your github username and password will not work.

## Conclusion

Well thats it for tonight. Im going to grab the mini PCs tomorrow night and wipe them.
I will need to set the Bios to wake on lan, collect the real mac addresses, and collect the network interfaces.

I should have the second lot of power supplies comming soon, so I will also need to add the other two mini PCs to the config.