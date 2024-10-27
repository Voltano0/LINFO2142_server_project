  GNU nano 7.2                                     README.md *                                            
# Server Project - LINFO2142 : Computer Network
## Context
This project aims to create a resilient server infrastructure using two Raspberry Pis. Since we will work across multiple machines, the location of each command execution will be specified.

The project is divided into four phases. This README outlines the work completed to achieve thoses phases.

## 1. Basic configuration

### Step 1 : installing the OS

To start, we installed a lightweight, Linux-based OS suitable for the Raspberry Pi. We used the *Raspberry Pi Imager* tool to flash the latest version of *Raspberry Pi OS* onto the SD card.
 > Note that usual OS, like ubuntu, can be too heavy for a RPI 

After completing the installation, we are ready to move on.

### Step 2 : creating a VLAN interface

Since our RPIs will be in the UCLouvain server, we need to create a VLAN for access. The VLAN ID is calculated as *254 + XX*, where *XX* is our group number, which in our case is *7*. The IPv6 address assigned to each Raspberry Pi varies: one has *::2/64* and the other *::3/64*.

In RPI shell :
1. Access the network interfaces file:
``` 
sudo nano /etc/network/interfaces
```
2. Configure the file as follows:
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
Since we’re editing the *interfaces* file, this VLAN configuration persists across reboots.

### Step 3 : configure SSH connection

To enable SSH access between our RPI and computer, we must create an *ED25519* SSH key containing the username *rasp* as requested in the guidelines. We generate the key with this command:

In computer shell
```
ssh-keygen -t ed25519 -C "rasp" -f ~/.ssh/id_ed25519_rasp
```
- *-t* specifies the key type, 
- *-C* allow us to add a label
- *-f* specifies the path of the file containing the key.

To finalize the configuration, we must add our public generated key (contained inside the id_ed25519_rasp.pub file) into the *.ssh/authorized_keys* file in our RPI.

### Step 4 : configure SSH configuration file

The SSH connection created in step 3 works when we have direct access to the RPI. For the further phase, once our RPI is placed within the school’s infrastructure, we will need to go through multiple gateways to access it. To achieve this, we configure our SSH config file.

In computer shell :
1. Access the config file
```
nano ~/.ssh/config
```
2. Configure as follow
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
Once configured, we can simply use *ssh rasp* to connect to the RPI, as rasp will automatically jump through the gateway and UCL gateway to reach it.

## 2. Security Hardening

Now that remote access to the RPI is configured, we need to implement security measures to protect the infrastructure. The following steps will do it

### Step 1 & 2 : restrict to pubkey authentification and disable the root access.

To secure SSH access, we’ll disable password-based login, allowing only authentication via SSH keys. Moreover, we'll disanle the root access. To achieve that, we must : 

In RPI shell :
1. Open SSH config file
```
sudo nano /etc/ssh/sshd_config
```
2. Set the following parameters as follow :
```
#To disable tunneled clear text passwords, change to no here!
PasswordAuthentication no
#PermitEmptyPasswords no
ChallengeResponseAuthentication no
PermitRootLogin no
PermitRootLogin prohibit-password
```
- *PasswordAuthentication no*: Only allows login with SSH keys.
- *ChallengeResponseAuthentication no*: Disables interactive authentication, further protecting the server.
- *PermitRootLogin no*: Prevents root login

3. Restart the SSH service
```
sudo systemctl restart ssh
```

### Step 3 : deploying a firewall
To control access, we used UFW (Uncomplicated Firewall), a tool for managing firewall rules. Since UFW was not pre-installed and the RPI was already placed in the infrastructure (without internet access), we downloaded the *.deb* package on our computers and transferred it to the RPI using the *scp* command.

Once the installation is done, we can use the following commands : 
1. Allow SSH connections:
```
sudo ufw allow ssh
```
2. Set the default policies:
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
- *deny incoming*: Blocks all incoming traffic 
- *allow outgoing*: Permits all outgoing traffic
3. Enable UFW to activate the firewall settings:
```
sudo ufw enable
```

### Step 4 : protect from DDos attacks 
Fail2Ban is a tool that automatically bans IP addresses showing signs of abusive behavior. This is why we use it to protect our server from DDos attacks. We use *Fail2Ban* and *rsyslog* for protection against DDoS attacks. Installation followed the same process as described in Step 3.

In RPI shell : 
1. Enable logging :
```
sudo systemctl enable fail2ban.service
```
2. Configure Fail2Ban : 
- Open config file
```
sudo nano /etc/fail2ban/jail.local
```
- Add the following :
```
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 5
bantime = 1200 
```
Where 
- *enabled = true*: activates the rule for SSH.
- *maxretry = 5*: Allows 5 failed login attempts before ban.
- *bantime = 1200*: Sets the ban duration to 1200 seconds.

3. Restart to apply changes
```
sudo systemctl restart fail2ban
```

With these settings, the Raspberry Pi is better protected.

## 3. Networking in a virtual infrastructure

In this phase, we’ll deploy a virtual network infrastructure using Infrastructure as Code (IaC) with the containerlab overlay on Docker. This setup enables services in containers and configures the network connecting them.

### Step 1 : building containers
Since the RPI lacks internet access, we'll need to cross-build Docker containers on our local machine for the RPI’s architecture, wich is *aarch64*.
To achieve such a goal, we do as follow : 

In computer shell :
1. Install and enable multi-platform builds with qemu :
```
sudo apt-get install qemu-user-static
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```
2. Cross build the container named "my-container"
```
docker buildx create --use
docker buildx build --platform linux/arm64 -t my-container .
```
3. Save the container as tar archive
```
docker save my-container > my-container.tar
```
Once it is achieved, we *scp* the tar achive into our RPI. We simply load it after.
 
In RPI shell : 
```
docker load < my-container.tar
```
```
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
