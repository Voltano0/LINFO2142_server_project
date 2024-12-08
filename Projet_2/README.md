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
To run the project, just run the bash script `run.sh` in the root folder of the project. This script will start the network topology in Docker containers. To access the different routers, you can use the following command where X is the number of the router (1 to 10) :

```bash
sudo docker exec -it clab-igp-s<X> vtysh
```

same for the hosts where X is the number of the host (1 to 3) :

```bash
sudo docker exec -it clab-igp-h<X> bash
```

if you want to stop the network, you can use the following command :

```bash
sudo containerlab destroy
```

## Network Topology
The network topology used consists of 3 AS (AS65001, AS65002, AS65003) and some routers in each AS as shown in the figure below. This topology is inspired by the Belnet network in Belgium in 2010 as we can see here : [Belnet Network](https://topology-zoo.org/maps/Belnet2010.jpg)

![Network Topology](topo.drawio.png)

## Configuration
In each AS, the ISIS routing protocol is used to exchange routing information between routers. iBGP sessions are established between routers in the same AS, and eBGP sessions are established between routers in different AS's. The configuration files for each router can be found in the clab-igp folder.

The paths used by the hosts (H1, H2, H3) in the initial configuration are:

|    	|              H1             	|           H2           	|              H3               |
|:--:	|:---------------------------:	|:----------------------:	|:-----------------------------:|
| H1 	|              /              	| H1->S1->S2->S6->S5->H2 	| H1->S1->S2->S3->S8->S10->H3 	|
| H2 	|    H2->S5->S6->S2->S1->H1   	|            /           	| H2->S5->S7->S9->S10->H3   	|
| H3 	| H3->S10->S8->S3->S4->S1->H1 	| H3->S10->S9->S7->S5-H2 	|              /            	|

## Task 0: ISIS and BGP Configuration
The first task was to configure ISIS and BGP on the routers. The configuration files for each router can be found in the `clab-igp` folder.

We decided not to use a Route Reflector in this project since there are a maximum of 4 routers in each AS. Instead, we opted for a full mesh of iBGP sessions between the routers in the same AS, and eBGP sessions between routers in different ASes. See the topology diagram above.

For reprensatation here is a plot about the mean latency between the different hosts over different packets size :

![Latency Plot](test/plots/latency.png)

We can see that the latency is increasing with the packet size. This is due to the fact that the routers have to fragment the packets when they are too big. This is why the latency is increasing with the packet size. Also the sample of data is not big enough to have a good representation of the latency we should have more data to have a better representation of the latency.

## Task 1: Local Preference
The Local Preference (LP) attribute is used to influence the path that traffic takes within an AS. The higher the LP value, the more preferred the path. The default LP value is 100.

### Task 1.1: Default Configuration
In the default configuration, the LP value is set to 100 on all routers. The paths used by the hosts in the default configuration are shown in the table above in section Configuration.

### Task 1.2: Changing the LP value
In this task, we changed the LP value on the routers to influence the path that traffic takes. We set the LP value to 200 on the routers S2 in AS65001. By setting the LP value to 200 on S2, we encourage traffic to take from AS65001 to AS65002 by passing through `S2->S6`.

We can change the LP value on the routers using the following command in the router's configuration mode:

```text
route-map SET-LOCAL-PREF permit 10
    set local-preference 200
exit

router bgp 65001
    ...
    address-family ipv6 unicast
        neighbor fc00:2143::6 route-map SET-LOCAL-PREF in
    exit
exit
```

We can confirm the new path taken by the hosts by looking at the routing table of the routers or by using the `traceroute` command on the hosts.

```bash
h1: traceroute fc00:2142:2::11 # H1 -> H2
traceroute to fc00:2142:2::11 (fc00:2142:2::11), 30 hops max, 72 byte packets
 1  fc00:2142:1::11 (fc00:2142:1::11)  0.016 ms  0.019 ms  0.012 ms # S1
 2  fc00:2142:1::2 (fc00:2142:1::2)  0.096 ms  0.017 ms  0.012 ms # S2
 3  fc00:2143::6 (fc00:2143::6)  0.011 ms  0.018 ms  0.013 ms # S6
 4  fc00:2142:2::11 (fc00:2142:2::11)  0.016 ms  0.020 ms  0.013 ms # S5->H2

h1: traceroute fc00:2142:3::11 # H1 -> H3
traceroute to fc00:2142:3::11 (fc00:2142:3::11), 30 hops max, 72 byte packets
 1  fc00:2142:1::11 (fc00:2142:1::11)  0.016 ms  0.018 ms  0.018 ms # S1
 2  fc00:2142:1::2 (fc00:2142:1::2)  0.009 ms  0.029 ms  0.012 ms # S2
 3  fc00:2143::6 (fc00:2143::6)  0.009 ms  0.017 ms  0.008 ms # S6
 4  fc00:2142:2::7 (fc00:2142:2::7)  0.006 ms  0.018 ms  0.014 ms # S7
 5  fc00:2143::9 (fc00:2143::9)  0.009 ms  0.017 ms  0.012 ms # S9
 6  fc00:2142:3::11 (fc00:2142:3::11)  0.011 ms  0.019 ms  0.012 ms # S10->H3

# H2 -> H1 : Stay the same
# H2 -> H3 : Stay the same
# H3 -> H1 : Stay the same
# H3 -> H2 : Stay the same

```
**old**:
|    	|              H1             	|           H2           	|              H3               |
|:--:	|:---------------------------:	|:----------------------:	|:-----------------------------:|
| H1 	|              /              	| H1->S1->S2->S6->S5->H2 	| H1->S1->S2->S3->S8->S10->H3 	|
| H2 	|    H2->S5->S6->S2->S1->H1   	|            /           	| H2->S5->S7->S9->S10->H3   	|
| H3 	| H3->S10->S8->S3->S4->S1->H1 	| H3->S10->S9->S7->S5-H2 	|              /            	|

**new**:
|    	|              H1             	|           H2           	|              H3               |
|:--:	|:---------------------------:	|:----------------------:	|:-----------------------------:|
| H1 	|              /              	| H1->S1->S2->S6->S5->H2 	| **H1->S1->S2->S6->S7->S9->S10->H3** 	|
| H2 	|    H2->S5->S6->S2->S1->H1   	|            /           	| H2->S5->S7->S9->S10->H3   	|
| H3 	| H3->S10->S8->S3->S4->S1->H1 	| H3->S10->S9->S7->S5-H2 	|              /            	|


We can see that the path taken by the hosts H1 to H3 has changed. The hosts now take the path `S2->S6` to reach `AS65002` and then `S7->S9->S10` to reach `H3`. 

### Task 1.3: Interpreting the results and possible use cases
The results clearly demonstrate how modifying the Local Preference (LP) attribute influences the path selection within an AS. By increasing the LP value to 200 on S2, traffic from H1 to H2 and H3 now consistently prefers the `S2->S6` path, as opposed to the alternative lower-preference routes.

#### Interpretation of the results

The Local Preference attribute serves as a mechanism to enforce traffic flow policies. Key observations from the results include:

- **Traffic Engineering**: By manipulating the LP attribute, network operators can influence the path that traffic takes within an AS. This can be useful for load balancing, optimizing network performance, and ensuring efficient resource utilization.

- **Path Selection**: The LP attribute allows network operators to define preferred paths for traffic within an AS. By assigning higher LP values to specific routes, operators can ensure that traffic takes the desired path.

- **Policy Enforcement**: The LP attribute can be used to enforce traffic flow policies within an AS. By setting LP values based on specific criteria, operators can control how traffic is routed through the network.

- **Path Redundancy**: LP values can be used to create path redundancy within an AS. By assigning different LP values to multiple paths, operators can ensure that traffic is distributed across redundant links.

- **Traffic backup**: By setting LP values, network operators can define backup paths for traffic in case of link failures or congestion. This ensures that traffic can be rerouted through optimal alternate paths when needed.

## Task 2: MED
The Multi-Exit Discriminator (MED) attribute is a tool used in BGP to influence the choice of entry points into an AS when a neighboring AS has multiple routes to choose from. Unlike Local Preference, which operates within an AS, MED is used between neighboring AS's to signal the preferred route from the perspective of the advertising AS. The lower the MED value, the more preferred the route.

### Topology problem
In the current topology, we were unable to test the effects of the MED attribute. This is due to a lack of multiple equal-length routes between the AS's

- MED is only considered when selecting between routes that have the same AS Path length. If there are no such routes, the MED attribute is not evaluated, and other BGP decision factors take precedence.
- Our topology consists of three AS's (AS65001, AS65002, and AS65003), but it does not include multiple routes between any two AS's with the same AS Path length.

### Rethinking the Topology
To effectively test the MED attribute and observe its behavior in real scenarios, we need to redesign the network topology to include multiple entry points between AS's. 

- **Multiple Connections Between AS's** : Add additional routers between neighboring AS's. For instance between AS65001 and AS65002, introduce two separate connections: one between S2 (AS65001) and S6 (AS65002), and another between S3 (AS65001) and S7 (AS65002).

- **Test Scenarios** : Configure MED values on the routers in AS65001 advertising routes to AS65002. For instance:
    - Advertise a route with a MED of 100 at one peering point (e.g., S2-S6).
    - Advertise the same route with a MED of 200 at the other peering point (e.g., S3-S7).
    - Test how traffic from AS65002 chooses its path into AS65001 based on the MED values.

## Task 3: Communities


## Task 4: AS Path

## Summary of the 4 attributes

| **Attribute**       | **Scope**       | **Purpose**                     | **Preference**                  |
|----------------------|-----------------|----------------------------------|----------------------------------|
| **MED**             | Inter-AS        | Influences the incoming path    | Lower value preferred           |
| **Local Preference**| Intra-AS        | Influences the outgoing path    | Higher value preferred           |
| **Communities**     | Flexible        | Tagging and diverse policies    | No direct preference            |
| **AS Path**         | Inter-AS        | Influences global announcements | Shortest path preferred          |

### Possible Use Cases

MED, Local Preference, AS Path can be used to influence the path. Each one has its own use case

- **Load Balancing**: By adjusting the LP values of different routes, network operators can distribute traffic across multiple paths to balance the load on network links.

- **Optimizing Traffic for Geographic Preferences** : Suppose AS65001 represents a European network and AS65002 represents a network in the USA. To prioritize traffic flow through specific European exit points before reaching the USA (e.g., due to cost agreements or latency optimization), higher LP values can be assigned to European routers such as S2. This ensures traffic destined for the USA exits through Europe.

- **Avoiding Untrusted or Congested Paths**: If certain AS's or paths are reputed unreliable, congested, or potentially compromised (e.g., suspected of traffic interception or surveillance), LP values can be lowered for those routes

- **Policy-Based Routing**: LP values can be used to enforce specific routing policies based on traffic type, source, destination, or other criteria. For example, traffic from specific subnets or services can be directed through specific routers based on LP values.

## Limitations

- **MED** : The MED attribute is only considered when selecting between routes that have the same AS Path length. If there are no such routes, the MED attribute is not evaluated, and other BGP decision factors take precedence.
- **Communities** : 
- **AS Path**:

## Conclusion
