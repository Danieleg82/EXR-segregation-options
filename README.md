# ExpressRoute - Traffic segregation over multiple Expressroute circuits in Hub & Spoke topologies


Hello Azure friends 😊
In this article I’d like to talk about traffic segregation over different Expressroute circuits when leveraging Hub & Spoke topologies, and to show the main available possible designs to reach such result.
Let’s start with the overall requirement:

<img src=pics/Picture1.jpg width="250" height="400" align="center" />
 
In our scenario we have 2 Azure VNETs with different workload types, needing to connect to Onpremise resources via 2 different Expressroute circuits, but still with possibility to connect to each other.
This requirement could rise in order to preserve guaranteed bandwidth SLAs and traffic isolation for applications hosted in different segments.

**The traffic from any resource in VNET1 is expected to use ExpressRoute circuit 1** [*C1 in all our following diagrams*] while **traffic from any resource in VNET2 is expected to use ExpressRoute circuit 2** [*C2*]
VNET1 must be able to connect to VNET2 as well, and we should leverage either standard Hub&Spoke or Virtual WAN connectivity models for future scalability.


### **THE PROBLEM:**

In Azure – today – we don’t have the possibility to perform the routing customizations needed in order to achieve a granular segregation in an easy and scalable way, not if we want to preserve connectivity traffic between VNETs and we want to leverage a single centralized HUB.

In Hub&Spoke scenarios, if we connect a HUB to 2 circuits, and those were connected to the same remote site, Azure would receive the same set of routes over both links, and – similarly – the IP ranges of Azure side would be equally advertised over both links with no possible differentiation.
Onprem-side *Local-Preferences* could be used to prefer a link over the other for certain ranges, but we don’t have a concept of granular Local-Preference to be applied in the Azure HUBs for the return traffic.

**Connection weights** (discussed later below) could help redirecting traffic over a single specific link in case of duplicate routes received, but the preferred link would handle the entire traffic with no real traffic segregation.
*“Route Maps”*-like concepts are today missing in Azure, but will likely be introduced in the near future. 

Fortunately, Azure offers a lot of possible workarounds to achieve the desired result, and we will here explore all the most common available solutions, from the most ordinary ones to the most complex.
Some of these solutions are based on double-HUB topologies, others try to achieve the same results leveraging a single HUB.

### **TABLE OF CONTENT:**

   [DOUBLE HUB SOLUTIONS](#double-hub-solutions)
   - [SCENARIOS LEVERAGING VNET PEERING FOR INTER-SPOKE COMMUNICATIONS](#scenarios-leveraging-vnet-peering-for-inter-spoke-communications)
     - [1.A – DOUBLE HUB AND DIRECT PEERING BETWEEN SPOKEs](#scenario-1a--double-hub-and-direct-peering-between-spokes)
     - [1.B – DOUBLE HUB AND DIRECT PEERING BETWEEN HUBs](#scenario-1b--double-hub-and-direct-peering-between-hubs)
     - [1.C – DOUBLE HUB AND DIRECT PEERING BETWEEN HUBs + AZURE ROUTE SERVER](#scenario-1c--double-hub-and-direct-peering-between-hubs--azure-route-server)
     - [1.D – DOUBLE vHUB (vWAN)](#scenario-1d--double-vhub-vwan)
   - [SCENARIOS LEVERAGING EXPRESSROUTE FOR INTER-SPOKE COMMUNICATIONS](#scenarios-leveraging-expressroute-for-inter-spoke-communications)
     - [2.A – DOUBLE HUB AND EXPRESSROUTE HAIRPINNING WITH “BOW-TIE” CONNECTIONS](#scenario-2a--double-hub-and-expressroute-hairpinning-with-bow-tie-connections)
     - [2.B – DOUBLE HUB AND EXPRESSROUTE HAIRPINNING WITH SINGLE SIDE CIRCUIT’S REDUNDANCY](#scenario-2b--double-hub-and-expressroute-hairpinning-with-single-side-circuits-redundancy)

   [SINGLE HUB SOLUTIONS](#single-hub-solutions)
   - [3.A: SINGLE HUB & SEPARATE BGP RANGES ADVERTISEMENT](#scenario-3a-single-hub--separate-bgp-ranges-advertisement)
   - [3.B: SINGLE HUB & ONPREM ROUTES’ CUSTOMIZATION VIA BGP PATH PREPENDING](#scenario-3b-single-hub--onprem-routes-customization-via-bgp-path-prepending)

   [COMPARISON TABLE](#comparison-table)
   
   [CONCLUSIONS](#conclusions)
   
   [CONFIGURATION OF EXPRESSROUTE WEIGHTs](#configuration-of-expressroute-weights)


# **DOUBLE HUB SOLUTIONS**

## **SCENARIOS LEVERAGING VNET PEERING FOR INTER-SPOKE COMMUNICATIONS**

### **SCENARIO 1.A – DOUBLE HUB AND DIRECT PEERING BETWEEN SPOKEs**

<img src=pics/1A.jpg />

In this topology we have VNET1 and VNET2 connected to 2 isolated HUBs (standard HUB VNETs, no vWAN).
Every HUB is connected to a single circuit (C1 or C2)
The 2 spoke VNETs are then linked **via direct VNET Peering.**

In this design, C1 receives only the routes of VNET1 from Azure, while C2 receives only the routes of VNET2.
Onprem can advertise the same set of IP ranges over both links.
Communication between VNET1 and VNET2 is granted via direct VNET-peering between them.


**TOPOLOGY PROs:**
-	No need for any routing manipulation anywhere, neither on Azure nor Onprem side.

**TOPOLOGY CONs:**
-	Costs: 2 HUBs (with relevant Expressroute Gateways) + VNET peering
- Management of the spokes’ direct VNET peerings (imagine you had to connect not 2, but “N” spokes…)
-	Not fault-tolerant: if one circuit is down, traffic can’t move to the other circuit

*Note about Azure ranges’ re-advertisement: in this scenario, the routing is not influenced by re-advertisement of Azure VNETs’ range learnt from one circuit over the second circuit, since VNET peering would have priority as nexthop*

**SUMMARY OF LINK ROUTES:**

| **Link Name**  | **Routes advertised by Azure** | **Routes learnt by Azure** | 
| ------------- | ------------- | ------------- |
| L1  | VNET1 + HUB1  | Onprem (+ VNET2 + HUB2 in case of re-advertisement)  |
| L2  | VNET2 + HUB2  | Onprem (+ VNET1 + HUB1 in case of re-advertisement)  |
| L3  | VNET1 + HUB1  | Onprem (+ VNET2 + HUB2 in case of re-advertisement)  |
| L4  | VNET2 + HUB2  | Onprem (+ VNET1 + HUB1 in case of re-advertisement)  |

### SCENARIO 1.B – DOUBLE HUB AND DIRECT PEERING BETWEEN HUBs
 
<img src=pics/1B.jpg />

This scenario leverages 2 central **standard HUB VNETs** (no vWAN).
VNET1 is peered with HUB1, VNET2 with HUB2
A VNET peering with no transit option connects the 2 HUBs

**Virtual appliances (NVAs)** deployed in the HUBs, with appropriate configuration of **Route Tables (UDR)** configured on their subnets, allow the communications between VNET1 and VNET2, and any other spoke connected to the HUBs.

*Route tables* will be needed in the spoke VNETs as well, in order to redirect traffic for remote spokes to the centralized appliances.
Here as well, C1 receives only the routes of VNET1 from Azure, while C2 receives only the routes of VNET2.
This design allows to interconnect spokes without leveraging a high number of VNET peerings, hence it reduces topology complexity, but it doesn’t offer fault-tolerance in case of failure of one ExpressROute circuit.

**TOPOLOGY PROs:**
-	No need for any routing manipulation anywhere, neither on Azure nor Onprem side.
-	No peering between spokes, just a central peering at HUB level

**TOPOLOGY CONs:**
-	Costs: 2 HUBs (with relevant Expressroute Gateways) + central HUB VNET peering + NVAs
-	Deployment / configuration complexity (Route tables / NVA … )
-	Potential performance concerns due to possible NVA-related bottlenecks
-	Not fault-tolerant: if one circuit is down, traffic can’t move to the other circuit

*Note about Azure ranges’ re-advertisement: in this scenario, the routing is not influenced by re-advertisement of Azure VNETs’ range learnt from one circuit over the second circuit, since UDRs configured at NVA subnet layer would have priority as nexthop, if properly configured with specific spokes’ IP ranges.*

**SUMMARY OF LINK ROUTES:**

| **Link Name**  | **Routes advertised by Azure** | **Routes learnt by Azure** | 
| ------------- | ------------- | ------------- |
| L1  | VNET1 + HUB1  | Onprem (+ VNET2 + HUB2 in case of re-advertisement)  |
| L2  | VNET2 + HUB2  | Onprem (+ VNET1 + HUB1 in case of re-advertisement)  |
| L3  | VNET1 + HUB1  | Onprem (+ VNET2 + HUB2 in case of re-advertisement)  |
| L4  | VNET2 + HUB2  | Onprem (+ VNET1 + HUB1 in case of re-advertisement)  |



### SCENARIO 1.C – DOUBLE HUB AND DIRECT PEERING BETWEEN HUBs + AZURE ROUTE SERVER
 
<img src=pics/1C.jpg />

This third variant of double-HUB scenario is probably the most complex to be implemented, but at the same time one of the most interesting ones.

It leverages **standard HUB VNETs + Azure Route Servers & NVAs** and it’s basically the fully customer-managed version of the solution we’ll see later on (Scenario 1.D) based on vWAN,

This solution should be preferred to the 1.D  **exclusively** in case of specific blockers regarding the adoption of vWAN/vHUBs, since this basically offers the same topology but with much higher implementation complexity.
In this double managed HUB solution, we configure **Azure Route Server (ARS)** with *Branch2Branch* option enabled, and BGP-peer it with a virtual appliance in our HUB (we do this on both sides).

We create an iBGP peering between the appliances in the different HUBs, so that the routes learnt from one HUB environment are propagated to the second one.
There are basically 2 ways of setting this up:

*1)Without any tunnelling technology (VXLAN / IPSEC) between the appliances and UDRs configured at NVA subnet level*

*2)With VXLAN/IPSEC and no need of UDRs at NVA subnet level*

The routes learnt by first NVA (in HUB1) – including routes learnt from Expressroute C1 branch – will be advertised to the second NVA, and vice versa.
NVAs must be programmed to perform AS-path override in order to remove ASN 65515 from the path.

ARS will program the routes learnt from the NVA deployed in its own HUB: 2 sets of Onprem routes are expected to be seen by ARS:

-	The first will come from its NVA 
-	The second will come from the ExpressRoute gateway of the HUB VNET 

Since we’re leveraging Azure Route Server, routes learnt over ExpressRoute have always precedence over routes learnt from other sources*

This scenario offers implicit failover in case of outage of one circuit.

No UDRs will be here needed in the spoke VNETs, independently by the usage of a VXLAN/IPSEC tunneling between NVAs
Note that – in this scenario – NVAs handle both the inter-spoke and potentially as well the SpokeOnprem traffic in case of link failure.

Regarding return traffic (Onprem to Azure): Both circuits will receive routes of both VNET1 and VNET2, so *Local Preference* should be used at Onprem side to redirect traffic over the appropriate links.

This article covers a similar setup and provides deeper details about the possible NVA routing configurations 
https://github.com/jocortems/azurehybridnetworking/tree/main/ExpressRoute-Transit-with-Azure-RouteServer#4-multi-region--multi-nic-nvas-with-route-tables 

**TOPOLOGY PROs:** 
-	No peering between spokes, just a central peering at HUB level
-	Fault tolerance in case of failure of one ExpressRoute link

**TOPOLOGY CONs:**
-	Costs: 2 HUBs (with relevant Expressroute Gateways) + central HUB VNET peering + NVAs + ARS
-	Deployment / Configuration complexity (configuration of NVA BGP settings / VXLAN / IPSEC)
-	Potential performance concerns due to possible NVA-related bottlenecks
-	To avoid inter-VNET traffic via circuits, Azure routes should not be re-advertised by provider-edge over the 2 circuits.*

* *Note about Azure ranges’ re-advertisement: in this scenario, the routing can be influenced by re-advertisement of Azure VNETs’ range learnt from one circuit over the second circuit, depending on the AS path length of routes received from ExpressRoute gateway and from NVAs
Since routes learnt from ExpressRoute have precedence, traffic between VNETs would take ExpressRoute links in case of routes’ readvertisement, so in this scenario the re-advertisement should be avoided.*

**SUMMARY OF LINK ROUTES:**

| **Link Name**  | **Routes advertised by Azure** | **Routes learnt by Azure** | 
| ------------- | ------------- | ------------- |
| L1  | VNET1 + HUB1 + VNET2 + HUB2  | Onprem (+ VNET2 + HUB2 in case of re-advertisement)  |
| L2  | VNET1 + HUB1 + VNET2 + HUB2  | Onprem (+ VNET1 + HUB1 in case of re-advertisement)  |
| L3  | VNET1 + HUB1 + VNET2 + HUB2  | Onprem (+ VNET2 + HUB2 in case of re-advertisement)  |
| L4  | VNET1 + HUB1 + VNET2 + HUB2  | Onprem (+ VNET1 + HUB1 in case of re-advertisement)  |



### SCENARIO 1.D – DOUBLE vHUB (vWAN) 

 <img src=pics/1D.jpg />

This scenario is conceptually similar to the 1.C, but it’s based on vWAN HUBs (vHUBs) hence without the need of NVAs deployments / VNET peering management / HUB routing configurations

The scenario is built with 2 separate vHUBs deployed in the context of the same vWAN (vHUBs could be deployed in same region, or different, depending on the specific case).

VNET1 will connect with vHUB1, which will connect to C1

VNET2 will connect with vHUB2, which will connect to C2

The 2 vHUBs are implicitly connected via MS backbone.

vHUB1 will receive Onprem routes both from its direct ExpressRoute gateway connection, and as well as inter-hub routes (from vHUB2).

The default vWAN behavior is to prefer routes learnt from ExpressRoute over routes learnt from other HUBs: this is what will actually do the segregation trick for us and will as well provide high availability to the solution.

VNET1 via vHUB1 will always use C1 to reach Onprem, and VNET2 via vHUB2 will always use C2 for the same.

Provider-edge side will have to be configured with Local Preferences for handling the return traffic over the appropriate link, since every link is receiving routes for both VNETs/HUBs.

In case of failure of one of the ExpressRoute links, the VNETs will leverage the inter-HUB connectivity to reach Onprem, so we have a highly available solution.

Traffic between VNET1 and VNET2 will leverage vWAN backbone as for standard vWAN behavior, unless route of remote spokes were re-advertised over the expressroute circuits*, in which case default behavior would be to prefer ExpressRoute again over inter-hub routes. 

This behavior can be tuned in vWAN by leveraging the feature called HUB routing preferences, which is today a Preview feature (https://docs.microsoft.com/en-us/azure/virtual-wan/about-virtual-hub-routing-preference )
Setting HRP (Hub routing Preference) to “AS-Path” mode would make vHUB always select best routes depending on AS-Path-length as first decisional parameter.


**TOPOLOGY PROs:** 
-	All the benefits of vWAN’s implicit transit routing and failover capabilities

**TOPOLOGY CONs:** 
-	Costs: 2 vHUBs with ExpressRoute gateways

*Note about Azure ranges’ re-advertisement: as per the above statement, in this scenario the routing can be influenced by re-advertisement of Azure VNETs’ range learnt from one circuit over the second circuit.*

**SUMMARY OF LINK ROUTES:**

| **Link Name**  | **Routes advertised by Azure** | **Routes learnt by Azure** | 
| ------------- | ------------- | ------------- |
| L1  | VNET1 + HUB1 + VNET2 + HUB2  | Onprem (+ VNET1 + HUB1 + VNET2 + HUB2 in case of re-advertisement)  |
| L2  | VNET1 + HUB1 + VNET2 + HUB2  | Onprem (+ VNET1 + HUB1 + VNET2 + HUB2 in case of re-advertisement)|
| L3  | VNET1 + HUB1 + VNET2 + HUB2  | Onprem (+ VNET2 + HUB2 in case of re-advertisement)  |
| L4  | VNET1 + HUB1 + VNET2 + HUB2  | Onprem (+ VNET1 + HUB1 in case of re-advertisement)  |


## SCENARIOS LEVERAGING EXPRESSROUTE FOR INTER-SPOKE COMMUNICATIONS

This class of scenarios makes use of the circuit connections to allow communications between spokes VNETs.
Note that this has implicit repercussions on the circuit’s available bandwidth, and may not be a desirable solution if you want to preserve your circuit’s capacity for pure AzureOnprem traffic, that is one of our prerequisites, but could be an acceptable compromise in case of negligible amount of traffic foreseen for inter-VNET scenarios. 

### SCENARIO 2.A – DOUBLE HUB AND EXPRESSROUTE HAIRPINNING WITH “BOW-TIE” CONNECTIONS

 <img src=pics/2A.jpg />

In this scenario we use 2 standard VNET HUBs (no vWAN), both connected to both ExpressRoute circuits, in a so-called “bow-tie” configuration.
The connection between HUB1 and C1 will be set with higher weight* than the connection between HUB1 and C2.
[*see below for weights’ configurations*]

Similar approach – symmetrically – will be adopted for the HUB2
HUB1 will receive the same set of routes (Onprem ranges + VNET/HUB2 ranges) from the 2 connections. 
The weight will make sure that C1 will always be used for any connection between VNET1 and Onprem.
C1 will also be used for traffic between VNET1 and VNET2 in normal conditions.
In case of failure of C1, C2 will be used by VNEt1 for every connection with Onprem or VNET2
Local preferences set on Provider side will grant onprem traffic to use the appropriate link to return in Azure.

**TOPOLOGY PROs:** 
-	No VNET-peering needed
-	Fault tolerance in case of failure of one ExpressRoute link

**TOPOLOGY CONs:** 
-	Costs: 2 HUBs (with relevant Expressroute Gateways) 
-	The inter-VNET traffic will consume circuit’s bandwidth, with relevant possible performance issues.

*Note about Azure ranges’ re-advertisement: in this scenario, the routing is not supposed to be influenced by re-advertisement of Azure VNETs’ range learnt from one circuit over the second circuit, since such ranges are anyway automatically advertised over both circuits due to the bow-tie configuration*

**SUMMARY OF LINK ROUTES:**

| **Link Name**  | **Routes advertised by Azure** | **Routes learnt by Azure** | 
| ------------- | ------------- | ------------- |
| L1  | VNET1 + HUB1 + VNET2 + HUB2  | Onprem (+ VNET1 + HUB1 + VNET2 + HUB2 in case of re-advertisement)  |
| L2  | VNET1 + HUB1 + VNET2 + HUB2  | Onprem (+ VNET1 + HUB1 + VNET2 + HUB2 in case of re-advertisement)|
| L3  | VNET1 + HUB1 | Onprem + VNET2 + HUB2   |
| L4  | VNET2 + HUB2 | Onprem + VNET1 + HUB1   |
| L5  | VNET1 + HUB1 | Onprem + VNET2 + HUB2   |
| L6  | VNET2 + HUB2 | Onprem + VNET1 + HUB1  |


### SCENARIO 2.B – DOUBLE HUB AND EXPRESSROUTE HAIRPINNING WITH SINGLE SIDE CIRCUIT’S REDUNDANCY

<img src=pics/2B.jpg />
 
This is just a small variant of 2.A, which could be considered in case we preferred the preservation of C2 capacity over symmetrical fault-tolerance.

In this scenario, the HUB of VNET1 is connected to C1 only, while the HUB of VNETs is connected to both circuits.

While the 2.A was offering the possibility to redirect the VNET1_to_VNET2 traffic over C2 in case of failure of C1 (with all the relevant consequences in terms of impact on C2’s bandwidth), this 2.B is eliminating that possibility in case we wanted to protect C2’s capacity.

Under normal conditions, the VNET1_to_VNET2 traffic is here handled through C1 only, unless provider edge side proceeds with routes’ re-advertisements between circuits*, which has here to be avoided to meet the objective of preserving C2’s capacity.

In case of failure of C1, VNET1 will be isolated (hence here talking about “partial” fault-tolerance), but C2 capacity is totally preserved.


**TOPOLOGY PROs:** 
-	No VNET peering management needed
-	Partial Fault tolerance in case of failure of one ExpressRoute circuit

**TOPOLOGY CONs:**
-	Costs: 2 HUBs (with relevant Expressroute Gateways) 
-	The inter-VNET traffic will consume circuit’s bandwidth, with relevant possible performance issues.
-	Partial Fault tolerance in case of failure of one ExpressRoute circuit

*Note about Azure ranges’ re-advertisement: in this scenario, the routing is for sure influenced by re-advertisement of Azure VNETs’ range learnt from one circuit over the second circuit.
In case of re-advertisement, and due to the higher connection’s weight set on the link between HUB2 and C2, all the inter-VNET communications will make use of C2*

**SUMMARY OF LINK ROUTES:**

| **Link Name**  | **Routes advertised by Azure** | **Routes learnt by Azure** | 
| ------------- | ------------- | ------------- |
| L1  | VNET1 + HUB1 + VNET2 + HUB2  | Onprem (+ VNET1 + HUB1 + VNET2 + HUB2 in case of re-advertisement)  |
| L2  | VNET2 + HUB2 | Onprem (+ VNET1 + HUB1 + VNET2 + HUB2 in case of re-advertisement)|
| L3  | VNET1 + HUB1 | Onprem + VNET2 + HUB2   |
| L4  | VNET2 + HUB2 | Onprem + VNET1 + HUB1   |
| L5  | VNET2 + HUB2 | Onprem + VNET1 + HUB1   |



# SINGLE HUB SOLUTIONS

Scenarios based on a single central HUB (or vHUB) are implicitly preferrable to double-HUB scenarios, because of the reduced complexity and optimized design, but they – today – do not allow to reach the traffic segregation effect without deep manipulation of advertised routes performed by the Onprem / Provider Edge side.
Future features like common routers’ Route-maps concept will facilitate these scenarios.
Let’s explore the main possibilities available today.

### SCENARIO 3.A: SINGLE HUB & SEPARATE BGP RANGES ADVERTISEMENT
 
 <img src=pics/3A.jpg />

In this scenario, we introduce a single central HUB to terminate both the circuits.

Note that this solution can be either a standard HUB VNET, or a vHUB (vWAN), with the only difference that in the case of managed HUB the connectivity between VNET1 and VNET2 will require the deployment of a virtual appliance (NVA) in the HUB + Route Tables in the spoke VNETs to bridge their connectivity.

The mechanism used here to achieve the traffic segregation is the advertisement of different BGP routes over the 2 different circuits + local preferences.
C1 will advertise to Azure exclusively the IP ranges of Onprem workloads needing to connect to VNET1 deployments.

Similarly, C2 will advertise to Azure exclusively the IP ranges of Onprem workloads needing to connect to VNET2 deployments.

For the upstream traffic (from Onprem To Azure) customer will have to implement BGP Local Preferences, since Azure will send 2 sets of identical routes over the 2 circuits.

In alternative (but only when using managed HUB VNET approach) customer will soon be able to leverage custom BGP communities to tag routes of VNET1 and VNET2 in a different way, thus facilitating the configuration of routing from Onprem perspective.

This feature is – today - still in Preview phase. (https://docs.microsoft.com/en-us/azure/expressroute/how-to-configure-custom-bgp-communities) 

This setup has no fault-tolerance, since the IP ranges advertised over the 2 circuits are different.

This scenario is also very unlikely to be considerable, since it’s realistically very difficult to have in a real-life case a so well-defined segregation of Onprem IP ranges correlated with VNET1/VNET2 endpoints.

**TOPOLOGY PROs:**
-	Leveraging a single HUB, with reduced costs

**TOPOLOGY CONs:**
-	It needs huge routing customization by Onprem / Provider edge side
-	Not fault-tolerant in case of ExpressRoute link failure
-	In case of standard HUB VNET utilized, it needs NVA/Azure Firewall to bridge inter-VNET traffic


**SUMMARY OF LINK ROUTES:**

| **Link Name**  | **Routes advertised by Azure** | **Routes learnt by Azure** | 
| ------------- | ------------- | ------------- |
| L1  | VNET1 + VNET2 + HUB | IP Range 1 |
| L2  | VNET1 + VNET2 + HUB | IP Range 2 |
| L3  | VNET1 + VNET2 + HUB | IP Range 1 |
| L4  | VNET1 + VNET2 + HUB | IP Range 2 |




### SCENARIO 3.B: SINGLE HUB & ONPREM ROUTES’ CUSTOMIZATION VIA BGP PATH PREPENDING 
As alternative to 3.A, we could advertise same ranges over the 2 links but applying AS Path Prepending technique for the ranges which are considered less-preferred over a specific link  in this scenario we would introduce fault-tolerance over the 2 different links
 
 <img src=pics/3B.jpg />

In our example here, IP Range1 will be the set of IP ranges representing common destinations for VNET1’s workload, and IP Range2 – similarly – will represent the set of endpoints for VNET2’s workload.

This solution can be either a standard HUB VNET, or a vHUB (vWAN), with the only difference that in the case of managed HUB the connectivity between VNET1 and VNET2 will require the deployment of a virtual appliance (NVA) in the HUB + Route Tables in the spoke VNETs to bridge their connectivity.

Here as well, the scenario is unlikely to be manageable, since it’s realistically difficult to have in a real-life case a so well-defined segregation of Onprem IP ranges correlated with VNET1/VNET2 endpoints


**TOPOLOGY PROs:**
-	Leveraging a single HUB, with reduced costs
-	Fault-tolerance in case of ExpressRoute link failure

**TOPOLOGY CONs:** 
-	It needs huge routing customization from Onprem / Provider edge side
-	In case of standard HUB VNET utilized, it needs NVA/Azure Firewall to bridge inter-VNET traffic


**SUMMARY OF LINK ROUTES:**

| **Link Name**  | **Routes advertised by Azure** | **Routes learnt by Azure** | 
| ------------- | ------------- | ------------- |
| L1  | VNET1 + VNET2 + HUB | IP Range 1 + AS-prepended Range 2 |
| L2  | VNET1 + VNET2 + HUB | IP Range 2 + AS-prepended Range 1 |
| L3  | VNET1 + VNET2 + HUB | IP Range 1 + AS-prepended Range 2 |
| L4  | VNET1 + VNET2 + HUB | IP Range 2 + AS-prepended Range 1 |


# **COMPARISON TABLE**

| **Solution**  | **Nr of HUBs** | **Requires routing config at ExpressRoute circuit side?** | **Requires routing config at provider/Onprem side?** | **Requires routing config inside Azure HUBs (UDRs etc..)?** | **Support failover in case of circuit failure?** | **General considerations** | **Recommended for...** | 
| ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- | ------------- |
| **[1.A](#scenario-1a--double-hub-and-direct-peering-between-spokes)** | 2 | No | No | No | No | 2 HUBs (considerable costs) + maintenance of VNET-peerings between spokes | Small topologies, with few VNET peerings to be managed between spokes. Not ideal for scenarios with more than 2 HUBs/Circuits. |
| **[1.B](#scenario-1b--double-hub-and-direct-peering-between-hubs)** | 2 | No | No | Yes | No | 2 HUBs (considerable costs) + proper NVA dimensioning | Topologies requiring future Azure workload’s expansion, when routing configuration at PE/Onprem side is not feasible. Not ideal for scenarios with more than 2 HUBs/Circuits. |
| **[1.C](#scenario-1c--double-hub-and-direct-peering-between-hubs--azure-route-server)** | 2 | No | Yes (LocalPref) | Yes/No (Depending on the implementation of tunneling technologies between the NVAs) | Yes | 2 HUBs (considerable costs) + proper NVA dimensioning + complex configuration | Topologies requiring future Azure workload’s expansion, requiring symmetrical failover capabilities in case of circuits’ outages. Not ideal for scenarios with more than 2 HUBs/Circuits. |
| **[1.D](#scenario-1d--double-vhub-vwan)** | 2 | No | Yes (LocalPref) | No | Yes | 2 vWAN HUBs with relevant costs.Probably the best solution in terms of ease of configuration. Scalable over > 2 HUBs/Circuits | Any scenario needing scalability and failover options, and where there’s no specific blocker for vWAN’s adoption. |
| **[2.A](#scenario-2a--double-hub-and-expressroute-hairpinning-with-bow-tie-connections)** | 2 | Yes (Weights) | Yes (LocalPref) | No | Yes | 2 HUBs (considerable costs). Cross-VNET traffic via both Expressroute circuits( impact on circuit’s bandwidth / performances). No need of VNET-peering maintenance | Topologies requiring future Azure workload’s expansion, requiring symmetrical failover capabilities in case of circuits’ outages. No VNET-peering needed (= ease of configuration and cost-saving), but inter-VNET traffic impacting on circuits’ capacity with possible performance impact.|
| **[2.B](#scenario-2b--double-hub-and-expressroute-hairpinning-with-single-side-circuits-redundancy)** | 2 | Yes (Weights) | Yes (LocalPref) | No | Yes (Partial / asymmetric) | 2 HUBs (considerable costs). Cross-VNET traffic via single Expressroute circuit( impact on circuit’s bandwidth / performances). No need of VNET-peering maintenance | Topologies requiring future Azure workload’s expansion, requiring asymmetrical failover capabilities in case of circuits’ outages. No VNET-peering needed (= ease of configuration and cost-saving), but inter-VNET traffic impacting on circuits’ capacity with possible performance impact. |
| **[3.A](#scenario-3a-single-hub--separate-bgp-ranges-advertisement)** | 1 | No | Yes (LocalPref + granular split onprem ranges) | No | No | Single HUB solution, with all relevant benefits on costs. No impact on circuit’s capacity for inter-VNET traffic. Requires heavy configuration of routing from provider/onprem side. Not fault-tolerant in case of circuit’s failure | Any scenario where customer has the possibility to perform fine-tuned routing manipulations from provider/onprem routers and doesn’t require/want failover capabilities. Azure workloads / amount of spokes can easily scale. |
| **[3.B](#scenario-3b-single-hub--onprem-routes-customization-via-bgp-path-prepending)** | 1 | No | Yes (LocalPref + granular split onprem ranges & path prepending) | No | Yes | Single HUB solution, with all relevant benefits on costs. No impact on circuit’s capacity for inter-VNET traffic. Requires heavy configuration of routing from provider/onprem side. Fault tolerant in case of circuit’s failure | Any scenario where customer has the possibility to perform fine-tuned routing manipulations from provider/onprem routers and  requires/wants failover capabilities. Azure workloads / amount of spokes can easily scale. |

# CONCLUSIONS:

As we’ve seen, there are several possible paths to achieve traffic segregation over different ExpressRoute links, while still leveraging centralized HUB/s solutions.
Some of these paths require high deployment & management complexity.
Others require high routing-configuration complexity. 
Finally, some of them may be simpler to be deployed and maintained, but more expensive.
Due to these characteristics, choosing the best topology for your environment is something that must be evaluated case by case depending on the specific needs and constraints (no “always right”/”always wrong” options here 😊 ) 
But it’s important to think that – despite lacking today real RouteMaps-like features – we have a lot of possible designs which can be used to achieve the desired goal 





# CONFIGURATION OF EXPRESSROUTE WEIGHTs
https://docs.microsoft.com/en-us/azure/expressroute/expressroute-optimize-routing#suboptimal-routing-from-microsoft-to-customer
https://docs.microsoft.com/en-us/azure/expressroute/expressroute-howto-linkvnet-arm#modify-a-virtual-network-connection 

Editing Connection weight in managed VNET scenario:

<img src=pics/weight1.jpg />
 
Editing Connection weight in vWAN scenario:
 
<img src=pics/weight2.jpg />

<img src=pics/weight3.jpg />

<img src=pics/weight4.jpg />

 










