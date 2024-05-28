# Master-Slave-DNS-Bind9

**Information**\
Domain used: domain.loc [Change and use your own].\
Hostname: srv01 and srv02 [Change and use your own].\
srv01 IP address: 192.168.0.201 [Change and use your own].\
srv02 IP address: 192.168.0.202[Change and use your own].
****************************
**Perquisites**\
An Ubuntu / Debian machine\
In my scenario, I used "Ubuntu 22.04.4 LTS". The image below shows the details of our operating system:\
![image](https://github.com/mrkhorasani/Master-Slave-DNS-Bind9/assets/51242725/f4fa3e09-a309-4783-bbba-53053ad6d08d)

#Updating Ubuntu/Debian (Both servers srv01 and srv02)\
Before starting it is always a good practice to update your Linux system. To do this, only open up your terminal and type the following commands:

```
sudo apt -y update && sudo apt -y upgrade
```
Install Bind9 (Both servers srv01 and srv02)\
The next step is to install Bind9 along with some utilities.
```
apt -y install bind9 bind9-utils bind9-dnsutils
```
Configuring a static IP address (Both servers srv01 and srv02)
The next logical step is to set a static IP address. The first thing we will do is type the following command to find out the name of the network interface we are using.

sudo ip -c link show
Open the YAML configuration file for your netplan by typing the command below.
```
sudo nano /etc/netplan/01-network-manager-all.yaml
```
Enter the information below but customize it to fit your network information. I.E your interface name, IP address, etc.

```
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    Interfacename:
      dhcp4: no
      addresses:
        - Your_IP_address
      routes:
        - to: default
          via: GATEWAY_IP
      nameservers:
          addresses: DNS_IP1, DNS_IP2...
```
Now let’s apply this netplan by typing the command below.
```
sudo netplan apply
```
Finally, restart your network manager by using the command below.
```
sudo systemctl restart network-manager.service
```
Changing the host file (Both servers srv01 and srv02)
In this step, we will change the host file to include the fully qualified name of this Ubuntu machine. The fully qualified name is the machine name followed by the domain name. To do so first we open the host file. I will be using the Nano text editor.
```
sudo nano /etc/hosts
```
Once the file opens change the host and IP name with your own IP and hostname along with the fully qualified name (see figure 1 and 2).

figure 1. /etc/hosts file srv01\
![image](https://github.com/mrkhorasani/Master-Slave-DNS-Bind9/assets/51242725/2c76a86b-86c7-4353-8a0d-b3c1ce98aeca)
\

figure 2. /etc/hosts file srv02             
![image](https://github.com/mrkhorasani/Master-Slave-DNS-Bind9/assets/51242725/cde41d91-ba73-4af2-be1a-b705670c7a6a)
\
                
Next, we will restart our server for the changes to take effect.

```
sudo reboot
```
Changing the DNS server (Both servers srv01 and srv02)\
In this step, we will point our Ubuntu machine to use itself as a DNS server. To do this open ```/etc/resolv.conf``` using your favorite text editor, I will be using Nano. After this,  add the IP address of your Linux machine (see Figures 3 and 4).

figure 3. /etc/resolv.conf file srv01\
![image](https://github.com/mrkhorasani/Master-Slave-DNS-Bind9/assets/51242725/0861b8f3-4411-4efb-87ad-3eddc7dbeef6)

figure 3. /etc/resolv.conf file srv02\
![image](https://github.com/mrkhorasani/Master-Slave-DNS-Bind9/assets/51242725/87896482-5264-4426-86ea-7aeff873c776)

An issue that I ran into from time to time when changing my DNS server in Ubuntu, is that it reverts to using localhost.\
A simple way to get past this is removing the resolv.conf file > create it again, add the name server info, then deny access. Here are the commands below.

```
sudo rm /etc/resolv.conf #Deleting the resolv.conf file
```
create the file again, open it #with nano, and add the DNS server IP address
```
sudo nano /etc/resolv.conf
```
creating the file again,  opening it #with nano, and adding the DNS server IP address\
limiting access to the resolv.conf file, after adding the DNS server.
```
sudo chattr +i /etc/resolv.conf
```
**#Firewall Configuration**\
Permit (TCP & UDP port 53) in the firewall rule to use the DNS port
```
firewall-cmd  --permanent --add-port=53/tcp
firewall-cmd  --permanent --add-port=53/udp
```
Reload the firewall service
```
firewall-cmd --reload
```
******************************************
# Editing Bind9 options file (Master Server)

Now that we have laid the groundwork for our Bind DNS server, let’s start editing the configuration.

Start with editing the ```named.conf.options``` file. In this file, we will add what network/IP addresses can send a query to this server by defining an ACL (Access control list). We will also disable the recursion since we will be only using this server as an authoritative DNS server.\
Open the Bind options file using the command below.
```
sudo nano /etc/bind/named.conf.options
```
Edit the configuration below to fit your environment.
```
//Creating an ACL with the subnet that will be allowed to do DNS queries against this server
acl "trusted" {
 192.168.0.0/24;
};
options {
    directory "/var/cache/bind";
//allowing only the subnet within the ACL to query this server
    allow-query { trusted; };
    listen-on port 53 {localhost; 192.168.0.201; };
// enables recursive queries
    recursion yes;
    allow-recursion { trusted; };
//Disabling the zone transfer for anyone.
    allow-transfer { none; };
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };
    dnssec-validation auto;
    listen-on-v6 { any; };
};
```
You can check the syntax using the following command. If everything is correct, you should get no error.
```
sudo named-checkconf /etc/bind/named.conf.options
```
The next step is to edit the ```named.conf.local``` file to add some details about the zone we will create. To do so open the file using the command below:
```
sudo nano /etc/bind/named.conf.local
```
Edit the details below to fit your infrastructure (zone name and secondary server IP address), then add those details to your file and save it.

Make sure to add the correct path for your master zone. I will create the zone within a folder called Zones, which We will create in the next step.
```
//MasterZone
zone "domain.loc" {
 type master;
 file "/etc/bind/zones/domain.loc.db";
//secondary server IP address
 allow-transfer {192.168.0.202; };
 also-notify {192.168.0.202; };
};
```
The next step is to create the zones folder and then the zone file. To do so use the following commands. Note that you should edit the file name to fit your own zone name, and make sure to update the name in the configurations above as well.

**#Creating the zone folder**
```
sudo mkdir /etc/bind/zones
```
**#creating the zone file**
```
sudo touch /etc/bind/zones/domain.loc.db
```
The next step is to edit the file and configure our master DNS zone, along with adding some records for testing purposes.

To open the file, use the following command.
```
sudo nano /etc/bind/zones/domain.loc.db
```
Add the following configuration to your own file, keeping in mind that you need to change the details to fit your environment.

P.S: Make sure to increment the serial by one every time you make a change.
```
; BIND data file for local loopback interface
;
$TTL      86400                         ; (DAILY)
@         IN      SOA     srv01.domain.local. root.domain.local. (
                              5         ; Serial
                         604800         ; Refresh (WEEKLY)
                          86400         ; Retry   (DAILY)
                        2419200         ; Expire
                         604800         ; Negative Cache TTL
                         )
;Primary and secondary IP address and fully qualified name
@          IN      NS      srv01.domain.local.
           IN      NS      srv02.domain.local.
@          IN      A       192.168.0.201

;Arecords
srv01      IN      A       192.168.0.201
srv02      IN      A       192.168.0.202
web        IN      A       20.20.20.20
ftp        IN      A       12.12.12.12
```
The next step is to test the zone file using the named-checkzone utility. If there are no errors then your configurations are correct.
```
named-checkzone domain.loc /etc/bind/zones/domain.loc.db
```
The last step is to restart the service. To do so type the command below.
```
sudo service bind9 restart
```
That’s it for the master server.

Configuring the slave/secondary Bind DNS server
Following the steps above so far: the master server should be configured completely and the secondary/slave dns server should have a static IP address and Bind9 already installed.

The next step is to configure the secondary server to receive a copy of the master zone.

Much like the master server, the first thing we will configure is the Bind9 ```named.conf.options``` file. We will open the file using the following command.
```
sudo nano /etc/bind/named.conf.options
```
Edit the following configuration to fit your environment.
```
//Creating an ACL with the subnet that will be allowed to do DNS queries against this server
acl "trusted" {
 192.168.0.0/24;
};
options {
    directory "/var/cache/bind";
//allowing only the subnet within the ACL to query this server
    allow-query { trusted; };
    listen-on port 53 {localhost; 192.168.0.202; };
// enables recursive queries
    recursion yes;
    allow-recursion { trusted; };
//Disabling the zone transfer for anyone.
    allow-transfer { none; };
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };
    dnssec-validation auto;
    listen-on-v6 { any; };
};
```

You can check the syntax using the following command. If everything is correct, you should get no error.
```
sudo named-checkconf /etc/bind/named.conf.options
```
The next step is to create the zones folder.\
**#Creating the zone folder**
```
sudo mkdir /etc/bind/zones
```
The next step is to edit the file ```named.conf.local``` to add the zone information. To do so, use the following command.\
```
sudo nano /etc/bind/named.conf.local
```
Edit the following information to fit your environment, then add it to the file.
```
zone "domain.loc" {
 type slave;
 //Master zone name
 file "domain.loc.db";
 //Master server IP address
 masters {192.168.0.201; };
 allow-notify {192.168.0.201; };
};
```
Let’s recheck the syntax using the following command.
```
sudo named-checkconf /etc/bind/named.conf.local
```
Finally, let’s restart the DNS server.
```
sudo systemctl restart bind9
```
Now let’s check if the zone file has been transferred. This is usually transferred to the location ```/var/cache/bind/```. I will list all the files within that folder and do a search text on the word “domain” which is a part of my domain to list all files that include that word. Make sure to edit the command below to fit your domain name.
```
ls /var/cache/bind | grep domain
```
If all goes well, you should get a similar result as below, showing you that a copy of your master zone file exists on the secondary server.


That’s it. I hope this was relatively painless. Should you run into problems please do not hesitate to ask me....
