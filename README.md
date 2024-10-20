# Server Project - LINFO2142 : Computer Network

As explained in the guidelines file, this project aim to create a server infrastructure based on two Raspberry PI 3.
Since we are working with differents computers, it will be denoted if a command must be execute on the computer (C) or on a RPI (R).

The project is split into four parts and this readme will show you how we worked to achieve that goal.

## Part one : basic configuration

First, we need to install a fresh linux based OS. Since we are working with a Raspberry PI, the installation method is very classic. 
 > Note that usual OS like ubuntu can be too heavy for a RPI, we then used Raspberry Pi OS (https://www.raspberrypi.com/software/)

The next step is to create a VLAN based on a ID, our group number XX and a number Y. The ID is 254 + XX, which is *7* thus *ID = 261*. Y is 2 or 3 depending on wich RPI we are working on.

First, we need to acces the network interfaces :

>In R shell 
``` 
sudo nano /etc/network/interfaces
```
And then, create the VLAN with : 
```
# Automatically start the eth0.261 interface on boot
auto eth0.261

# Set eth0.261 to use a static IPv6 address
iface eth0.261 inet6 static

    # IPv6 address for this interface (group 7, device 2)
    address fc00:2142:ff07::2/64

    # Gateway address for this subnet
    gateway fc00:2142:ff07::1

    # Specify eth0 as the underlying physical device for VLAN 261
    vlan-raw-device eth0
```
The next step is to create an SSH link between C and R. We have few constrains :
1. The SSH key must be an ED25519 one and it must contain the username on R wich is **rasp**.
2. We must access the UCLouvain Network via the studssh.info.ucl.ac.be gateway
3. After the UCLouvain network we must acces our RPI via another gateway, 130.104.78.202

We open the ssh config file :

>In C shell
```
nano ~/.ssh/config
```
And copy : 
```
Host gateway
    HostName 130.104.78.202         # IP address of the SSH gateway
    User rasp                       # Username used to connect to the SSH gateway
    IdentityFile ~/.ssh/id_ed25519  # Path to your private SSH key
    ProxyJump UCL

Host rasp
    HostName fc00:2142:ff07::2  	# IP address of RPI
    User rasp                           # Username used to connect to RPI
    IdentityFile ~/.ssh/id_ed25519      # Path to your private SSH key
    ProxyJump gateway                   # Use the SSH gateway as a jump host

Host UCL
    HostName studssh.info.ucl.ac.be  # IP address of the UCLouvain gateway
    User gustint                     # Username used to connect to UCLouvain network
    IdentityFile ~/.ssh/id_rsa       # Path to your private SSH key
```
This code create 3 hosts, we can simply use **ssh rasp** to connect to our RPI since rasp jump to gateway that jump to the UCL gateway.
 
