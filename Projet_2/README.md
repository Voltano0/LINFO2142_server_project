# LINFO2142 - Traffic Engineering Using BGP Attributes
## Authors 
- **Jacques Hogge** - 26432000
- **Théo Gustin** - 42052000

## Introduction
BGP Traffic Engineering (BGP-TE) is used to better control the flow of traffic within an Autonomous System (AS) or between AS's. BGP (Border Gateway Protocol) is a protocol used to exchange routing information across the Internet, and it also allows network operators to influence the path that traffic takes to reach its destination.

In this project, we will focus on the BGP attributes that can be manipulated to influence the path that traffic takes. We will use :

- Local Preference
- MED (Multi-Exit Discriminator)
- Communities
- AS Path

## How to run the project
To run the project, just run the bash script `run.sh` in the root folder of the project. This script will start the network topology in Docker containers. To access the different routers, you can use the following command where X is the number of the router (1 to 13) :

```bash
sudo docker exec -it clab-igp-s<X> vtysh
```

same for the hosts where X is the number of the host (1 to 4) :

```bash
sudo docker exec -it clab-igp-h<X> bash
```

if you want to stop the network, you can use the following command :

```bash
sudo containerlab destroy
```

## Network Topology
The network topology used consists of 4 AS (AS65001, AS65002, AS65003, AS65004) and some routers in each AS as shown in the figure below.

![Network Topology](images/PossibleTopo.png)

## Configuration
In each AS, the ISIS routing protocol is used to exchange routing information between routers. iBGP sessions are established between routers in the same AS, and eBGP sessions are established between routers in different AS's. The configuration files for each router can be found in the [clab-igp folder](/clab-igp/).

The paths used by the hosts (H1, H2, H3,H3) in the initial configuration are (You can run the commande `traceroute fc00:2142:<AS>::15` with AS = {1,2,3,4}):

|    	|              H1             	|           H2           	|              H3               | H4                        |
|:--:	|:---------------------------:	|:----------------------:	|:-----------------------------:|:-:                        |
| H1 	|              /              	|  H1->S1->S2->S6->S5-H2    | H1->S1->S4->S3->S8->S10->H3   |   H1->S1->S4->S11->H4     |
| H2 	|  H2->S5->S6->S2-S1->H1        |            /           	|   H2->S5->S7->S8->S10->H3     |   H2->S5->S7-S13->S11->H3 *|
| H3 	|  H3->S10->S8->S3->S4->S1->H1  | H3->S10->S8->S7->S5->H2   |              /            	|H3->S10->S8->S12->S11->H4   |
| H4    |  H4->S11->S4->S1->H1          | H4->S11->S12->S7->S5->H2  | H4-S11->S12->S8->S10->H3      |/                          |

*<em>The path from H2 to H3 have a equal cost path, so the path can be `H2->S5->S7->S12->S11->H3`
 or `H2->S5->S7->S13->S11->H3`. When using the command `traceroute fc00:2142:3::15` we can see that the path is `H2->S5->S7->S13->S11->H3`. We talk about this in the section [Task 2.2](#task-22-scenario-1--). But it just to avoid S12 to have a lot of traffic, we set a MED value lower than the MED value of S12 on S13 to force the traffic to go through S13</em>.

For a visual représentation here is a diagram with the different init routes :

![Initial Routes](images/InitialRoutes.png) 

We can see that the hosts are taking different paths to reach their destination. Always the shortest path is taken by the hosts. We can also see some routes between some AS are never used (e.g S6-S7, S8-S9, S12-13, S9-S10).
Also, S9 never forward packet, this is due to the topology of the network.

## Task 0: ISIS and BGP Configuration
The first task was to configure ISIS and BGP on the routers. The configuration files for each router can be found in the [clab-igp folder](/clab-igp/).

We decided not to use a Route Reflector in this project since there are a maximum of 4 routers in each AS. Instead, we opted for a full mesh of iBGP sessions between the routers in the same AS, and eBGP sessions between routers in different ASes. See the topology diagram above.

For reprensatation here is a plot about the mean latency between the different hosts over different packets size :

![Latency Plot](test/plots/latency.png)

We can see that the latency is increasing with the packet size. This is due to the fact that the routers have to fragment the packets when they are too big.This is why the latency is increasing with the packet size. 
Also our sample of data is not big enough to have a good representation of the latency we should do more tests to have a better idea of the latency between the different hosts. 
It's only here to show that the latency is increasing with the packet size. And that the distance between the hosts is not the only factor that can influence the latency (even if it affect, there is also the number of router it pass, ...).

## Task 1: Local Preference
The Local Preference (LP or LOCAL_PREF) attribute is used to influence the outbound from a local AS. It is used to determine the preferred exit point from an AS when there are multiple exit points. The higher the Local Preference value, the more preferred the route. The default value for Local Preference is 100. It's important to note that Local Preference is only used within an AS using iBGP and is not propagated to other AS's. If a AS receive a LOCAL_PREF from a other AS it must ignore it.

### Task 1.1: Default Configuration
In the default configuration, the LP value is set to 100 on all routers. The paths used by the hosts in the default configuration are shown in the table above in section [Configuration](#configuration).

### Task 1.2:  Scenario 1 : Going through a specific AS
In this scenario, we wanted to force traffic to go through a specific AS. It can be for security reasons or to use a specific service provided by the AS. In this case, we wanted to force traffic from H1 to AS65002 to go through AS65003 before.

In our case the init path used by AS65001 to AS65003 is lister in the table in the section [Configuration](#configuration). Know we want to force to path going throug AS65002, to do it we can config the router S2 (the only one connected to AS65002) and setup a route-map LOCAL_PREF by doing this

```bash

route-map SET-LOCAL-PREF permit 10
 set local-preference 200
exit

router bgp 65001
    ...
 !
 address-family ipv6 unicast
    ...
  neighbor fc00:2143:1::6 route-map SET-LOCAL-PREF in
    ...
 exit-address-family
exit
```

After this configuration, We can see the new path taken by the hosts. We can see the routing table of the router S2 :

```bash
S2: show ip bgp ipv6 unicast
    Network          Next Hop                   Metric LocPrf Weight        Path
*>  fc00:2142:1::/64 ::                         0              32768            i
* i                  fc00:2142:1::3             0        100       0            i
*>  fc00:2142:2::/64 fe80::a8c1:abff:fe29:5bd9  0        200       0      65002 i
*>  fc00:2142:3::/64 fe80::a8c1:abff:fe29:5bd9           200       0      65002 65003 i
*>  fc00:2142:4::/64 fe80::a8c1:abff:fe29:5bd9           200       0      65002 65004 i
```

Or we can see the new path taken by the hosts by using the command `traceroute` :

```bash
h1: traceroute fc00:2142:3::11 # H1 -> H3
traceroute to fc00:2142:3::11 (fc00:2142:3::11), 30 hops max, 72 byte packets
 1  fc00:2142:1::11 (fc00:2142:1::11)  0.018 ms  0.019 ms  0.012 ms # S1
 2  fc00:2142:1::2 (fc00:2142:1::2)  0.009 ms  0.142 ms  0.023 ms # S2
 3  fc00:2143:1::6 (fc00:2143:1::6)  0.009 ms  0.019 ms  0.019 ms # S6
 4  fc00:2142:2::7 (fc00:2142:2::7)  0.010 ms  0.017 ms  0.331 ms # S7 
 5  fc00:2143:5::8 (fc00:2143:5::8)  0.012 ms  0.016 ms  0.008 ms # S8
 6  fc00:2142:3::11 (fc00:2142:3::11)  0.006 ms  0.018 ms  0.014 ms # S10->H3
```

With these commands, we can see that the traffic is now going through AS65002 before going to AS65003. 

But by applying the local preference on the router S2, we force all the traffic going through this router to go through AS65002. For example, the traffic from H1 to H4 is also going through AS65002 (test with the commande in H1 : `traceroute fc00:2142:4::11`). To avoid this, we can apply the same local preference on the router S3.

In S4 :
```bash
route-map SET-LOCAL-PREF permit 10
 set local-preference 200
exit

router bgp 65001
    ...
 !
 address-family ipv6 unicast
    ...
  neighbor fc00:2143:2::3 route-map SET-LOCAL-PREF in
    ...
 exit-address-family
exit
```
Note that it works only because the path AS65001->AS65002 is shorter than the path AS65001->AS65003->AS65004. Also by setting the local preference on S3 will overdrive the desire to use AS65002 via LOCAL_PREF setup at router S2.

Here is a proof that the traffic from H1 to H4 is now going through AS65003 :

```bash
h1:/ traceroute fc00:2142:3::11 #H1 -> H3
traceroute to fc00:2142:3::11 (fc00:2142:3::11), 30 hops max, 72 byte packets
 1  fc00:2142:1::11 (fc00:2142:1::11)  0.018 ms  0.031 ms  0.016 ms # S1
 2  fc00:2142:1::2 (fc00:2142:1::2)  0.010 ms  0.037 ms  0.013 ms # S2
 3  fc00:2143:1::6 (fc00:2143:1::6)  0.016 ms  0.018 ms  0.010 ms # S6
 4  fc00:2142:2::7 (fc00:2142:2::7)  0.008 ms  0.018 ms  0.013 ms # S7
 5  fc00:2143:5::8 (fc00:2143:5::8)  0.009 ms  0.018 ms  0.010 ms # S8
 6  fc00:2142:3::11 (fc00:2142:3::11)  0.012 ms  0.017 ms  0.013 ms # S10->H3

h1:/ traceroute fc00:2142:4::11 #H1 -> H4
traceroute to fc00:2142:4::11 (fc00:2142:4::11), 30 hops max, 72 byte packets
 1  fc00:2142:1::11 (fc00:2142:1::11)  0.019 ms  0.014 ms  0.010 ms # S1
 2  fc00:2142:1::4 (fc00:2142:1::4)  0.007 ms  0.018 ms  0.006 ms # S4
 3  fc00:2142:4::11 (fc00:2142:4::11)  0.006 ms  0.016 ms  0.010 ms # S11->H4
```

Or via the routing table of the router S8:
```bash
S8:/ show ip bgp ipv6 unicast

     Network          Next Hop                  Metric LocPrf Weight Path
 *>  fc00:2142:1::/64 fe80::a8c1:abff:fe5b:1c6a        150    0      65002 65001 i
 *                    fe80::a8c1:abff:fe40:1185 0             0      65001 i
 *>  fc00:2142:2::/64 fe80::a8c1:abff:fe5b:1c6a 0      150    0      65002 i
 *                    fe80::a8c1:abff:fe40:1185               0      65001 65002 i
 *>  fc00:2142:3::/64 ::                        0             32768  i
 * i                  fc00:2142:3::9            0      100    0      i
 *>  fc00:2142:4::/64 fe80::a8c1:abff:fe5b:1c6a        150    0      65002 65001 65004 i
 *                    fe80::a8c1:abff:fe40:1185               0      65001 65004 i

```
### Task 1.3: Scenario 2  : Backup path in case of failure
In this scenario, we wanted to set up a backup path in case of failure of the main path. For example, we want to setup the link `AS65003->AS65002->AS65001->AS65004` as a backup link of the link AS65003->65004 (We can pass by AS65001 without AS65002 but we force for some reasons). For example The main link `S8->S12` fail and we know that the links `S7->S12` and `S7->S13` is under maintenance with a lot of congestion or latency and we want to pass by AS65002 before going to AS65004 cause of some reasons. We can setup the router S8 with a local preférence lower than the local pref of `S8-S12` but higher than the normal, and then setup the router S6 with a local preference higher than the préférence of `S7->S12` and `S7->S13`.

Here is a implémentation of this scenario (see folder [Configs/Task1/Senario 2](/Configs/Task1/Scenario%202/)):

in `S8` :
```bash
route-map PERMIT-ALL permit 10
exit
!
route-map SET-LOCAL-PREF permit 10
 set local-preference 200
exit
!
route-map BACKUP permit 10
 set local-preference 150
exit
!

...
router bgp 65003
 ...
 !
 address-family ipv6 unicast
  ...
  neighbor fc00:2143:3::3 activate
  neighbor fc00:2143:3::3 route-map PERMIT-ALL in 
  neighbor fc00:2143:3::3 route-map PERMIT-ALL out
  neighbor fc00:2143:5::7 activate
  neighbor fc00:2143:5::7 route-map BACKUP in # backup path S8->S7
  neighbor fc00:2143:5::7 route-map PERMIT-ALL out
  neighbor fc00:2143:8::12 activate
  neighbor fc00:2143:8::12 route-map SET-LOCAL-PREF in # Main path S8->S12
  neighbor fc00:2143:8::12 route-map PERMIT-ALL out
 exit-address-family
exit
!
```

in `S6` :
```bash
route-map PERMIT-ALL permit 10
exit
!
route-map BACKUP permit 10
 set local-preference 150
exit
!

...
router bgp 65002
 ...
 !
 address-family ipv6 unicast
  ...
  neighbor fc00:2143:1::2 route-map BACKUP in
  neighbor fc00:2143:1::2 route-map PERMIT-ALL out
 exit-address-family
exit
!
```

Now we can simulate a failure in the link `S8->S12` by using this command in configuraion of the router S8 :

```bash
S8 :/ conf t
s8 (config) :/ interface eth-s12
s8 (config-if) :/ shutdown
```

Now we can see that the traffic is before the link failure  (`H3->S10->S8->S12->S11->H4`):

```bash
h3:/ traceroute fc00:2142:4::11 # H3 -> H4
traceroute to fc00:2142:4::11 (fc00:2142:4::11), 30 hops max, 72 byte packets
 1  fc00:2142:3::11 (fc00:2142:3::11)  0.016 ms  0.019 ms  0.050 ms # S10
 2  fc00:2142:3::8 (fc00:2142:3::8)  0.014 ms  0.028 ms  0.010 ms # S8
 3  fc00:2143:8::12 (fc00:2143:8::12)  0.009 ms  0.017 ms  0.011 ms # S12
 4  fc00:2142:4::11 (fc00:2142:4::11)  0.009 ms  0.019 ms  0.010 ms # S11->H4
```

And after the link failure `S8->S12` we can see that the traffic is now going through the backup path (`H3->S10->S8->S7->S6->S2->S3->S12->S11->H4`):

```bash
h3:/ traceroute fc00:2142:4::11 # H3 -> H4
traceroute to fc00:2142:4::11 (fc00:2142:4::11), 30 hops max, 72 byte packets
 1  fc00:2142:3::11 (fc00:2142:3::11)  0.017 ms  0.019 ms  0.012 ms # S10
 2  fc00:2142:3::8 (fc00:2142:3::8)  0.018 ms  0.029 ms  0.011 ms # S8
 3  fc00:2143:5::7 (fc00:2143:5::7)  0.011 ms  0.021 ms  0.012 ms # S7
 4  fc00:2142:2::6 (fc00:2142:2::6)  0.010 ms  0.018 ms  0.010 ms # S6
 5  fc00:2143:1::2 (fc00:2143:1::2)  0.009 ms  0.019 ms  0.011 ms # S2
 6  fc00:2142:1::3 (fc00:2142:1::3)  0.011 ms  0.020 ms  0.011 ms # S3
 7  fc00:2143:4::12 (fc00:2143:4::12)  0.008 ms  0.017 ms  0.011 ms # S12
 8  fc00:2142:4::11 (fc00:2142:4::11)  0.008 ms  0.018 ms  0.011 ms # S11->H4
```

With this configuration, we can see that the traffic is now going through the backup path in case of failure of the main path.
And by restoring the link `S8->S12` we can see that the traffic is now going through the main path.

```bash
S8 :/ conf t
s8 (config) :/ interface eth-s12
s8 (config-if) :/ no shutdown
```

```bash
h3:/ traceroute fc00:2142:4::11 # H3 -> H4
traceroute to fc00:2142:4::11 (fc00:2142:4::11), 30 hops max, 72 byte packets
 1  fc00:2142:3::11 (fc00:2142:3::11)  0.016 ms  0.019 ms  0.050 ms # S10
 2  fc00:2142:3::8 (fc00:2142:3::8)  0.014 ms  0.028 ms  0.010 ms # S8
 3  fc00:2143:8::12 (fc00:2143:8::12)  0.009 ms  0.017 ms  0.011 ms # S12
 4  fc00:2142:4::11 (fc00:2142:4::11)  0.009 ms  0.019 ms  0.010 ms # S11->H4
```

```bash
     Network          Next Hop                    Metric LocPrf Weight Path
 *>  fc00:2142:1::/64 fe80::a8c1:abff:fe24:abbb          200      0    65004 65001 i
 *                    fe80::a8c1:abff:fe5b:1c6a          150      0    65002 65001 i
 *                    fe80::a8c1:abff:fe40:1185     0             0    65001 i
 *>  fc00:2142:2::/64 fe80::a8c1:abff:fe24:abbb          200      0    65004 65002 i
 *                    fe80::a8c1:abff:fe5b:1c6a     0    150      0    65002 i
 *                    fe80::a8c1:abff:fe40:1185                   0    65001 65002 i
 *>  fc00:2142:3::/64 ::                            0           32768  i
 * i                  fc00:2142:3::9                0    100      0    i
 *>  fc00:2142:4::/64 fe80::a8c1:abff:fe24:abbb     0    200      0    65004 i
 *                    fe80::a8c1:abff:fe5b:1c6a          150      0    65002 65001 65004 i
 *                    fe80::a8c1:abff:fe40:1185                   0    65001 65004 i

```

If The link `S7-S12` or `S7-S13` is not under maintenance anymore, we can restore the normal path by removing the local preference on the router S6.

### Task 1.4: Load Balancing
We can also use the Local Preference attribute to perform load balancing between multiple paths. For example, we can set the Local Preference value to 200 on one path and 200 on the other path. This will cause the traffic to be distributed between the two paths based on the Local Preference values. But to acheive this we need two paths with the same AS Path length and our BGP session need to be configured to allow multiple paths. Which is a little out of the scope of our project

## Task 2: MED
The Multi-Exit Discriminator (MED) attribute is used to influence the inbound traffic from a neighboring AS. It is used to determine the preferred entry point into an AS when there are multiple entry points. The lower the MED value, the more preferred the route. The default value for MED is 0. It's important to note that MED is propagated to all routers within the neighbor AS. but it's not passed alon to any other AS.

### Task 2.1: Default Configuration
In the default configuration, the MED value is set to 0 on all routers. The paths used by the hosts in the default configuration are shown in the table above in section [Configuration](#configuration).

### Task 2.2: Scenario 1 : ???
In this scenario we have the router `S7` that have 2 connection in the same AS (`S7->S12` and `S7->S13`) and we want to force the traffic to go through the link `S7->S13` (the less used router). To do this we can setup the router `S13` with a MED value lower than the MED value of the router `S12`.

Here is a implementation of this scenario (see folder [Configs/Task2/Senario 1](/Configs/Task2/Scenario%201/)):
in `S12`:
```bash
!
route-map PERMIT-ALL permit 10
exit
!
route-map SET-MED permit 10
 set metric 50
exit
!
router bgp 65004
 ...
 !
 address-family ipv6 unicast
  ...
  neighbor fc00:2143:4::3 activate
  neighbor fc00:2143:4::3 route-map PERMIT-ALL in
  neighbor fc00:2143:4::3 route-map PERMIT-ALL out
  neighbor fc00:2143:6::7 activate
  neighbor fc00:2143:6::7 route-map PERMIT-ALL in
  neighbor fc00:2143:6::7 route-map SET-MED out
  neighbor fc00:2143:8::8 activate
  neighbor fc00:2143:8::8 route-map PERMIT-ALL in
  neighbor fc00:2143:8::8 route-map PERMIT-ALL out
 exit-address-family
exit
!
```

in `S13`:
```bash
!
route-map PERMIT-ALL permit 10
exit
!
route-map SET-MED permit 10
 set metric 10
exit
!
router bgp 65004
 ...
 !
 address-family ipv6 unicast
  ...
  neighbor fc00:2143:7::7 activate
  neighbor fc00:2143:7::7 route-map PERMIT-ALL in
  neighbor fc00:2143:7::7 route-map SET-MED out
 exit-address-family
exit
!
```

Now we can see that the traffic is going through the link `S7->S13`:

```bash
s7:/ show ip bgp ipv6 unicast

Network                                Next Hop                                  Metric LocPrf Weight Path
 *>i fc00:2142:1::/64                fc00:2142:2::6                                 0    100      0 65001 i
 *                                   fe80::a8c1:abff:fedb:5238                                    0 65003 65001 i
 *                                   fe80::a8c1:abff:feef:17ae                     10             0 65004 65001 i
 *                                   fe80::a8c1:abff:fe5e:de43                     50             0 65004 65001 i
 *>  fc00:2142:2::/64                ::                                             0         32768 i
 * i                                 fc00:2142:2::5                                 0    100      0 i
 * i                                 fc00:2142:2::6                                 0    100      0 i
 *>  fc00:2142:3::/64                fe80::a8c1:abff:fedb:5238                      0             0 65003 i
 *                                   fe80::a8c1:abff:feef:17ae                     10             0 65004 65003 i
 *                                   fe80::a8c1:abff:fe5e:de43                     50             0 65004 65003 i
 *>  fc00:2142:4::/64                fe80::a8c1:abff:feef:17ae                     10             0 65004 i
 *                                   fe80::a8c1:abff:fe5e:de43                     50             0 65004 i
 *                                   fe80::a8c1:abff:fedb:5238                                    0 65003 65004 i
```
We can see that the traffic to AS65004 is going through the link `S7->S13` with a MED value of 10.

This is a simple example show how we can use the MED attribute to influence the path that traffic takes. For example, here where S12 has 3 exit points and S13 has only one exit point same as S12, we can use the MED attribute to force the traffic to go through the link `S7->S13` and to relieve traffic congestion on S12.
