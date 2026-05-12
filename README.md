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
<img width="781" height="501" alt="image" src="https://github.com/user-attachments/assets/457a6e4e-90c2-469e-b68f-0baaf96cca9b" />
</p>
<br>

<p>
- To start configuration on the first router (R1), I connect my laptop with a RS-232 serial cable to R1’s console port to access its CLI.
</p>
<p>
<img width="767" height="486" alt="image" src="https://github.com/user-attachments/assets/33ee3c59-30c9-46ee-9f7b-1e1e6df1612d" />
</p>
<br>

<p>
- Now in R1’s CLI, I begin by assigning IP addresses to each of its interfaces. This router serves as both a DHCP server and a NAT device, so it must connect to multiple networks. The 172.16.10.0/27 subnet is used for the admin network where end devices automatically receive IP configuration. This network connects to Switch 1 via interface g0/1 and represents the internal administrative users. The 172.16.20.0/24 subnet is used for the user network where internal users will be translated for internet access. This network connects to Switch 2 via interface g0/2. The 203.0.113.0/30 subnet is used as the WAN link connecting Router 1 to the Internet router via interface g0/0. It is important to use a consistent addressing scheme, and in this lab the last usable IP address in each subnet is used as the default gateway. After configuring interfaces, a static route is added so R1 can reach R2 for the remote LAN subnet via the next-hop IP 203.0.113.2 on the WAN link. A default route is also added so any unknown traffic is forwarded toward the internet, it also uses the next-hop IP as the static route. 
</p>
<p>
<img width="772" height="530" alt="image" src="https://github.com/user-attachments/assets/6abcf07d-697c-4040-971f-a9e39e79d7dc" />
</p>
<br>

<p>
- Next, is R1’s DHCP configuration, it will provide DHCP services for the three separate networks. The administrative devices in the 172.16.10.0/27 admin network will be receiving automatic IP configuration. Because this is a smaller administrative subnet, fewer addresses are needed and NAT is not required. The 172.16.20.0/24 user network is the subnet used for the clients, and because it must always have available IP addresses for hosts it will be using NAT overload for internet access. The 172.16.30.0/27 subnet is the remote site network, where DHCP requests will be forwarded from a remote site using DHCP relay. Each subnet has its own DHCP pool. The first ten IP addresses in each subnet are excluded, it is good practice to reserve space for extra IPs in case they are needed. Each pool assigns its respective default gateway (last IP of the subnet), uses DNS server 8.8.8.8, and includes the domain name robertoporta.net.
</p>
<p>
<img width="776" height="589" alt="image" src="https://github.com/user-attachments/assets/cbef6e2a-efeb-4e4d-85af-02967f1bf8a4" />
</p>
<br>

<p>
- For the 172.16.20.0/24 user network subnet, Port Address Translation (PAT) also known as NAT overload is configured, allowing multiple internal hosts in this network to share the single public-facing IP address 203.0.113.1 on interface g0/0. The reason for this design is because this subnet represents the network that holds a large number of clients simultaneously, and IPs for internet access are required to always be available. The 172.16.10.0/27 admin network is not included in NAT because it contains fewer hosts and it is manageable to be used without the implementation of NAT overload. This is achieved by using an access-list to identify the internal traffic and the overload keyword to map multiple sessions to a single public IP using unique port numbers. Defining ip nat inside and outside on the interfaces establishes the boundary where the router intercept and rewrites packet headers. 
</p>
<p>
<img width="773" height="198" alt="image" src="https://github.com/user-attachments/assets/fc0a457f-7086-472e-9ef1-a710ec673ded" />
</p>
<br>

<p>
- Now that DHCP and NAT overload have been configured, I can verify that the PCs are correctly obtaining their IP addresses. In the PC configurations for the admin network, I make sure they are configured to use DHCP in its IP settings. This allows the PCs to request an IP address form R1, which allows R1 to successfully lease addresses to the 3 PCs. DHCP assigns IP addresses to clients using a lease system, meaning addresses are usually temporary rather than permanent. Lease times allows IP addresses to be reused efficiently, and if a device remains connected it will automatically request a new lease, seamlessly without losing connection. DHCP normally hands out IP addresses from lowest to highest available in the pool, if a returning client finds their old IP add taken they will usually get the next lowest one. In this example, we can see that PC1, PC2, and PC3 (in that order) get the first 3 IP addresses available in the pool after the excluded addresses.
</p>
<p>
<img width="758" height="176" alt="image" src="https://github.com/user-attachments/assets/395de904-2ab1-4d2d-85b0-75f07c38c9eb" />
</p>
<p>
<img width="759" height="178" alt="image" src="https://github.com/user-attachments/assets/6d7a3027-9e37-411c-ae42-74af797c7416" />
</p>
<p>
<img width="769" height="182" alt="image" src="https://github.com/user-attachments/assets/f7d377f3-7d39-4b7b-af9d-af6799a6c3b5" />
</p>
<br>

<p>
- For the user network, the clients also successfully get their IP address from DHCP, as shown in PC4’s IP configuration below. But the goal of this subnet is not DHCP, but it is to showcase the use of PAT (NAT overload), which is shown in the next step. PAT allows an entire subnet to share a single public IP address by assigning unique source port numbers to each individual translation entry. This is highly advantageous for large subnets, such as a /24, because it conserves limited public IPv4 addresses while still providing internet access to hundreds of internal hosts simultaneously. Without PAT, every device would require its own expensive public IP, but with it, the router can track thousands of separate traffic streams by using the 16-bit port field in the TCP/UDP headers.
</p>
<p>
<img width="775" height="185" alt="image" src="https://github.com/user-attachments/assets/0c578c77-68d8-426b-ad95-267c91ee9591" />
</p>
<br>

<p>
- To test PAT, from PC4 in the user network, I ping robertoporta.net using 1.1.1.1, which represents a website in the internet. The ping is successful, but when analyzing the packet, the PDU details show us that the Source IP field in the outbound packet changes as it moves from the PC to R1. While the packet starts (first image) with the PC's private address of 172.16.20.11 (the Inside Local), the Layer 3 header is rewritten (second image) to 203.0.113.1 (the Inside Global) before it hits the internet. This confirms that the router is successfully masking the internal network topology. This mechanism allows the Internet router to only ever see the single public WAN IP, even when multiple internal devices are communicating simultaneously. 
</p>
<p>
<img width="775" height="734" alt="image" src="https://github.com/user-attachments/assets/25cdcec2-0a7a-4223-9c60-d9c1361c7cd0" />
</p>
<p>
<img width="774" height="709" alt="image" src="https://github.com/user-attachments/assets/51891299-6fb9-4624-8759-75a51a8e2930" />
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
