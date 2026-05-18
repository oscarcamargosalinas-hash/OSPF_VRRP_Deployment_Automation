**Multi-Area OSPF & A-A VRRP Deployment Automation**

<img width="2449" height="1220" alt="Image" src="https://github.com/user-attachments/assets/be313ff5-ee5f-4feb-8051-16d203723e9b" />

Overall Architecture

The OSPF Cisco infrastructure deployed using Ansible consists of a 2-area network topology (Backbone Area 0 and Area 1).

- Backbone Area 0: Consists of 2 routers working in High Availability (HA) and one router acting as an Area Border Router (ABR) for Area 1.
- LAN Side Redundancy: Routers within the VRRP groups provide failover capabilities for the LAN side. VRRP VLANs provide gateway redundancy for the LAN side only.
- OSPF Adjacencies: R1 and R2 build their OSPF neighbor relationship over dedicated routed links on 172.16.2.0/24 and direct paths towards R3.

LAN & Routing Optimizations
L2 configurations: In the LAN side, I have opted to using PortFast on the trunks going from SW2 to the routers. This helped in increasing the failover speed when routers were unavailable.

OSPF Convergence: On the OSPF side, I have lowered the OSPF dead interval from 40 to 4 seconds to drastically speed up failure detection during my failover tests.

Code Definition
The following configuration is designed to deploy OSPF configurations across 3 routers. Two of these routers utilize VRRP with a design that provides HA and load sharing across VLANs, similar to an Active-Active (A-A) deployment. In this topology, R1 is the master for VLAN 5 and R2 is the master for VLAN 10.

I have chosen Ansible because of its scaling capability, idempotency, and clean error handling.

Core Automation Components
1. Per-Host Variable Definitions (host_vars): Used by the deployments.yml playbook to gather interface details, VRRP settings, and OSPF definitions for each router. The host variables use general naming to better query values, which helps us scale seamlessly when more nodes need to be included.

2. Deployment Playbook (deployments.yml): This Ansible definition consists of different tasks such as configure VLANs, interfaces, sub-interfaces, and OSPF, while others store the final running config. For each task, I have opted for looping through interfaces (such as VRRP) to make the output match the overall configuration layout.

3. Configuration File (ansible.cfg): Contains essential operational information about inventory tracking, execution timeouts, and host key checking.

4. Inventory File (hosts.ini): Includes explicit definitions for SW2, R1, R2, R3, along with the required authentication settings for each router.

By keeping the configuration fully in code, it allows us to easily track changes and version updates to VRRP, OSPF, and physical interfaces as our infrastructure grows in size.

State Tracking Note: The framework includes an Output section where a Nornir/Netmiko collector gathers all live information from the 3 routers and organizes them into structured JSON for easier management. This helps track our existing configuration of OSPF, BGP, Routes, and Interfaces. This execution block also includes a configuration difference task used post deployment during validation tests.

Post-Deployment Validation
To ensure deployment success, I ran a packet capture and state verification testing 2 core variables:

1. Neighbor Establishment & Control Plane Flow
I used the Ansible playbook to pass show ip ospf neighbor to confirm operational adjacency status, while also running packet captures on the neighboring links.

The idea with the second option was to check for raw OSPF messages such as Hello packets, DB Descriptions, LS Updates, and LS ACK. This verified directly that the live communication flow was matching OSPF V2 communication standards.

2. Failover, Convergence, & Route Updates
To test stability, I simulated a failover and convergence event by failing R1, alongside testing a route update scenario for when R3 experiences a link down event.

Data Plane Recovery: During a continuous ICMP request/reply stream, traffic recovered after the VRRP/OSPF reconvergence event. During failover, the available router immediately takes the shared virtual MAC address, and SW2 tags the traffic according to the static VLAN on the access port.

Drop Window & Latency Metrics: Pings recovered quickly as seen in the capture below, the path cutover results in a minor gap, with the first packet returning through the backup path before stabilizing back down to normal flow.

Route Updates Analysis: For the route updates tracking example, I opted to use a custom parsing collector that I created, chosen mainly for its direct compatibility with Cisco IOSv models and clean parsing capabilities: Available here at Cisco Baseline Checker. This allowed me to correlate easily exactly how the R3 interface went down via a clean JSON state difference output.

Capabilities & Limitations
Scalability Properties
This code can be expanded to more routers since the base deployment.yml doesn't need to change and can persist regardless.

This same configuration can stretch to cover other switch definitions (such as additional VLAN settings or PortFast enablement) or other routers needing OSPF configurations.

Identified Limitations
General Module Selection: One limitation that I identified includes the heavy use of the module cisco.ios.ios_config. Even though it is general and can cover many of our administrative needs, it can fall short compared to more specific modules that handle structured data natively. Also, as software versions are updated, this syntax implementation may not work perfectly across all Cisco versions.

Multi Vendor usage: Using this solution as a "one size fits all" for our different vendors may require some planning. We will need to carefully coordinate the logic flow and specific modules to be used on the other brands. We should verify with official Ansible documentations for support and capabilities.

Project Conclusion
The goal of this project was not only to deploy routing and HA features but also to prove the deployment via raw packet captures between OSPF neighbors, show their OSPF status via Cisco command outputs, and completely automate their configuration status verification via state comparison.
