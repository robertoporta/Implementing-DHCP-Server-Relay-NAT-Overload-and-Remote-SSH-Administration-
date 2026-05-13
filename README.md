#
<h1>Implementing DHCP Server Relay NAT Overload and Remote SSH Administration</h1>

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
- After generating traffic with pings from some of the active PCs in the user network, the NAT translation table on R1 populates to show active sessions. Each entry maps an Inside Local address (the private IP of the PC) to the Inside Global address (the public IP of the R1 WAN interface). Although the unique source port used for this translation is not visible in this specific PDU shown in the previous step (as ICMP is a Layer 3 protocol that lacks traditional TCP/UDP port fields) the router still tracks the session internally using the ICMP Identifier to ensure the reply returns to the correct host.
</p>
<p>
<img width="769" height="266" alt="image" src="https://github.com/user-attachments/assets/31dcebbf-6373-4631-b22e-0921af691ab5" />
</p>
<br>

<p>
- Now onto the second router (R2), which will be configured through SSH to ensure secure configuration changes over the WAN link by encrypting credentials and session data. This is required because R2 is being accessed over the network rather than through direct console access as it is part of the remote site of the organization. A network administrator at the remote site has already completed the initial setup of the interfaces and of SSH. For SSH to be enabled in R2, the following requirements were configured: a local username, domain name (robertoporta.net), RSA key generation, and enabling of all VTY lines (0 through 15). SSH version 2 was also specified.
</p>
<p>
<img width="775" height="383" alt="image" src="https://github.com/user-attachments/assets/c423195c-13e6-464c-b5fc-9d8b7aa26309" />
</p>
<br>

<p>
- In addition to R2’s SSH configuration, the network administrator of the remote site also had to configure its interfaces. The 172.16.30.0/27 subnet is used for the remote LAN where end devices will receive IP addresses via DHCP relay and connect locally to the remote switch via interface g0/1. Interface g0/1 is configured with 172.16.30.254/24 for the remote LAN subnet, while interface g0/0 is configured with 203.0.113.5/30 for the WAN link connecting this remote site to the internet. This separation allows the router to simulate a branch network that depends on external services, such as DHCP provided by another router. Lastly, static routes are added so Router 2 can reach R1, the admin network, and the user network via the next-hop IP 203.0.113.6 on the WAN link. 
</p>
<p>
<img width="775" height="387" alt="image" src="https://github.com/user-attachments/assets/00cea1b4-cb22-4306-972e-8726dd0130af" />
</p>
<br>

<p>
- Now that R2 has SSH configured and its interfaces properly set up, I establish a remote session from my laptop command prompt using the router’s WAN IP address (203.0.113.5). I am connecting my laptop to Switch 1 using a Fast Ethernet cable to pull a temporary IP address from the Admin DHCP pool, effectively acting as a mobile management station. While the laptop is not a permanent member of the Admin Network, this connection allows me to verify that the management segment has the correct permissions and routing path to reach the remote site. This setup tests the "In-Band" management path, forcing the SSH traffic to traverse from the local switch, through R1, and across the simulated internet link. Successful login confirms that SSH authentication and end-to-end routing connectivity across the WAN are functioning correctly before the laptop is disconnected. I am now able to configure R2 from my laptop back at the admin network.
</p>
<p>
<img width="775" height="492" alt="image" src="https://github.com/user-attachments/assets/3e19e5f4-b843-49e3-9651-664b30d6a9ad" />
</p>
<p>
<img width="760" height="187" alt="image" src="https://github.com/user-attachments/assets/5a43231f-04ca-4fcb-b8bf-46b113802beb" />
</p>
<p>
<img width="777" height="282" alt="image" src="https://github.com/user-attachments/assets/ac907731-1222-457f-8043-60dc72fdad1e" />
 />
</p>
<br>

<p>
- To provide IP addresses to hosts in the remote site, I have to setup DHCP relay. Devices on the 172.16.30.0/27 remote LAN subnet cannot directly reach the DHCP server because DHCP broadcasts do not cross router boundaries. To resolve this, I configured the ip helper-address on the G0/1 interface of R2. This command converts local DHCP broadcasts into unicast packets and forwards them to the centralized DHCP server at 203.0.113.1 on R1.
Additionally, I configured R2 with a global DNS pointer to ensure the router itself can resolve enterprise domain names during remote troubleshooting. While the ip domain-lookup command is enabled by default on most modern Cisco IOS versions, explicitly including it in the configuration is a professional best practice. This ensures consistent name resolution behavior if the configuration is ever migrated to older legacy hardware where the feature might be disabled by default.
</p>
<p>
<img width="734" height="170" alt="image" src="https://github.com/user-attachments/assets/2ba9ae13-094c-4ec7-9cfb-aab00d11b264" />
</p>
<br>

<p>
- Now that DHCP relay has been configured, the devices in the remote site are able to obtain IP addresses with DHCP, without the devices being able to directly reach the DHCP server at R1. Just like the the other subnets, the IPs given out start after the excluded addresses.
</p>
<p>
<img width="780" height="185" alt="image" src="https://github.com/user-attachments/assets/551ff06f-7754-429c-8413-bc8e46d4519c" />
</p>
<p>
<img width="780" height="187" alt="image" src="https://github.com/user-attachments/assets/6433870c-d7a8-41c4-9726-3ce07d5252ef" />
</p>
<p>
<img width="780" height="182" alt="image" src="https://github.com/user-attachments/assets/ca643e53-ad64-4859-9c81-7f12753b97df" />
</p>
<br>

<p>
- To conclude the lab, I disconnected my management laptop and requested the remote site administrator perform final connectivity validation from the remote network. First, PC9 successfully pinged PC2 in the admin network, confirming internal routing between sites. Next, PC10 reached PC5 in the user network, verifying that the branch can access primary site resources. Finally, PC11 pinged the domain robertoporta.net (resolving to the 1.1.1.1 address used in a previous test), the success of this test confirms that DNS services, DHCP relay, and external routing are all fully operational across the integrated network. 
</p>
<p>
<img width="770" height="492" alt="image" src="https://github.com/user-attachments/assets/40f9f980-1898-42d5-b467-2a40403d14c2" />
</p>
<p>
<img width="775" height="357" alt="image" src="https://github.com/user-attachments/assets/755b0ba2-3358-4c13-a4eb-69eb048acd36" />
</p>
<p>
<img width="771" height="362" alt="image" src="https://github.com/user-attachments/assets/9a27d499-3907-4fe6-bf28-8baad7949528" />
</p>
<p>
<img width="771" height="364" alt="image" src="https://github.com/user-attachments/assets/902ea7c0-15e3-45b5-97f6-ad20f0792dca" />
</p>
<br>
