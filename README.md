# Implementing-DHCP-Server-Relay-NAT-Overload-and-Remote-SSH-Administration-

<h2>Implementing DHCP Server Relay NAT Overload and-Remote SSH Administration</h2>

<p>
- This lab demonstrates four core networking concepts: Dynamic Host Configuration Protocol (DHCP), Network Address Translation (NAT Overload), DHCP relay using the “ip helper-address” feature, and Secure Shell (SSH) for remote management. The topology represents an enterprise network where Router 1 serves as the main gateway and Router 2 acts as a remote branch dependent on centralized services.
As the lead engineer at the main site, I manage all enterprise subnets. While a remote admin pre-configured R2 with basic interfaces and SSH, I must integrate the site into our infrastructure. This involves using SSH to configure DHCP relay so the branch can receive IP addresses from the main site, while providing ongoing oversight to ensure the remote site's routing and connectivity remain stable and performant. 
The design omits VLANs because each switch is physically connected to a router interface, making Layer 2 segmentation unnecessary. Using distinct Layer 3 subnets instead of VLANs allows the lab to focus directly on routing, DHCP scope design, and NAT policy. The network uses three functional segments: SW1 is a management network for administrative devices, SW2 is the primary user network requiring NAT for internet access, and SW3 is a remote branch network requiring centralized DHCP via relay.
A centralized DNS server (8.8.8.8) is distributed via DHCP to allow name resolution. Final validation will confirm connectivity using the domain robertoporta.net, verifying that DHCP, routing, NAT, and DNS are fully integrated. The external network is simulated using a router with a cloud icon to represent an ISP. This upstream transit network handles traffic between enterprise sites and external destinations to simulate real-world internet access in a controlled environment.
The following screenshot shows the topology before configuration.
</p>
<p>

</p>
<br>

<p>
- 
</p>
<p>

</p>
<br>

<p>
- 
</p>
<p>

</p>
<br>
