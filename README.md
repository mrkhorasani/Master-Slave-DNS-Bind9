# Master-Slave-DNS-Bind9

Information
Domain used: My.domain.loc [Change and use your own].\
Hostname: SRV1 and SRV2 [Change and use your own]./
SRV1 IP address: 192.168.0.252 [Change and use your own].
SRV2 IP address: 192.168.0.253[Change and use your own].
Perquisites
An Ubuntu / Debian machine
Updating Ubuntu/Debian (Both servers SRV1 and SRV2)
Before starting it is always a good practice to update your Linux system. To do this just open up your terminal and type the following commands:
![image](https://github.com/mrkhorasani/Master-Slave-DNS-Bind9/assets/51242725/3583ba74-b340-4a38-97d0-421860c31339)

sudo apt -y update && sudo apt -y upgrade
Install Bind9 (Both servers SRV1 and SRV2)
The next step is to install Bind9 along with some utilities.

![gtmetrix](https://github.com/mrkhorasani/Master-Slave-DNS-Bind9/assets/51242725/75113440-01dc-4471-b0df-d1c6a71bd619)


sudo nano apt -y install bind9 bind9-utils bind9-dnsutils
Configuring a static IP address (Both servers SRV1 and SRV2)
The next logical step is to set a static IP address. The first thing we will do is to type the following command to find out the name of the network interface we are using.

sudo ip -c link show
Open the YAML configuration file for your netplan by typing the command below.

sudo nano /etc/netplan/01-network-manager-all.yaml
Enter the information below but customize it to fit your network information. I.E your interface name, IP address etc.

network:
  version: 2
  renderer: NetworkManager
  ethernets:
    Interfacename:
      dhcp4: no
      addresses:
        - Your IP address
      gateway4: Your gateway IP address
      nameservers:
          addresses: Your DNS server IP address
Now let’s apply this netplan by typing the command below.

sudo netplan apply
Finally restart your network manager by using the command below.

sudo systemctl restart network-manager.service
Changing the hosts file (Both servers SRV1 and SRV2)
In this step we will change the hosts file to include the fully qualified name to this Ubuntu machine. The fully qualified name is the machine name followed by the domain name. To do so first we open the hosts file. I will be using Nano text editor.

sudo nano /etc/hosts
Once the file opens change the host and IP name with your own IP and hostname along with the fully qualified name (see figure 1 and 2).
![image](https://github.com/mrkhorasani/Master-Slave-DNS-Bind9/assets/51242725/935ae2d2-a630-4907-b2ea-e868f7c9b2d8)


figure 1. /etc/hosts file SRV1

figure 2. /etc/hosts file SRV2
Next we will restart our server for the changes to take effect.

sudo reboot
Changing the DNS server (Both servers SRV1 and SRV2)
In this step we will point our Ubuntu machine to use itself as a DNS server. To do this open /etc/resolv.conf using your favourite text editor, I will be using Nano. After this just add the IP address of your Linux machine (see figure 3 and 4).


figure 3. /etc/resolv.conf file SRV1

figure 3. /etc/resolv.conf file SRV2
An issue that I ran to from time to time when changing my DNS server in Ubuntu, is that it reverts to using localhost.

A simple way to get passed this is removing the resolv.conf file > create it again, add the name server info to it, then deny access. Here are the commands below.

sudo rm /etc/resolv.conf #Deleting the resolv.conf file
sudo nano /etc/resolv.conf #creating the file again and opening it #with nano and add the DNS server IP address
#limiting access to the resolv.conf file.
sudo chattr +i /etc/resolv.conf #after adding the DNS server 
Editing Bind9 options file (Master Server)
Now that we have laid the groundwork for our Bind DNS server, let’s start editing the configuration.

Start with editing the named.conf.options file. In this file we will add what network/IP addresses can send a query to this server by defining an ACL (Access control list). We will also disable the recursion since we will only be using this server as an authoritative DNS server.

Open the Bind options file using the command below.

sudo nano /etc/bind/named.conf.options
Edit the configuration below to fit your environment.

//Creating an ACL with the subnet that will be allowed to do DNS queries against this server
acl “trusted” {
 10.10.10.0/24;
};
options {
 directory “/var/cache/bind”;
//Disabling the zone transfer for anyone. 
 allow-transfer {none;};
//allowing only the subnet within the ACL to query this server 
 allow-query {trusted;};
 listen-on port 53 {localhost; 192.168.0.252;};
//Disabling recursion
 recursion no;
 dnssec-validation auto;
 listen-on-v6 { any; };
};
You can check the syntax using the following command. If everything is correct, you should get no error.

sudo named-checkconf /etc/bind/named.conf.options
The next step is to edit the named.conf.local file to add some details about the zone we will create. To do so open the file using the command below:

sudo nano /etc/bind/named.conf.local
Edit the details below to fit your infrastructure (zone name and secondary server IP address), then add those details to your file and save it.

Make sure to add the correct path for your master zone. I will create the zone within a folder called zones, that I will create in the next step.

//MasterZone 
zone “My.domain.loc” {
 type master;
 file “/etc/bind/zones/db.My.domain.loc”;
//secondary server IP address
 allow-transfer {192.168.0.253;};
 also-notify {192.168.0.253;};
};
The next step is to create the zones folder and then the zone file. To do so use the following commands. Note that you should edit the file name to fit your own zone name, and make sure to update the name in the configurations above as well.

#creating the folder
sudo mkdir /etc/bind/zones
#creating the zone file
sudo touch /etc/bind/zones/db.My.domain.loc
The next step is to edit the file and configure our master DNS zone, along with adding some records for testing purposes.

To open the file, use the following command.

sudo nano /etc/bind/zones/[your file name]
Add the following configuration to your own file, keeping in mind that you need to change the details to fit your environment.

P.S: Make sure to increment the serial by one every time you make a change.


The next step is to test the zone file using the named-checkzone utility. If there are no errors then your configurations are correct.

named-checkzone My.domain.loc /etc/bind/zones/db.My.domain.loc
The last step is to restart the service. To do so type the command below.

sudo service bind9 restart
That’s it for the master server.

Configuring the slave/secondary Bind DNS server
Following the steps above so far: the master server should be configured completely and the secondary/slave dns server should have a static IP address and Bind9 already installed.

The next step is to configure the secondary server to receive a copy of the master zone.

Much like the master server, the first thing we will configure is the Bind9 named.conf.options file. We will open the file using the following command.

sudo nano /etc/bind/named.conf.options
Edit the following configuration to fit your environment.

acl “trusted” {
 10.10.10.0/24;
};
options {
 directory “/var/cache/bind”;
 allow-transfer {none;};
 allow-query {trusted;};
 listen-on port 53 {localhost; 192.168.0.253;};
 recursion no;
 dnssec-validation auto;
 listen-on-v6 { any; };
};
You can check the syntax using the following command. If everything is correct, you should get no error.

sudo named-checkconf /etc/bind/named.conf.options
The next step is to edit the file named.conf.local to add the zone information. To do so, use the following command.

sudo nano /etc/bind/named.conf.local
Edit the following information to fit your environment, then add it to the file.

zone “My.domain.loc” {
 type slave;
 //Master zone name
 file “db.My.domain.loc”;
 //Master server IP address
 masters {192.168.0.252;};
 allow-notify {192.168.0.252;};
};
Let’s check the syntax again using the following command.

sudo named-checkconf /etc/bind/named.conf.local
Finally, let’s restart the DNS server.

sudo systemctl restart bind9
Now let’s check if the zone file has been transfered. This is usally transfered to the location “/var/cache/bind”. I will list all the files within that folder and do a search text on the word “linux” that is a part of my domain to list all files that includes that word. Make sure to edit the command below to fit your domain name.

ls /var/cache/bind | grep linux
If all went well, you should get a similar result as below, showing you that a copy of your master zone file exists on the secondary server.


That’s it. I hope this was relatively painless. Should you run into problems please do not hesitate to ask in the comments below.
