  GNU nano 7.2                                     README.md *                                            
# Server Project - LINFO2142 : Computer Network
## Context
This project aims to create a resilient server infrastructure using two Raspberry Pis. Since we will work across multiple machines, the location of each command execution will be specified.

The project is divided into four phases. This README outlines the work completed to achieve thoses phases.

## 1. Basic configuration
### Step 1 : installing the OS

To start, we installed a lightweight, Linux-based OS suitable for the Raspberry Pi. We used the Raspberry Pi Imager tool to flash the latest version of Raspberry Pi OS onto the SD card.
 > Note that usual OS like ubuntu can be too heavy for a RPI 
After completing the installation, we are ready to move on.

### Step 2 : creating a VLAN interface
Since our RPI's will be in the UCLouvain server, we need to create a VLAN to acces it. The ID of this vlan is calculate via 254 + XX where XX is our group number, wich is *7*. The IPv6 address of the VLAN differ from the RPI, one have the ::2/64 and the other have ::3/64.

In RPI shell :
1. Acces the Network interfaces files
``` 
sudo nano /etc/network/interfaces
```
2. Configure the file.
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
SInce we are working inside the interfaces file, the configuration of our VLAN is persistent across reboots.

### Step 3 : configure SSH connection

To configure a connection between our RPI and our computers we must create a key following the guidelines : ED25519 key containing the username used on our RPI. To create such a key, we use this commands :

In computer shell
```
ssh-keygen -t ed25519 -C "rasp" -f ~/.ssh/id_ed25519_rasp

```
In this command,*-t* specifies the key type, *-C* allow us to add a label and *-f* allow us to choose the path of the file containing the key.

To finalize the configuration, we must add our public generated key into the *.ssh/authorized_keys* file in our RPI.










1. The SSH key must be an ED25519 one and it must contain the username used on our RPI wich is **rasp**.
2. We must establish a multi-hop SSh access through two gateways : 
  1. UCLouvain : studdssh.info.ucl.ac.be
  2. INfrastructure : 130.104.78.202

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
