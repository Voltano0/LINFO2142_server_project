# Server Project - LINFO2142 : Computer Network

As explained in the guidelines file, this project aim to create a server infrastructure based on two Raspberry PI 3.
Since we are working with differents computers, it will be denoted if a command must be execute on the computer (C) or on a RPI (R)
The project is split into four parts and this readme will show you how we worked to achieve that goal.

## Part one : basic configuration

First, we need to install a fresh linux based OS. Since we are working with a Raspberry PI, the installation method is very classic. 
 > Note that usual OS like ubuntu can be too heavy for a RPI, we then used Raspberry Pi OS (https://www.raspberrypi.com/software/)

The next step is to create a VLAN based on a ID, our group number XX and a number Y. The ID is 254 + XX, which is *7* thus *ID = 261*. Y is 2 or 3 depending on wich RPI we are working on.
First, we need to acces the network interfaces : 
``` R
sudo nano /etc/network/interfaces
```
And then, create the VLAN with : 
```R
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

