# LINFO2142 - Traffic Engineering Using BGP Attributes
## Authors 
- **Jacques Hogge** - 26432000
- **Théo Gustin** - 42052000

## Introduction
BGP Traffic Engineering (BGP-TE) is used to better control the flow of traffic within an Autonomous System (AS) or between AS's. BGP (Border Gateway Protocol) is a protocol used to exchange routing information across the Internet, and it also allows network operators to influence the path that traffic takes to reach its destination.

In this project, we will focus on the BGP attributes that can be manipulated to influence the path that traffic takes. We will use :

- **Local Preference**
- **MED (Multi-Exit Discriminator)**
- **Communities**
- **AS Path**

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
The network topology used consists of 4 AS (`AS65001, AS65002, AS65003, AS65004)` and some routers in each AS as shown in the figure below.

![Network Topology](images/PossibleTopo.png)

## Configuration
In each AS, the ISIS routing protocol is used to exchange routing information between routers. iBGP sessions are established between routers in the same AS, and eBGP sessions are established between routers in different AS's. The configuration files for each router can be found in the [clab-igp folder](/clab-igp/).

The paths used by the hosts (`H1, H2, H3,H3`) in the initial configuration are (You can run the commande `traceroute fc00:2142:<AS>::15` with AS = {1,2,3,4}):

|    	|              H1             	|           H2           	|              H3               | H4                        |
|:--:	|:---------------------------:	|:----------------------:	|:-----------------------------:|:-:                        |
| H1 	|              /              	|  H1->S1->S2->S6->S5-H2    | H1->S1->S4->S3->S8->S10->H3   |   H1->S1->S4->S11->H4     |
| H2 	|  H2->S5->S6->S2-S1->H1        |            /           	|   H2->S5->S7->S8->S10->H3     |   H2->S5->S7-S13->S11->H3 *|
| H3 	|  H3->S10->S8->S3->S4->S1->H1  | H3->S10->S8->S7->S5->H2   |              /            	|H3->S10->S8->S12->S11->H4   |
| H4    |  H4->S11->S4->S1->H1          | H4->S11->S12->S7->S5->H2  | H4-S11->S12->S8->S10->H3      |/                          |

*<em>The path from H2 to H3 have a equal cost path, so the path can be `H2->S5->S7->S12->S11->H3`
 or `H2->S5->S7->S13->S11->H3`. When using the command `traceroute fc00:2142:3::15` we can see that the path is `H2->S5->S7->S13->S11->H3`. We talk about this in the section [Task 2.2](#task-22-scenario-1--). But it just to avoid `S12` to have a lot of traffic, we set a MED value lower than the MED value of `S12` on `S13` to force the traffic to go through S13</em>.

For a visual représentation here is a diagram with the different init routes :

![Initial Routes](images/InitialRoutes.png) 

We can see that the hosts are taking different paths to reach their destination. Always the shortest path is taken by the hosts. We can also see some routes between some AS are never used (e.g S6-S7, S8-S9, S12-13, S9-S10).
Also, S9 never forward packet, this is due to the topology of the network.

### Choice of the best route
It's important to note that, in BGP, the best route is chosen based on the following criteria (in order of priority) :
|order|Criteria|Description|
|:---:|:---:|:---:|
| 1   |weight| Administrative Local Preference|
| 2   |LOCAL_PREF| Inside AS preference|
| 3   |AS_PATH| Shortest AS Path|
| 4   |Origin| IGP > EGP > Incomplete| 
| 5   |MED| External preference|
| 6   |external| Prefer eBGP over iBGP|
| 7   |IGP| Shortest IGP path|
| 8   |Router ID| Lowest router ID| 

## Scripts of configuration
We implemented a bash script for each scenarios of each tasks. The scripts are located in the folder [ScriptConfig](/ScriptConfig/). Each script will show the route table  and the traceroute before and after the configuration of the routers.

To run the scripts, you can use the following command : 

```bash
sudo bash ./<script_name>.sh
```

## Task 0: Initial and IP's configuration
The first task was to configure ISIS and BGP on the routers. The configuration files for each router can be found in the [clab-igp folder](/clab-igp/).

We decided not to use a Route Reflector in this project since there are a maximum of 4 routers in each AS. Instead, we opted for a full mesh of iBGP sessions between the routers in the same AS, and eBGP sessions between routers in different ASes. See the topology diagram above.

To work with a topology, it is important to choose an appropriate IP range. We can divide our IPv6 addresses into two categories:

### 1. **fc00:2142:X::Y**
This range is used for IS-IS and iBGP configuration. 
- **X** represents the AS number (1, 2, 3, 4).
- **Y** represents the switch number. Y can also be equal to 15 to represent the host. 

Below is the detailed breakdown of the full IP range:

| AS1               | AS2               | AS3               | AS4               |
|:------------------:|:-----------------:|:-----------------:|:-----------------:|
| fc00:2142:1::1    | fc00:2142:2::5    | fc00:2142:3::8    | fc00:2142:4::11   |
| fc00:2142:1::2    | fc00:2142:2::6    | fc00:2142:3::9    | fc00:2142:4::12   |
| fc00:2142:1::3    | fc00:2142:2::7    | fc00:2142:3::10   | fc00:2142:4::13   |
| fc00:2142:1::4    | /  | /  | /|
| fc00:2142:1::15   | fc00:2142:2::15|fc00:2142:3::15 | fc00:2142:4::15 |

### 2. **fc00:2143:A::B**
This range is used exclusively for eBGP. 
- **A** represents the link number.
- **B** represents the switch number.

The link number is chosen according to the schema below:
![My Diagram](/link_numbers.jpg)

## Task 1: Local Preference
The Local Preference (LP or LOCAL_PREF) attribute is used to influence the outbound from a local AS. It is used to determine the preferred exit point from an AS when there are multiple exit points. The higher the Local Preference value, the more preferred the route. The default value for Local Preference is 100. It's important to note that Local Preference is only used within an AS using iBGP and is not propagated to other AS's. If a AS receive a LOCAL_PREF from a other AS it must ignore it.

### Task 1.1: Default Configuration
In the default configuration, the LP value is set to 100 on all routers. The paths used by the hosts in the default configuration are shown in the table above in section [Configuration](#configuration).

### Task 1.2:  Scenario 1 : Going through a specific AS
In this scenario, we wanted to force traffic to go through a specific AS. It can be for security reasons or to use a specific service provided by the AS, or to avoid a other AS. In this case, we wanted to force traffic from `H1` to `AS65002` to go through `AS65003` before.

In our case the init path used by `AS65001` to `AS65003` is listed in the table in the section [Configuration](#configuration). 
Or listed below via the route-table
```bash
S2:/ show ip bgp ipv6 unicast
          Network            Next Hop                         Metric LocPrf Weight Path
 *>  fc00:2142:1::/64        ::                                    0         32768 i
 * i                         fc00:2142:1::3                        0    100      0 i
 *>  fc00:2142:2::/64        fe80::a8c1:abff:fee1:6452             0             0 65002 i
 *>i fc00:2142:3::/64        fc00:2142:1::3                        0    100      0 65003 i
 *                           fe80::a8c1:abff:fee1:6452                           0 65002 65003 i
 *>i fc00:2142:4::/64        fc00:2142:1::3                        0    100      0 65004 i
 * i                         fc00:2142:1::4                        0    100      0 65004 i
 *                           fe80::a8c1:abff:fee1:6452                           0 65002 65004 i
```

Or the path used by `H1` to `H3` is :
```bash
h1:/ traceroute fc00:2142:3::15 # H1 -> H3
traceroute to fc00:2142:3::15 (fc00:2142:3::15), 30 hops max, 72 byte packets # H1
 1  fc00:2142:1::11 (fc00:2142:1::11)  0.015 ms  0.020 ms  0.036 ms # S1
 2  fc00:2142:1::4 (fc00:2142:1::4)  0.010 ms  0.027 ms  0.009 ms # S4
 3  fc00:2142:1::3 (fc00:2142:1::3)  0.008 ms  0.019 ms  0.011 ms # S3
 4  fc00:2143:3::8 (fc00:2143:3::8)  0.027 ms  0.019 ms  0.011 ms # S8
 5  fc00:2142:3::10 (fc00:2142:3::10)  0.008 ms  0.019 ms  0.014 ms # S10
 6  fc00:2142:3::15 (fc00:2142:3::15)  0.010 ms  0.020 ms  0.011 ms # H3
```

Knowing we want to force to path going throug `AS65002`, to do it we can configure the router `S2` (the only one connected to `AS65002`) and setup a `route-map LOCAL_PREF` by doing this configuration in the router `S2`.

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
We set a route-map `SET-LOCAL-PREF` with a local preference of 200. We apply this route-map to the neighbor `fc00:2143:1::6` in the direction `in`. This will force the traffic from `AS65001` to `AS65002` to go through `AS65003` before going to `AS65002`.
After this configuration and convergence of the networkd , We can see the new path taken by the hosts. We can see the routing table of the router `S2` :

```bash
S2: show ip bgp ipv6 unicast
      Network              Next Hop                      Metric LocPrf Weight Path
 *>  fc00:2142:1::/64      ::                              0            32768 i
 * i                       fc00:2142:1::3                  0       100      0 i
 *>  fc00:2142:2::/64      fe80::a8c1:abff:fe1e:4606       0       200      0 65002 i
 *>  fc00:2142:3::/64      fe80::a8c1:abff:fe1e:4606               200      0 65002 65003 i
 *>  fc00:2142:4::/64      fe80::a8c1:abff:fe1e:4606               200      0 65002 65004 i

```
We this table we see that all path are going through `AS65002` via router `S2`.
Or we can see the new path taken by the hosts by using the command `traceroute` :

```bash
h1: traceroute fc00:2142:3::11 # H1 -> H3
traceroute to fc00:2142:3::15 (fc00:2142:3::15), 30 hops max, 72 byte packets # H1
 1  fc00:2142:1::11 (fc00:2142:1::11)  0.180 ms  0.018 ms  0.011 ms # S1
 2  fc00:2142:1::2 (fc00:2142:1::2)  0.053 ms  0.025 ms  0.010 ms # S2
 3  fc00:2143:1::6 (fc00:2143:1::6)  0.008 ms  0.017 ms  0.010 ms # S6
 4  fc00:2142:2::7 (fc00:2142:2::7)  0.007 ms  0.018 ms  0.011 ms # S7
 5  fc00:2143:5::8 (fc00:2143:5::8)  0.009 ms  0.019 ms  0.011 ms # S8
 6  fc00:2142:3::10 (fc00:2142:3::10)  0.010 ms  0.017 ms  0.013 ms # S10
 7  fc00:2142:3::15 (fc00:2142:3::15)  0.009 ms  0.017 ms  0.011 ms # H3
```

With these commands, we can see that the traffic is now going through `AS65002` before going to `AS65003`. 

But by applying the local preference on the router `S2`, we force all the traffic going through this router to go through `AS65002`. For example, the traffic from `H1` to `H4` is also going through `AS65002` (test with this command in H1 : `traceroute fc00:2142:4::15`). 
```bash
h1:/ traceroute fc00:2142:3::15 # H1 -> H3
traceroute to fc00:2142:3::15 (fc00:2142:3::15), 30 hops max, 72 byte packets # H1
 1  fc00:2142:1::11 (fc00:2142:1::11)  0.018 ms  0.018 ms  0.028 ms # S1
 2  fc00:2142:1::2 (fc00:2142:1::2)  0.177 ms  0.027 ms  0.021 ms # S2
 3  fc00:2143:1::6 (fc00:2143:1::6)  0.010 ms  0.019 ms  0.011 ms # S6
 4  fc00:2142:2::7 (fc00:2142:2::7)  0.008 ms  0.021 ms  0.014 ms # S7
 5  fc00:2143:5::8 (fc00:2143:5::8)  0.011 ms  0.018 ms  0.011 ms # S8
 6  fc00:2142:3::10 (fc00:2142:3::10)  0.009 ms  0.023 ms  0.014 ms # S10
 7  fc00:2142:3::15 (fc00:2142:3::15)  0.008 ms  0.018 ms  0.015 ms # H3
```


To avoid this, we can apply the same local preference on the router `S4`.

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
Note that it works only because the path `AS65001->AS65002` is shorter than the path `AS65001->AS65003->AS65004`. Also by setting the local preference on `S3` will overdrive the desire to use AS65002 via LOCAL_PREF setup at router `S2`. This is due to the BGP choice of the best route talked in the section [Choice of the best route](#choice-of-the-best-route).

Here is a proof that the traffic from H1 to H4 is now going directly through AS65003 :

```bash
h1: traceroute fc00:2142:3::15 # H1 -> H3
traceroute to fc00:2142:3::15 (fc00:2142:3::15), 30 hops max, 72 byte packets # H1
 1  fc00:2142:1::11 (fc00:2142:1::11)  0.180 ms  0.018 ms  0.011 ms # S1
 2  fc00:2142:1::2 (fc00:2142:1::2)  0.053 ms  0.025 ms  0.010 ms # S2
 3  fc00:2143:1::6 (fc00:2143:1::6)  0.008 ms  0.017 ms  0.010 ms # S6
 4  fc00:2142:2::7 (fc00:2142:2::7)  0.007 ms  0.018 ms  0.011 ms # S7
 5  fc00:2143:5::8 (fc00:2143:5::8)  0.009 ms  0.019 ms  0.011 ms # S8
 6  fc00:2142:3::10 (fc00:2142:3::10)  0.010 ms  0.017 ms  0.013 ms # S10
 7  fc00:2142:3::15 (fc00:2142:3::15)  0.009 ms  0.017 ms  0.011 ms # H3

h1:/# traceroute fc00:2142:4::15 # H1 -> H4
traceroute to fc00:2142:4::15 (fc00:2142:4::15), 30 hops max, 72 byte packets # H1
 1  fc00:2142:1::11 (fc00:2142:1::11)  0.017 ms  0.018 ms  0.011 ms # S1
 2  fc00:2142:1::4 (fc00:2142:1::4)  0.009 ms  0.028 ms  0.012 ms # S4
 3  fc00:2143:2::11 (fc00:2143:2::11)  0.081 ms  0.019 ms  0.013 ms # S11
 4  fc00:2142:4::15 (fc00:2142:4::15)  0.015 ms  0.008 ms  0.004 ms # H4
```

Or via the routing table of the router `S4`:
```bash
S8:/ show ip bgp ipv6 unicast

     Network             Next Hop               Metric LocPrf Weight Path
 *>i fc00:2142:1::/64    fc00:2142:1::3              0    100      0 i
 * i                     fc00:2142:1::2              0    100      0 i
 *>i fc00:2142:2::/64    fc00:2142:1::2              0    200      0 65002 i
 *                       fe80::a8c1:abff:fecf:218         200      0 65004 65002 i
 *>  fc00:2142:3::/64    fe80::a8c1:abff:fecf:218         200      0 65004 65003 i
 * i                     fc00:2142:1::2                   200      0 65002 65003 i
 *>  fc00:2142:4::/64    fe80::a8c1:abff:fecf:218    0    200      0 65004 i
```
We can also setup a Local-pref of 150 on the router `S4` for example. All trafic is goind out by the router `S2` but in case of failure of the link `S2->S6`. This will force the traffic to go through `AS65004` by `S4->S11` and keep the avoiding the traffic to go directly to `AS65003` by `S3->S8`. See the scenario 2 in the section [Task 1.3](#task-13-scenario-2-backup-path-in-case-of-failure) for more details.

### Task 1.3: Scenario 2: Backup path in case of failure
In this scenario, we wanted to set up a backup path in case of failure of the main path. For example, we want to setup the link `AS65003->AS65002->AS65001->AS65004` as a backup link of the link `AS65003->65004` (We can pass by `AS65001` without `AS65002` but we force for some reasons). For example The main link `S8->S12` fail and we know that the links `S7->S12` and `S7->S13` is under maintenance with a lot of congestion or latency and we want to pass by `AS65002` before going to `AS65004` cause of some reasons. We can setup the router `S8` with a local preference lower than the local pref of `S8-S12` but higher than the normal(100), and then setup the router `S6` with a local preference higher than the préférence of `S7->S12` and `S7->S13`.

Reminder that the init paths are listed in the table in the section [Configuration](#configuration).

Here is a implémentation of this scenario:

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

after configuring the routers `S8` and `S6` we can see that the traffic stay the same as the init configuration.

```bash
s8:/ show ip bgp ipv6 unicast
     Network                 Next Hop                          Metric LocPrf Weight Path
 *>  fc00:2142:1::/64        fe80::a8c1:abff:feb6:fa4c                   200      0 65004 65001 i
 *                           fe80::a8c1:abff:fe75:f72b                   150      0 65002 65001 i
 *                           fe80::a8c1:abff:fe69:81f9              0             0 65001 i
 *>  fc00:2142:2::/64        fe80::a8c1:abff:feb6:fa4c                   200      0 65004 65002 i
 *                           fe80::a8c1:abff:fe75:f72b              0    150      0 65002 i
 *                           fe80::a8c1:abff:fe69:81f9                            0 65001 65002 i
 *>  fc00:2142:3::/64        ::                                     0         32768 i
 * i                         fc00:2142:3::9                         0    100      0 i
 *>  fc00:2142:4::/64        fe80::a8c1:abff:feb6:fa4c              0    200      0 65004 i
 *                           fe80::a8c1:abff:fe75:f72b                   150      0 65002 65001 65004 i
 *                           fe80::a8c1:abff:fe69:81f9                            0 65001 65004 i
```
Or via traceroute :
```bash
h3:/ traceroute fc00:2142:4::11 # H3 -> H4
traceroute to fc00:2142:4::15 (fc00:2142:4::15), 30 hops max, 72 byte packets #H3
 1  fc00:2142:3::11 (fc00:2142:3::11)  0.009 ms  0.020 ms  0.029 ms # S10
 2  fc00:2142:3::8 (fc00:2142:3::8)  0.211 ms  0.020 ms  0.011 ms # S8
 3  fc00:2143:8::12 (fc00:2143:8::12)  0.009 ms  0.019 ms  0.377 ms # S12
 4  fc00:2142:4::11 (fc00:2142:4::11)  0.265 ms  0.019 ms  0.012 ms # S11
 5  fc00:2142:4::15 (fc00:2142:4::15)  0.636 ms  0.016 ms  0.010 ms # H4
```

Now we can simulate a failure in the link `S8->S12` by using this command in configuraion of the router `S8` :

```bash
S8 :/ conf t
s8 (config) :/ interface eth-s12
s8 (config-if) :/ shutdown
```

And after  the link failure `S8->S12` and after convergence of the network, we can see that the traffic is now going through the backup path (`H3->S10->S8->S7->S6->S2->S3->S12->S11->H4`):

```bash
s8:/ show ip bgp ipv6 unicast
     Network              Next Hop                         Metric LocPrf Weight Path
 *>  fc00:2142:1::/64     fe80::a8c1:abff:fe75:f72b                  150      0 65002 65001 i
 *                        fe80::a8c1:abff:fe69:81f9             0             0 65001 i
 *>  fc00:2142:2::/64     fe80::a8c1:abff:fe75:f72b             0    150      0 65002 i
 *                        fe80::a8c1:abff:fe69:81f9                           0 65001 65002 i
 *>  fc00:2142:3::/64     ::                                    0         32768 i
 * i                      fc00:2142:3::9                        0    100      0 i
 *>  fc00:2142:4::/64     fe80::a8c1:abff:fe75:f72b                  150      0 65002 65001 65004 i
 *                        fe80::a8c1:abff:fe69:81f9                           0 65001 65004 i
```

Or via traceroute :
```bash
h3:/ traceroute fc00:2142:4::11 # H3 -> H4
traceroute to fc00:2142:4::15 (fc00:2142:4::15), 30 hops max, 72 byte packets #H3
 1  fc00:2142:3::11 (fc00:2142:3::11)  0.015 ms  0.022 ms  0.035 ms # S10
 2  fc00:2142:3::8 (fc00:2142:3::8)  0.010 ms  0.020 ms  0.011 ms # S8
 3  fc00:2143:5::7 (fc00:2143:5::7)  0.008 ms  0.014 ms  0.065 ms # S7
 4  fc00:2142:2::6 (fc00:2142:2::6)  0.012 ms  0.036 ms  0.018 ms # S6
 5  fc00:2143:1::2 (fc00:2143:1::2)  0.189 ms  0.040 ms  0.012 ms # S2
 6  fc00:2142:1::3 (fc00:2142:1::3)  0.008 ms  0.018 ms  0.014 ms # S3
 7  fc00:2143:4::12 (fc00:2143:4::12)  0.008 ms  0.016 ms  0.010 ms # S12
 8  fc00:2142:4::11 (fc00:2142:4::11)  0.009 ms  0.021 ms  0.017 ms # S11
 9  fc00:2142:4::15 (fc00:2142:4::15)  0.009 ms  0.018 ms  0.011 ms # H4
```

With this configuration, we can see that the traffic is now going through the backup path in case of failure of the main path.
And by restoring the link `S8->S12` we can see that the traffic is now going through the main path.

```bash
S8 :/ conf t
s8 (config) :/ interface eth-s12
s8 (config-if) :/ no shutdown
```

```bash
s8:/ show ip bgp ipv6 unicast
     Network                 Next Hop                          Metric LocPrf Weight Path
 *>  fc00:2142:1::/64        fe80::a8c1:abff:feb6:fa4c                   200      0 65004 65001 i
 *                           fe80::a8c1:abff:fe75:f72b                   150      0 65002 65001 i
 *                           fe80::a8c1:abff:fe69:81f9              0             0 65001 i
 *>  fc00:2142:2::/64        fe80::a8c1:abff:feb6:fa4c                   200      0 65004 65002 i
 *                           fe80::a8c1:abff:fe75:f72b              0    150      0 65002 i
 *                           fe80::a8c1:abff:fe69:81f9                            0 65001 65002 i
 *>  fc00:2142:3::/64        ::                                     0         32768 i
 * i                         fc00:2142:3::9                         0    100      0 i
 *>  fc00:2142:4::/64        fe80::a8c1:abff:feb6:fa4c              0    200      0 65004 i
 *                           fe80::a8c1:abff:fe75:f72b                   150      0 65002 65001 65004 i
 *                           fe80::a8c1:abff:fe69:81f9                            0 65001 65004 i
```
Or via traceroute :
```bash
h3:/ traceroute fc00:2142:4::11 # H3 -> H4
traceroute to fc00:2142:4::15 (fc00:2142:4::15), 30 hops max, 72 byte packets #H3
 1  fc00:2142:3::11 (fc00:2142:3::11)  0.009 ms  0.020 ms  0.029 ms # S10
 2  fc00:2142:3::8 (fc00:2142:3::8)  0.211 ms  0.020 ms  0.011 ms # S8
 3  fc00:2143:8::12 (fc00:2143:8::12)  0.009 ms  0.019 ms  0.377 ms # S12
 4  fc00:2142:4::11 (fc00:2142:4::11)  0.265 ms  0.019 ms  0.012 ms # S11
 5  fc00:2142:4::15 (fc00:2142:4::15)  0.636 ms  0.016 ms  0.010 ms # H4
```

If The link `S7-S12` or `S7-S13` is not under maintenance anymore, we can restore the normal path by removing the local preference on the router `S6`.

### Task 1.4: Load Balancing
We can also use the Local Preference attribute to perform load balancing between multiple paths. For example, we can set the Local Preference value to 200 on one path and 200 on the other path. This will cause the traffic to be distributed between the two paths based on the Local Preference values. But to acheive this we need two paths with the same AS Path length and our BGP session need to be configured to allow multiple paths. Which is a little out of the scope of our project

## Task 2: MED
The Multi-Exit Discriminator (MED) attribute is used to influence the inbound traffic from a neighboring AS. It is used to determine the preferred entry point into an AS when there are multiple entry points. The lower the MED value, the more preferred the route. The default value for MED is 0. It's important to note that MED is propagated to all routers within the neighbor AS. but it's not passed alon to any other AS.

### Task 2.1: Default Configuration
In the default configuration, the MED value is set to 0 on all routers. The paths used by the hosts in the default configuration are shown in the table above in section [Configuration](#configuration).

### Task 2.2: Scenario 1 : Influence inbound traffic
In this scenario we have the router `S7` that have 2 connection in the same AS (`S7->S12` and `S7->S13`) and we want to force the traffic to enter via the router `S13` (the less used router). To do this we can setup the router `S13` with a MED value lower than the MED value of the router `S12`.

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
or via traceroute :
```bash
traceroute to fc00:2142:4::15 (fc00:2142:4::15), 30 hops max, 72 byte packets # H2
 1  fc00:2142:2::11 (fc00:2142:2::11)  0.028 ms  0.019 ms  0.016 ms # S5
 2  fc00:2142:2::7 (fc00:2142:2::7)  0.012 ms  0.028 ms  0.012 ms # S7 
 3  fc00:2143:7::13 (fc00:2143:7::13)  0.010 ms  0.019 ms  0.016 ms # S13
 4  fc00:2142:4::11 (fc00:2142:4::11)  0.026 ms  0.026 ms  0.018 ms # S11
 5  fc00:2142:4::15 (fc00:2142:4::15)  0.017 ms  0.009 ms  0.005 ms # H4
```
We can see that the traffic to `AS65004` is going through the link `S7->S13` with a MED value of 10.

This is a simple example show how we can use the MED attribute to influence the path that traffic takes. For example, here where `S12` has 3 exit points and `S13` has only one exit point same as `S12`, we can use the MED attribute to force the traffic to go through the link `S7->S13` and to relieve traffic congestion on `S12`.

Now we can simulate a link failure in the link `S7->S13` by using this command in configuraion of the router `S7` :

```bash
S7 :/ conf t
s7 (config) :/ interface eth-s13
s7 (config-if) :/ shutdown
```

After convergence of the network, we can see that the traffic is now going through the link `S7->S12` as a backup path:

```bash
s7:/ show ip bgp ipv6 unicast
     Network                   Next Hop                         Metric LocPrf Weight Path
 *>i fc00:2142:1::/64          fc00:2142:2::6                        0    100      0 65001 i
 *                             fe80::a8c1:abff:fe07:3d55                           0 65003 65001 i
 *                             fe80::a8c1:abff:fe93:ea27             50            0 65004 65001 i
 *>  fc00:2142:2::/64          ::                                    0         32768 i
 * i                           fc00:2142:2::5                        0    100      0 i
 * i                           fc00:2142:2::6                        0    100      0 i
 *>  fc00:2142:3::/64          fe80::a8c1:abff:fe07:3d55             0             0 65003 i
 *                             fe80::a8c1:abff:fe93:ea27             50            0 65004 65003 i
 *>  fc00:2142:4::/64          fe80::a8c1:abff:fe93:ea27             50            0 65004 i
 *                             fe80::a8c1:abff:fe07:3d55                           0 65003 65004 i
```
or via traceroute :
```bash
traceroute to fc00:2142:4::15 (fc00:2142:4::15), 30 hops max, 72 byte packets # H2
 1  fc00:2142:2::11 (fc00:2142:2::11)  0.006 ms  0.017 ms  0.033 ms # S5
 2  fc00:2142:2::7 (fc00:2142:2::7)  0.113 ms  0.018 ms  0.012 ms # S7
 3  fc00:2143:6::12 (fc00:2143:6::12)  0.011 ms  0.019 ms  0.011 ms # S12
 4  fc00:2142:4::11 (fc00:2142:4::11)  0.009 ms  0.017 ms  0.011 ms # S11
 5  fc00:2142:4::15 (fc00:2142:4::15)  0.014 ms  0.027 ms  0.011 ms # H4
```

## Task 3 : Communities

BGP communities are **tags** used in BGP routing to group routes. We can then apply specific routing policies to specific tags. The tags follow the `AS:value` format. There are also **well-known communities** that can be invoked by their names:


- `graceful-shutdown`  
- `accept-own`  
- `route-filter-translated-v4`  
- `route-filter-v4`  
- `route-filter-translated-v6`  
- `route-filter-v6`  
- `llgr-stale`  
- `no-llgr`  
- `accept-own-nexthop`  
- `blackhole`  
- `no-export`  
- `no-advertise`  
- `local-AS`  
- `no-peer`

Since these communities are well-known, BGP knows how to handle them and reacts **automatically**. However, in the general `AS:value` case, we must define our own behavior. By default, BGP uses the `0:0` tag, which means _advertise to all peers_.



### A simple community

This first scenario is very simple and can be achieved with a few changes to the original configuration. We are going to set the tag **`no-advertise`** on the routes `S12-S7` and `S13-S7`. Doing so means that `S7` will learn the routes from `S12` and `S13` but **won't advertise them to its peers.** 

First, let's look at the changes in `S12`:


```bash
 address-family ipv6 unicast
  ....
  neighbor fc00:2143:6::7 activate
  neighbor fc00:2143:6::7 route-map PERMIT-ALL in
  neighbor fc00:2143:6::7 route-map COMMU out
....
 exit-address-family
exit
!
...
route-map COMMU permit 10
 set community no-advertise
exit
!
end
```
In `S13`:
```bash
...
  neighbor fc00:2143:7::7 activate
  neighbor fc00:2143:7::7 route-map PERMIT-ALL in
  neighbor fc00:2143:7::7 route-map COMMU out
 exit-address-family
exit
!
...
route-map COMMU permit 10
 set community no-advertise
exit
!
end
```
We can show that `S7` still received the prefix:

```bash
s7/ show ipv6 route fc00:2142:4::
Routing entry for fc00:2142:4::/64
  Known via "bgp", distance 20, metric 0, best
  Last update 00:10:18 ago
  * fe80::a8c1:abff:fe8d:c75, via eth-s12, weight 1
  * fe80::a8c1:abff:fef4:2115, via eth-s13, weight 1

s7/ 
```
But it doesn't advertise it. This can be demonstrated by performing a ping from `H2` to `H4` and observing the path taken:  

```bash
h2:/ traceroute fc00:2142:4::15 #H2 -> H4
traceroute to fc00:2142:4::15 (fc00:2142:4::15), 30 hops max, 72 byte packets
 1  fc00:2142:2::11 (fc00:2142:2::11)  0.011 ms  0.016 ms  0.022 ms #S5
 2  fc00:2142:2::6 (fc00:2142:2::6)  0.007 ms  0.016 ms  0.009 ms #S6
 3  fc00:2143:1::2 (fc00:2143:1::2)  0.009 ms  0.015 ms  0.007 ms #S2
 4  fc00:2142:1::3 (fc00:2142:1::3)  0.006 ms  0.015 ms  0.006 ms #S3
 5  fc00:2143:4::12 (fc00:2143:4::12)  0.010 ms  0.017 ms  0.007 ms #S12
 6  fc00:2142:4::11 (fc00:2142:4::11)  0.013 ms  0.014 ms  0.008 ms #S11
 7  fc00:2142:4::15 (fc00:2142:4::15)  0.006 ms  0.013 ms  0.011 ms #S11 -> H4
```
As shown, the path taken goes through `AS65001` and not via `S7`, as expected.  

### Crash simulation 

As a crash simulation, we can shut down the links `S2-S6` and `S8-S7`. With that, `H2` will no longer be able to ping `H4` since there is no link between them. However, `S7` will still be able to ping `H4`.  It can be shown : 
```bash
h2:/ ping fc00:2142:4::15
PING fc00:2142:4::15 (fc00:2142:4::15) 56 data bytes
From fc00:2142:2::11 icmp_seq=1 Destination unreachable: No route
```
But from `S7` :
```bash
s7/ show ipv6 route fc00:2142:4::
Routing entry for fc00:2142:4::/64
  Known via "bgp", distance 20, metric 0, best
  Last update 00:2:07 ago
  * fe80::a8c1:abff:fe8d:c75, via eth-s12, weight 1
  * fe80::a8c1:abff:fef4:2115, via eth-s13, weight 1
```


## Task 4 : AS Path

**AS Path** is an attribute of BGP that records the sequence of AS numbers a BGP route traverses. It serves two main purposes:

1. **Loop Prevention:** It prevents loops by rejecting routes that already contain the local AS in the AS Path.
2. **Path Selection:** BGP prefers routes with the **shortest AS Path**, making them more attractive.

In our case, we are using **AS Path Prepending**, which means **intentionally making the AS Path longer** to influence routing decisions by making certain routes preferred.

### Simple case

In our simple case, we are going to make the link `S4 -> S11` appear **much longer** by using AS Path Prepending. This will cause BGP to avoid this link when routing traffic. As a result, a ping from `H4 -> H1` **will not go through that link anymore.** Let's look in `S4`:
```bash
...
  neighbor fc00:2143:2::11 activate
  neighbor fc00:2143:2::11 route-map PERMIT-ALL in
  neighbor fc00:2143:2::11 route-map PREPEND-AS out
 exit-address-family
exit
!
...
route-map PREPEND-AS permit 10
 set as-path prepend 65001 65001 
exit
!
```
Let's now examine the traceroute of a ping `H4 -> H1` :
```bash
h4:/ traceroute fc00:2142:1::15 #H4 -> H1
traceroute to fc00:2142:1::15 (fc00:2142:1::15), 30 hops max, 72 byte packets
 1  fc00:2142:4::14 (fc00:2142:4::14)  0.012 ms !N  0.014 ms  0.012 ms # S11
 2  fc00:2142:4::12 (fc00:2142:4::12)  0.008 ms  0.007 ms  0.004 ms #S12
 3  fc00:2143:4::3 (fc00:2143:4::3)  0.008 ms  0.014 ms  0.008 ms #S3
 4  fc00:2142:1::2 (fc00:2142:1::2)  0.008 ms  0.025 ms  0.003 ms #S2
 5  fc00:2142:1::1 (fc00:2142:1::1)  0.014 ms  0.018 ms  0.007 ms #S1
 6  fc00:2142:1::15 (fc00:2142:1::15)  0.014 ms  0.014 ms  0.003 ms #S1 -> H1
```
As expected, our preferred route have changed and it is not going through the `S4 -> S11` link

### Crash simulation

- We've shown that we can influence routing with the **AS Path** attribute. Now, let's imagine that the link `S12-S3` breaks. What would happen? There are now four links with an **AS Path** length equal to 3:  

  - via `S4-S11`: AS65004 - AS65001 - AS65001
  - via `S12-S8` followed by `S8-S3` : AS65004 - AS65003 - AS65001
  - via `S12-S7` followed by `S6-S2` : AS65004 - AS65002 - AS65001
  - via `S13-S7` followed by `S6-S2` : AS65004 - AS65002 - AS65001

Since AS Path is the same, BGP will decide with other attributes. Let's see wich one is choosed : 
```bash
h4:/ traceroute fc00:2142:1::15 # H4 -> H1
traceroute to fc00:2142:1::15 (fc00:2142:1::15), 30 hops max, 72 byte packets
 1  fc00:2142:4::14 (fc00:2142:4::14)  0.009 ms  0.017 ms  0.015 ms #S11
 2  fc00:2142:4::12 (fc00:2142:4::12)  0.006 ms  0.023 ms  0.021 ms #S12
 3  fc00:2143:8::8 (fc00:2143:8::8)  0.008 ms  0.030 ms  0.033 ms #S8
 4  fc00:2143:3::3 (fc00:2143:3::3)  0.008 ms  0.016 ms  0.031 ms #S3
 5  fc00:2142:1::2 (fc00:2142:1::2)  0.016 ms  0.017 ms  0.016 ms #S2
 6  fc00:2142:1::1 (fc00:2142:1::1)  0.007 ms  0.010 ms  0.005 ms #S1
 7  fc00:2142:1::15 (fc00:2142:1::15)  0.004 ms  0.022 ms  0.006 ms #S1 -> H1
 ```
- Now if we shutdown links : `S12-S3` `S12-S7` `S13-S7` `S8-S3`. There will be only two route from H4 to H1 : 
  - via `S4-S11`: AS65004 - AS65001 - AS65001
  - via `S12-S8` followed by `S8-S7` then `S6 - S2` : AS65004 - AS65003 - AS65002 - AS65004

The path with **AS prepending** will be chosen in this case:  

```bash
h4:/ traceroute fc00:2142:1::15 #H4 -> H1
traceroute to fc00:2142:1::15 (fc00:2142:1::15), 30 hops max, 72 byte packets
 1  fc00:2142:4::14 (fc00:2142:4::14)  0.010 ms  0.020 ms  0.005 ms #S11
 2  fc00:2143:2::4 (fc00:2143:2::4)  0.011 ms  0.021 ms  0.005 ms #S4
 3  fc00:2142:1::1 (fc00:2142:1::1)  0.004 ms  0.026 ms  0.015 ms #S1
 4  fc00:2142:1::15 (fc00:2142:1::15)  0.005 ms  0.031 ms  0.011 ms #S1 -> H1
```

# Conclusion
