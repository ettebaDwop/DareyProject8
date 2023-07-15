#  PROJECT 8: Load Balancer Solution With Apache
## Brief Overview
This project enhances the tooling project implemented in project 7. It introduces the Load Balancing concept to ensure that the DevOps Tooling websites will be scalable to serve more users by deploying additional Web Servers using a horizontal architecture. This project will utilise the Apache Load Balancer and local DNS Names resolution.

![proj8](https://github.com/ettebaDwop/dareyProject8/assets/7973831/ff64d604-485a-4b36-b348-271d09b5855c)

## Task
To deploy and configure an Apache Load Balancer for the Tooling Website solution implemented in Project 7 on a separate Ubuntu EC2 instance. This should enable users to be served by Web servers through the Load Balancer.

The task involves a simple implementation using 2 Web Servers. This approach can be extended to 3 or more Web Servers.

## Prerequisites
The prerequisites for this project have already been configured in project 7:
- Two RHEL8 Web Servers
- One MySQL DB Server (based on Ubuntu 20.04)
- One RHEL8 NFS server
  
![Prereq_proj8](https://github.com/ettebaDwop/dareyProject8/assets/7973831/c7c3c702-355c-4ce8-b485-aa6e344d820b)

## Implementation
#### Configure Apache As A Load Balancer
1. Create an Ubuntu Server 22.04 EC2 instance and name it *Project-8-apache-lb* :
  
![Screenshot (503)](https://github.com/ettebaDwop/dareyProject8/assets/7973831/101ec19c-97d7-4aec-a21d-25bfdd010ad6)
   
2. Open TCP port 80 on Project-8-apache-lb by creating an Inbound Rule in Security Group.

![Screenshot (504)](https://github.com/ettebaDwop/dareyProject8/assets/7973831/23a9b718-9300-40b7-aa93-0bc1700aa642)

3. Install Apache Load Balancer on Project-8-apache-lb server and configure it to point traffic coming to LB to both Web Servers:
   
#Install apache2

```
sudo apt update
sudo apt install apache2 -y
sudo apt-get install libxml2-dev

#Enable following modules:
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic

#Restart apache2 service
sudo systemctl restart apache2
```

Make sure apache2 is up and running

`sudo systemctl status apache2`


Configure load balancing

```sudo vi /etc/apache2/sites-available/000-default.conf

#Add this configuration into this section <VirtualHost *:80>  </VirtualHost>

<Proxy "balancer://mycluster">
               BalancerMember http://<WebServer1-Private-IP-Address>:80 loadfactor=5 timeout=1
               BalancerMember http://<WebServer2-Private-IP-Address>:80 loadfactor=5 timeout=1
               ProxySet lbmethod=bytraffic
               # ProxySet lbmethod=byrequests
        </Proxy>

        ProxyPreserveHost On
        ProxyPass / balancer://mycluster/
        ProxyPassReverse / balancer://mycluster/

#Restart apache server

sudo systemctl restart apache2
```



The *bytraffic* balancing method will distribute incoming load between your Web Servers according to current traffic load. We can control in which proportion the traffic must be distributed by loadfactor parameter.

4. Verify that our configuration works – try to access your LB’s public IP address or Public DNS name from your browser:
   
`http://<Load-Balancer-Public-IP-Address-or-Public-DNS-Name>/index.php`

Note: If in the Project-7 you mounted /var/log/httpd/ from your Web Servers to the NFS server – unmount them and make sure that each Web Server has its own log directory.

Open two ssh/Putty consoles for both Web Servers and run following command:

`sudo tail -f /var/log/httpd/access_log`

Try to refresh your browser page http://<Load-Balancer-Public-IP-Address-or-Public-DNS-Name>/index.php several times and make sure that both servers receive HTTP GET requests from your LB – new records must appear in each server’s log file. The number of requests to each server will be approximately the same since we set loadfactor to the same value for both servers – it means that traffic will be disctributed evenly between them.

If you have configured everything correctly – your users will not even notice that their requests are served by more than one server.

# Optional Step – Configure Local DNS Names Resolution
Sometimes it is tedious to remember and switch between IP addresses, especially if you have a lot of servers under your management.
What we can do, is to configure local domain name resolution. The easiest way is to use /etc/hosts file, although this approach is not very scalable, but it is very easy to configure and shows the concept well. So let us configure IP address to domain name mapping for our LB.

```
#Open this file on your LB server

sudo vi /etc/hosts

#Add 2 records into this file with Local IP address and arbitrary name for both of your Web Servers

<WebServer1-Private-IP-Address> Web1
<WebServer2-Private-IP-Address> Web2
```
Now you can update your LB config file with those names instead of IP addresses.

```
BalancerMember http://Web1:80 loadfactor=5 timeout=1
BalancerMember http://Web2:80 loadfactor=5 timeout=1
```
You can try to curl your Web Servers from LB locally curl http://Web1 or curl http://Web2 – it shall work.

Remember, this is only internal configuration and it is also local to your LB server, these names will neither be ‘resolvable’ from other servers internally nor from the Internet.
