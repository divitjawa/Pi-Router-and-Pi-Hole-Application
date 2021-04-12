# 1. Raspberry Pi Router

**A summary of what I did in this project. Used a Raspberry Pi, and turned it into a WiFi router.**

**Step 1:** Install Raspberry Pi OS Lite on to your pi using a Micro SD card, and enable SSH

After installing the OS on to the Micro SD card, insert the card into your Pi. Open Powershell, and log in using ssh by typing `ssh pi@raspberrypi.local`. The default password for the pi user is `raspberry`. You can change it after logging in by using the command `passwd`.

After logging in, set timezone, keyboard type, editor, etc. You can also update the hostname from pi to anything that you'd like by using the command `sudo hostnamectl set-hostname <INSERT NAME HERE>`. You should also replace any references to raspberrypi in /etc/hosts by using the command `sudo nano /etc/hosts`. After this, you can feel free to add your public SSH key to the pi, and enabling passwordless login.

**Step 2:** Connect to WiFi

Wireless settings for the Pi are controlled by a service called *wpa_supplicant*, which stores network connection settings inside `/etc/wpa_supplicant/wpa_supplicant-wlan0.conf`. The updated version of the file has been added to the repository under the CONFIGS folder. Once you are done with that, you can reconfigure the wireless interface by calling `wpa_cli -i wlan0 reconfigure`. You can check that you are connected wirelessly by using the command `wpa_cli -i wlan0 status`.

It would also be useful to update packages using apt, and install dnsutils using `sudo apt install dnsutils`.

**Part 3: Enable Networkd:** Current version of Debian supports systemd based network management via the systemd-networkd service. Unlike dhcpcd, networkd is well-suited to managing the base configuration for our router.

We will create 2 .network files in `/etc/systemd/network/`. Since the default service for Raspbian networking is a component called dhcpcd, we will disable it.

Use these commands to disable dhcpcd: `sudo systemctl mask dhcpcd.service` & `sudo systemctl mask networking.service`. We will also disable resolvconf, which manages DNS queries for the OS. by adding a line at the beginning of the file that says `resolvconf=NO`. Do this in the file using this command: `sudo nano /etc/resolvconf.conf`. 

Now, we can finally enable systemd services by using the commands: `sudo systemctl enable systemd-networkd` & `sudo systemctl enable systemd-resolved`. We also want to manage wpa_supplicant as an interface. For that, we must enable the service specifically for wlan0, and disable the non-interface version using the commands below. 

`sudo systemctl enable wpa_supplicant@wlan0` & `sudo systemctl disable wpa_supplicant` 

Something to note while creating a network block in your wpa_supplicant file is to **not put a plaintext password** in the block. We can avoid doing so by assigning the passphrase to the psk parameter. We can compute a hashed passphrase using a function called PBKDF in conjunction with SHA1. To do so, put in your network SSID in the command: `wpa_passphrase "Home Wifi"`. This will then prompt you for the passcode. Enter it, and it will generate a hash, which we will then paste in place of the plaintext passcode in the network block. 

To apply these change, run the command `wpa_cli -i wlan0 reconfigure`.

**Part 4: LAN Planning & Installing DHCP Server:** We must plan how to assign addresses. In order to avoid conflicts, the addresses must not overlap with these common subnets: 

- 10.0.0.0/24
- 10.1.0.0/24
- 192.168.0.0/24
- 192.168.1.0/24

I created address planning using subnet prefix 25(CIDR/25), and it supports a minimum of 15 hosts. LAN planning for my project can be found **here**.

Now, we will create the second .network file which will hold the address range that we selected in CIDR notation.

Now, we will install the DHCP server using `sudo apt install isc-dhcp-server`. After that, we must specify the interface as `eth0` in `/etc/default/isc-dhcp-server`, since right now we only have access to a wired internet connection. Remember, we are creating a router, and for this project, we do not have access to a wireless connection through our device. Restart the dhcp server using `sudo systemctl restart isc-dhcp-server.service`.

**Part 5: NAT Enabled Router:** In this part we will set up NAT and enable packet forwarding. We will configure the Pi to be a simple  gateway that routes traffic between an isolated, wired LAN and a wireless network.

NAT functionality is supported natively in Linux through the netfilter firewall. We'll use a common Linux tool known as iptables to manage this. We must install iptables using the command `sudo apt install iptables-persistent`. We will apply certain rules, add them to `/etc/iptables/rules.v4` and create a basic NAT configuration file that will allow us to route traffic. Instead of adding the rules directly to rules.v4, we will create a file called untested_rules in the Pi's home directory, and then apply those rules using the command `sudo iptables-apply -t 60 -w /etc/iptables/rules.v4 ~/untested_rules`. This is much more secure.

After this step, we want to enable packet forwarding by going to `/etc/sysctl.d/99-sysctl.conf` and setting `net.ipv4.ip_forward=1`.

**Part 6: Enable Caching DNS Server:** In this step we will add a DNS server using Bind that will resolve the DNS queries. This will replace the public DNS servers provided by your ISP. 

Install Bind using `sudo apt-get install bind9 bind9utils bind9-doc`. Move into the folder where Bind config files are using `cd /etc/bind`. We must now edit the config file using `sudo nano named.conf.options`. To avoid the possibility of your server being used for malicious purposes, we will configure a list of IP addresses or network ranges that we trust. Create an acl block in the file, and add your trusted IP addresses. 

After that, in the options block, turn recursion on and set `allow-query { goodclients; };`. This will allow only our good clients to use the recursion services.

**Final Comments:**

This is the end of the project. We successfully configured a wireless router on a Raspberry Pi that has a DHCP server, a DNS resolver, is decently secure, and can turn your old computer into a wireless enabled one! There are different checkpoints in the checkpoint folder that demonstrate that the desired outcome was achieved.

Summary of files:

|       **File Name**       |                      **File Contents**                       |
| :-----------------------: | :----------------------------------------------------------: |
| wpa_supplicant-wlan0.conf | Wireless settings for the Pi are controlled by a service called *wpa_supplicant*, which stores network connection settings. An interface-specific configuration where the interface name is appended to the file. |
|      99-wlan.network      | Default wireless configuration inside `/etc/systemd/network/` |
|      20-eth0.network      | Configuration file that includes your chosen IP address and CIDR range inside `/etc/systemd/network/`. Naming the file in such a way that networkd will load it first (networkd processes these files in number order; lower number = loaded first) and by ensuring that the configuration will match `eth0` exactly. |
|        dhcpd.conf         | This file contains our subnet declaration to configure the DHCP server and to distribute addresses within our predefined address range. |
|    untested_rules.txt     |                 Contains NAT(iptables) rules                 |
|    named.conf.options     |             BIND DNS resolver configuration file             |



# 2. Pi-Hole

[Pi-Hole](https://en.wikipedia.org/wiki/Pi-hole) is a Linux network-level [advertisement](https://en.wikipedia.org/wiki/Online_advertising) and Internet tracker blocking [application](https://en.wikipedia.org/wiki/Application_software) which acts as a [DNS sinkhole ](https://en.wikipedia.org/wiki/DNS_sinkhole)and optionally a [DHCP server](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol), intended for use on a [private network](https://en.wikipedia.org/wiki/Private_network). It is designed for low-power embedded devices with network capability, such as the [Raspberry Pi](https://en.wikipedia.org/wiki/Raspberry_Pi), but supports any [Linux](https://en.wikipedia.org/wiki/Linux) machines. (Wikipedia) 

This was another project that I had worked on, which was similar to the one above, in terms of set up, however, it had a completely different functionality.

Initial set up is the same of installing Raspberry Pi OS Lite and enabling ssh.

**Part 2:** Install the Pi-Hole Application

Install the application by running this command in your Pi's terminal window: `curl -sSL https://install.pi-hole.net | bash`. Complete the installation setup.

**Part 3:** Install Recursive DNS Server

Here, we will install a recursive DNS server. However, this time we will not use BIND, instead, we'll use Unbound. It is the recommended DNS server for the Pi-Hole application. Install it using the command `sudo apt install unbound`. Configure the DNS server by `sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf`. 

![image-20210412052126021](C:\Users\divit\Desktop\Senior Year\Pi-Router-and-Pi-Hole-Application\assets\image-20210412052126021.png)

![image-20210412052147874](C:\Users\divit\Desktop\Senior Year\Pi-Router-and-Pi-Hole-Application\assets\image-20210412052147874.png)

Restart unbound using `sudo service unbound restart`.

**Part 4:** Assign DNS Server

In this part we will tell our computer to use the Pi-Hole as the DNS server, instead of using the default DNS server.

For **Windows**, type `ncpa.cpl` in PowerShell, locate your network adapter, right click on it, and select properties. Look for ipv4 and double click on it. Enter the Pi-Hole static address in the "Use the following DNS server addresses:". That should be it!

For **Mac OS**, Apple Menu -> Sys. Pref. -> Network -> locate your connection(WiFi) -> Advanced -> Select DNS from the top row and click "Add". Enter the Pi-Hole static address here. That's all!

Here are some screenshots of the Pi-Hole working for me.

![image-20210311201255996](file://C:\Users\divit\Desktop\Senior Year\Pi-Router-and-Pi-Hole-Application\assets/image-20210311201255996.png)

![image-20210311201506830](file://C:\Users\divit\Desktop\Senior Year\Pi-Router-and-Pi-Hole-Application\assets\image-20210311201506830.png)

Screenshot from within the Pi:

![image-20210311201519346](file://C:\Users\divit\Desktop\Senior Year\Pi-Router-and-Pi-Hole-Application\assets\image-20210311201519346.png)