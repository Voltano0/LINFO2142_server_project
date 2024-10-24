  GNU nano 7.2                                     README.md *                                            
# Server Project - LINFO2142 : Computer Network
## Context
As explained in the guidelines file, this project aim to create a server infrastructure based on two Raspberry PI. 
Since we are working with differents computers and 2 RPI, it will be denoted if a command must be executed on the RPI or on the computer.
1. username : rasp
2. Y number = 2

The project is split into four parts and this readme will show you how we worked to achieve that goal.

## 1. Basic configuration

First, we need to install a fresh linux based OS. Since we are working with a Raspberry PI, the installation is quite simple. We used the Raspberry Pi Imager to flash the OS on a SD card.
 > Note that usual OS like ubuntu can be too heavy for a RPI, we then used Raspberry Pi OS (https://www.raspberrypi.com/software/)      

The next step is to create a VLAN based on a ID, our group number XX and a number Y. The ID is 254 + XX

First, we need to acces the network interfaces :

In R shell :
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
The next step is to create an SSH link between our computer and the raspberry . We have few constrains :
1. The SSH key must be an ED25519 one and it must contain the username used on our RPI wich is **rasp**.
2. We must access the UCLouvain Network via the studssh.info.ucl.ac.be gateway
3. After the UCLouvain network we must acces our RPI via another gateway, 130.104.78.202

We open the ssh config file :

In C shell : 
```
nano ~/.ssh/config
```
And copy :
```
Host rasp
    HostName fc00:2142:ff07::2          # IP address of RPI
    User rasp                           # Username used to connect to RPI
    IdentityFile ~/.ssh/id_ed25519      # Path to the private SSH key
    ProxyJump gateway                   # Use the SSH gateway as a jump host

Host gateway
    HostName 130.104.78.202             # IP address of the SSH gateway
    User rasp                           # Username used to connect to the SSH gateway
    IdentityFile ~/.ssh/id_ed25519      # Path to the private SSH key
    ProxyJump UCL

Host UCL
    HostName studssh.info.ucl.ac.be     # IP address of the UCLouvain gateway
    User <Login UCL>                    # Username used to connect to UCLouvain network
    IdentityFile ~/.ssh/id_rsa          # Path to the private SSH key
```
We can simply use **ssh rasp** to connect to our RPI since rasp jump to gateway that jump to the UCL gateway.

## 2. Hardening your RPI
## 3. Networking in a virtual infrastructure
```
sudo apt-get install qemu-user-static
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
docker buildx create --use


sudo containerlab deploy --topo my-topology.clab.yml

sudo brctl addbr br0  # Create the bridge
sudo ip link set br0 up  # Bring the bridge up








sudo docker exec -it <container_id_of_r0> /bin/sh

router isis
  net fc00:2142:XX:00  # Your specific IPv6 prefix
  metric 10
  passive-interface eth0
  passive-interface eth1

echo 1 > /proc/sys/net/ipv6/conf/all/forwarding

vtysh -c "wr"
```
## 4. hosting services in our VI
