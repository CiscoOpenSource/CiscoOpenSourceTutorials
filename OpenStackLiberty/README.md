# OpenStack Liberty
[http://twitter.com/vallard](@vallard)

## Introduction

Is OpenStack dead?  Or is OpenStack too big to fail?  Maybe both, but
as I need a cloud and I need it in an internal data center, right now
its the easiest thing I've got.  

In this tutorial we'll install the OpenStack Liberty release on 
Ubuntu 14.04.  We will use Ansible to configure and install OpenStack.
We will also configure aggregate groups so that we can run some big
data workloads on the few machines that actually have SSDs. 

Can we do it faster?  Sure, you could use something like packstack from 
RedHat and get it up in a fraction of the time, but how are you going to 
learn anything if you have scripts doing everything for you? Hands on helps
you troubleshoot and gain understanding for when things go wrong, because,
well... They always do... at some point anyway.

In this tutorial we will mostly follow the 
[http://docs.openstack.org/liberty/install-guide-ubuntu/environment.html](official documentation) 
as found on the OpenStack website.  But we'll have some opinions and things
to say about it. 

## Physical Architecture

Our architecture is a mixed bag of different servers we have
in the lab.  It's a frankenstack, but it can work beautifully for
the tests we are looking to do.  

### Controller Node
We have one controller node for our system.  In production workloads
like what Metapod provides you would have 3 control nodes.  We aren't
too concerned with that.  

The Controller Node is C240 with 2 x 120GB SSD hard drives.  It has 
128GB of RAM that I'll probably change and add to one of the data nodes.

The controller node will run: 
* Keystone
* Glance
* Nova Controller
* Neutron Networking


### Data Nodes

We have a few data nodes.  These are C240s that we will specifically
be using to run HDFS and other big data stacks on.  They have about 12
drives of SSDs! 

### Other Compute

We also have several blades we will be using for compute.  This will 
serve us well to run standard nodes on.  


## Installing the controller node

There are lots of automated ways to do this, but we just used 
the default Ubuntu Server 14.04.6 we found from Ubuntu's website.  

### Enable remote ssh
Who wants to enter a password everytime we log in? 
```
ssh-keygen -t rsa
```
from your laptop: 
```
scp ~/.ssh/id_rsa.pub controller01:/home/vallard/.ssh/authorized_keys
```
you should be able log in seemlessly!

### Sudo with no password

The administrator user that we created we allowed to ```sudo``` without a
password.  This was done to make sure the ansible scripts could run
properly. 

This was done with: 
```
sudo visudo
```
Then adding to the bottom of the file:
```
vallard ALL=(ALL) NOPASSWD:ALL
```
This will allow the ```vallard``` user to run sudo commands without entering
passwords. 

### Set host name

```
sudo hostname controller
```
Edit ```/etc/hosts``` to make sure that your host name matches.  Mine looks like: 

```
127.0.0.1       localhost
192.168.2.210   controller01

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
### Make sure DNS is set right
In my case I'm going through a proxy server and DNS has to be set up.  I fixed it
by modifying ```/etc/resolv.conf``` and adding my name servers


### Update the machine
```
sudo apt-get update 
sudo apt-get upgrade
```

### Network Settings

The public network is a GbE connection and has been numbered on my machine as p1p1.
I also have dual 10GbE connections leading up to a pair of Cisco Nexus switches.  These
were easy to see by running ```ifconfig```.  We now need to bond these and give them 
a private IP address. 

The Nexus has been configured with a [http://www.cisco.com/c/en/us/products/collateral/switches/nexus-5000-series-switches/configuration_guide_c07-543563.html](VPC) so we have the machine bonded.  

To create the bond on the controller node [https://help.ubuntu.com/community/UbuntuBonding](we follow the instructions)
```
sudo apt-get install ifenslave
```
Make sure we ```/etc/modules``` loads this file on boot time: 
```
lp
rtc
bonding
```

Next add the following to ```/etc/network/interfaces```
```
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto p1p1
iface p1p1 inet static
        address 10.93.234.96
        netmask 255.255.255.0
        network 10.93.234.0
        gateway 10.93.234.1
        dns-nameservers 171.70.168.183 173.36.131.10

auto bond0
iface bond0 inet static
        address 192.168.2.210
        netmask 255.255.255.0
        network 192.168.2.0
        bond-mode 4
        bond-miimon 100
        bond-lacp-rate 1
        bond-slaves p4p1 p4p2

auto p4p1
        iface p4p1 inet manual
        bond-master bond0
auto p4p2
        iface p4p2 inet manual
        bond-master bond0
```
Then run:
```
sudo modprobe bonding
sudo ifup bond0
sudo ifup p1p2
```

### NTP
```
sudo apt-get install -y chrony
```
Edit /etc/chrony/chrony.conf to look like:
```
server ntp.esl.cisco.com iburst
keyfile /etc/chrony/chrony.keys
commandkey 1
driftfile /var/lib/chrony/chrony.drift
log tracking measurements statistics
logdir /var/log/chrony
maxupdateskew 100.0
dumponexit
dumpdir /var/lib/chrony
local stratum 10
allow 10/8
allow 192.168/16
allow 172.16/12
logchange 0.5
rtconutc
```
Make sure the service is stopped:
```
service chrony stop
```
Now set the clock to the right time: 
```
ntpdate -bs ntp.esl.cisco.com
hwclock -w
```
Now start the chrony service: 
```
service chrony start
```
If you need to set the right time zone do something like: 
```
cp /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
```
Now you should be rolling with good time!

### Install OpenStack Repositories
```
sudo apt-get install -y software-properties-common
sudo add-apt-repository -y cloud-archive:liberty
sudo apt-get install -y python-openstackclient
```

### MariaDB SQL Database
```
sudo apt-get install mariadb-server python-pymysql 
```
__Note: If the python-pymysql package can't be found try running apt-get update and then
apt-get upgrade__

#### Tangent
At this point I took a tangent, which I shouldn't have and ended up running a few
unnecessary commands:
```
sudo apt-cache pkgnames python | grep sql
```
Couldn't find it, so I did: 
```
sudo apt-get install -y python-pip
sudo pip --proxy http://proxy.esl.cisco.com install PyMySQL
```
(you'll note I had to use my proxy server to get out)


Next we created the ```/etc/mysql/conf.d/mysqld_openstack.cnf``` file. 
Mine looks like: 
```
[mysqld]
bind-address = 192.168.2.210
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
```
Next restart MariaDB:

```
service mysql restart
```
I ran ```mysql_secure_installation``` and did allow remote root access even
though its probably not the most secure thing to do. 

### NoSQL database
Because I'm not using Telemetry services, I didn't install any NoSQL database. 

### Rabbit MQ Installation
```
sudo apt-get install -y rabbitmq-server
sudo rabbitmqctl add_user openstack Cisco.123 # change this to a different password
sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

### Identity Service 

From here on out we'll just run the commands as the root user to save us some keystrokes:
```
sudo su -
```
Now we'll create a database for the Identity service (keystone) to use
```
mysql -u root -p
```
Then I ran the following:
```
MariaDB [(none)]> CREATE DATABASE keystone;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'Cisco.123';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'Cisco.123';
Query OK, 0 rows affected (0.00 sec)
quit;
```
Obviously, you would replace Cisco.123 by your password. 

Now just follow the directions: 
```
echo "manual" >/etc/init/keystone.override
apt-get install -y keystone apache2 libapache2-mod-wsgi memcached python-memcache
```

Create an admin token with
```
openssl rand -hex 10
```
Copy and paste that token into the ```/etc/keystone/keystone.conf``` file as shown: 
```
[DEFAULT]
verbose = True
admin_token = 6a5e0722b889dea9f8b1
log_dir = /var/log/keystone

[database]
connection = mysql+pymysql://keystone:Cisco.123@controller01/keystone

[memcache]
servers=localhost:11211

[revoke]
driver = sql

[token]
provider= uuid
driver = memcache

[extra_headers]
Distribution = Ubuntu
```
Now I ran: 
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
and got a big time error.  Apparently, I couldn't connect to my database as the keystone user. I found out
that my host name was bound to the wrong IP address.  So changing ```/etc/hosts``` and making sure my entry was
```
192.168.2.210 controller01
```
made it work.

#### Apache Server
The HTTP server is used for the WSGI listener for keystone.  
```
echo "ServerName controller01" >> /etc/apache2/apache2.conf
```
 
Next I copied and pasted the contents of (http://docs.openstack.org/liberty/install-guide-ubuntu/keystone-install.html)[the wsgi conf]
to ```/etc/apache2/sites-available/wsgi-keystone.conf```

Then ran:
```
ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled
service apache2 restart
rm -f /var/lib/keystone/keystone.db
```
#### Service Creation

We now have something that looks like keystone installed.  Now we have to create some services for the
rest of it to work. 
```
export OS_TOKEN=6a5e0722b889dea9f8b1
export OS_URL=http://controller01:35357/v3
export OS_IDENTITY_API_VERSION=3
```

Let's now install the service: 
```
openstack service create --name keystone --description "OpenStack Identity" identity
```

At this point it all stopped working for me.  I figured my packages were out of date. 
```
apt-get update
apt-get upgrade
```
Hurray!  After this, I could actually install the elusive ```python-pymysql``` package.
I also found that my rabbitmq-server was using the old hostname, so I had to kill it and
start it up again.  I had to uninstall the python-openstackclient and then reinstall to 
get the latest update. Same with the keystone packages. I also had to regenerate my 
database.  (I ran: ```drop keystone``` from mysql and then reinstalled... shesh!)

After that everything started working again.  So we ran: 
```
openstack endpoint create --region RegionOne identity public http://controller01:5000/v2.0
openstack endpoint create --region RegionOne identity internal http://controller01:5000/v2.0
openstack endpoint create --region RegionOne identity admin http://controller01:35357/v2.0
```
[http://docs.openstack.org/liberty/install-guide-ubuntu/keystone-users.html](Now start here to finish..)

```
openstack project create --domain default --description "Admin Project" admin
openstack user create --domain default --password-prompt admin
# enter password
openstack role create admin
openstack role add --project admin --user admin admin
```
Service stuff:
```
openstack project create --domain default --description "Service Project" service
openstack project create --domain default --description "Demo Project" demo
```

Create a user for myself
```
openstack user create --domain default --password-prompt vallard
openstack role create user
openstack role add --project demo --user vallard user
```

#### Verify Keystone

Edit ```/etc/keystone/keyston-paste.ini``` and get rid of ```admin_token_auth``` from the pipeline stanzas.

Next, append some values to the ```.profile``` in your admin user account:
```
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=Cisco.123
export OS_AUTH_URL=http://controller01:35357/v3
export OS_IDENTITY_API_VERSION=3
```
This created my admin account. I verified by running ```openstack token issue``` to make sure
there were no problems.  Worked great!


#### Glance Installation

Go into mysql and create the account for Glance. 
```
mysql -u root -p
```
Entering the commands:
```
MariaDB [(none)]> CREATE DATABASE glance;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'Cisco.123';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'Cisco.123';
Query OK, 0 rows affected (0.01 sec)
```
Now set glance up
```
openstack user create --domain default --password-prompt glance
openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image service" image
openstack endpoint create --region RegionOne image public http://controller01:9292
openstack endpoint create --region RegionOne image internal http://controller01:9292
openstack endpoint create --region RegionOne image admin http://controller01:9292 
```
Next we install the glance packages
```
sudo apt-get install -y glance python-glanceclient
```
When configuring I first laid out a partition when I installed Linux that was called
```
/openstack
```
I created a directory structure:
```
mkdir -p /openstack/glance
chown -R glance:glance /openstack/glance
```
To store all the images. 

Next we modify the ```/etc/glance/glance-api.conf``` file to look like the following:
```
[DEFAULT]
notification_driver = noop
verbose = True

[database]
sqlite_db = /var/lib/glance/glance.sqlite
connection = mysql+pymysql://glance:Cisco.123@controller01/glance
backend = sqlalchemy

[glance_store]
default_store = file
filesystem_store_datadir = /openstack/glance/images/

[image_format]
[keystone_authtoken]
auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = glance
password = Cisco.123
[matchmaker_redis]
[matchmaker_ring]
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_qpid]
[oslo_messaging_rabbit]
[oslo_policy]
[paste_deploy]
flavor = keystone
[store_type_location_strategy]
[task]
[taskflow_executor]
```
Then we edit ```/etc/glance/glance-registry.conf``` to look like
```
[DEFAULT]
notification_driver = noop
verbose = True
[database]
connection = mysql+pymysql://glance:Cisco.123@controller01/glance
sqlite_db = /var/lib/glance/glance.sqlite
backend = sqlalchemy
[glance_store]
[keystone_authtoken]
auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = glance
password = Cisco.123

[matchmaker_redis]
[matchmaker_ring]
[oslo_messaging_amqp]
[oslo_messaging_qpid]
[oslo_messaging_rabbit]
[oslo_policy]
[paste_deploy]
flavor = keystone
```
The files are nearly identical!

As the root user run: 
```
su -s /bin/sh -c "glance-manage db_sync" glance
```
This will get the database ready for glance!  Now start the service:
```
service glance-registry restart
service glance-api restart
rm -f /var/lib/glance/glance.sqlite
```

Now test it to make sure it works: 
```
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```

Now add the image to glance to ensure it works:
```
glance image-create --name "cirros" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --visibility public --progress
```

### Nova Controller Installation
Update the database so we can install Nova goodness: 

```
mysql -p 

MariaDB [(none)]> create database nova;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'Cisco.123';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'Cisco.123';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> quit
```

Now add the nova service information to keystone
```
openstack user create --domain default --password-prompt nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute
openstack endpoint create --region RegionOne compute public http://controller01:8774/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute internal http://controller01:8774/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute admin http://controller01:8774/v2/%\(tenant_id\)s
```

Installing Nova is more packages:
```
apt-get install -y nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy nova-scheduler python-novaclient
```
And then modifying the configuration file.  The ```/etc/nova/nova.conf``` file looks like this:
```
[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/var/lock/nova
force_dhcp_release=True
libvirt_use_virtio_for_bridges=True
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
enabled_apis=ec2,osapi_compute,metadata
rpc_backend = rabbit
auth_strategy = keystone
my_ip = 192.168.2.210
network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.NeutronLinuxBridgeInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
enabled_apis=osapi_compute,metadata

[database]
connection = mysql+pymysql://nova:Cisco.123@controller01/nova

[keystone_authtoken]
auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = nova
password = Cisco.123

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[oslo_messaging]
rabbit_host = controller01
rabbit_userid = openstack
rabbit_password = Cisco.123

[vnc]
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip

[glance]
host = controller01

[neutron]
url = http://controller01:9696
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
region_name = RegionOne
project_name = service
username = neutron
password = Cisco.123

service_metadata_proxy = True
metadata_proxy_shared_secret = 6a5e0722b889dea9f8b1

[cinder]
os_region_name = RegionOne 
```
Now put all the info in the datbase: 
```
su -s /bin/sh -c "nova-manage db sync" nova
```
Then restart all the services:
```
service nova-api restart
service nova-cert restart
service nova-consoleauth restart
service nova-scheduler restart
service nova-conductor restart
service nova-novncproxy restart
```
finally, remove the sqlite file:
```
rm -f /var/lib/nova/nova.sqlite
```

Testing to make sure it all works: 
```
nova list
+----+------+--------+------------+-------------+----------+
| ID | Name | Status | Task State | Power State | Networks |
+----+------+--------+------------+-------------+----------+
+----+------+--------+------------+-------------+----------+
```

## Installing the Compute Nodes
Our compute nodes were auto installed with xCAT (old habits die
hard.).  All of these operations we did as the root user.
We will be adding a ton of post installation scripts but
for now here's what we did: 

### Setup remote user

The ```vallard``` user is added for administration:
```
useradd -m -G sudo -p $(openssl passwd -1 Cisco.123) -s /bin/bash vallard
```

Add this user to the ```/etc/sudoers``` file so that commands can run without
authentication.  We did this for the controller node.  
We append the following:
```
vallard ALL=(ALL) NOPASSWD:ALL
```

Now make sure we can ssh into this node without password authentication from the controller node.  From the __Controller Node__ run the following: 
```
cd ~/.ssh
cat id_rsa.pub >> authorized_keys
```
You shoud be able to log into yourself with out having to enter
a password:
```
ssh localhost
```
Now copy this directory to the remote host: 
```
scp -r ~/.ssh lhv01:/home/vallard
```
You'll have to enter the password once, but now you should be able
to ssh into this node without any authentication. 

### Compute node networking

Now we should statically assign the management network. 
This is again done by editing /etc/network/interfaces: 
```
auto eth0
iface eth0 inet static
  address 192.168.2.213
  netmask 255.255.255.0 
  network 192.168.2.0
  gateway 192.168.2.210
```
restarting the interface can make sure this is configured correctly:
```
ifdown eth0; ifup eth0
```

We need IP forwarding to work from the host so that we can get this 
system to access the internet. On the ```controller01``` machine
edit /etc/sysctl.conf and make sure the line is not commented out:
```
net.ipv4.ip_forward=1
```
and add the lines: 
```
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.all.rp_filter=0
```
Then run: 
```
sysctl -p
```

I also added to the ```/etc/rc.local``` file (before the ```exit 0```)

the following: 
```
iptables -t nat -A POSTROUTING -o p1p1 -j MASQUERADE
iptables -A FORWARD -i bond0 -j ACCEPT
```
Running those commands on the prompt enables ip forwarding.  Now 
my node can access the internet!

### /etc/hosts

You should ensure that all compute nodes and the controller nodes share the same ```/etc/hosts``` file. 
This will ensure name resolution can happen without worrying about an external DNS. 

### Update Compute node

```
apt-get -y update
apt-get -y upgrade
apt-get install -y software-properties-common
add-apt-repository -y cloud-archive:liberty
```

### Sysctl on the compute node

Edit ```/etc/sysctl.conf``` and add: 
```
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.all.rp_filter=0
```
then run 
```
sysctl -p
```
### NTP on the compute node

It's important that all nodes in the cluster be syncronized 
to the same clock.  
```
apt-get install -y chrony
```
edit ```/etc/chrony/chrony.conf``` to look like: 
```
server controller01 iburst
keyfile /etc/chrony/chrony.keys
commandkey 1
driftfile /var/lib/chrony/chrony.drift
log tracking measurements statistics
logdir /var/log/chrony
maxupdateskew 100.0
dumponexit
dumpdir /var/lib/chrony
local stratum 10
allow 10/8
allow 192.168/16
allow 172.16/12
logchange 0.5
rtconutc
```

The only difference with this configuration is that it points to the 
controller node instead of an external service. 

```
service chrony stop
ntpdate -bs controller01
hwclock -w
service chrony start
```

### Install Nova on the compute node

```
apt-get install -y nova-compute sysfsutils vim
apt-get install -y neutron-plugin-linuxbridge-agent
```

Edit the /etc/nova/nova.conf file to look as follows:
```
[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/var/lock/nova
force_dhcp_release=True
libvirt_use_virtio_for_bridges=True
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
enabled_apis=ec2,osapi_compute,metadata

auth_strategy = keystone
rpc_backend = rabbit
my_ip = 192.168.2.213
network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.NeutronLinuxBridgeInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[neutron]
url = http://controller01:9696
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
region_name = RegionOne
project_name = service
username = neutron
password = Cisco.123


[glance]
host = controller01

[keystone_authtoken]
auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = nova
password = Cisco.123


[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[oslo_messaging_rabbit]
rabbit_host = controller01
rabbit_userid = openstack
rabbit_password = Cisco.123

[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://controller01:6080/vnc_auto.html
```


Since we're using neutron networking we need to 
configure ```/etc/neutron/neutron.conf```.  It looks as follows:
```
[DEFAULT]
core_plugin = ml2
rpc_backend = rabbit
auth_strategy = keystone

[matchmaker_redis]
[matchmaker_ring]
[quotas]
[agent]
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf
[keystone_authtoken]
auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = Cisco.123
[database]
connection = sqlite:////var/lib/neutron/neutron.sqlite
[nova]
[oslo_concurrency]
lock_path = $state_path/lock
[oslo_policy]
[oslo_messaging_amqp]
[oslo_messaging_qpid]
[oslo_messaging_rabbit]
rabbit_host = controller01
rabbit_userid = openstack
rabbit_password = Cisco.123
[qos]
```

We also edit ```/etc/neutron/plugins/ml2/linuxbridge_agent.ini``` to look like:
```
[linux_bridge]
physical_interface_mappings = public:eth0
[vxlan]
enable_vxlan = False
[agent]
prevent_arp_spoofing = True
[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

Restart the service:
```
service nova-compute restart
service neutron-plugin-linuxbridge-agent restart
rm -f /var/lib/nova/nova.sqlite
```

Check from the ```controller01``` node that it works:
```
nova service-list
+----+------------------+--------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host         | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+--------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-cert        | controller01 | internal | enabled | up    | 2015-11-30T20:55:53.000000 | -               |
| 2  | nova-consoleauth | controller01 | internal | enabled | up    | 2015-11-30T20:55:52.000000 | -               |
| 3  | nova-scheduler   | controller01 | internal | enabled | up    | 2015-11-30T20:55:45.000000 | -               |
| 4  | nova-conductor   | controller01 | internal | enabled | up    | 2015-11-30T20:55:52.000000 | -               |
| 5  | nova-compute     | lhv01        | nova     | enabled | up    | 2015-11-30T20:55:46.000000 | -               |
+----+------------------+--------------+----------+---------+-------+----------------------------+-----------------+
```

## Neutron Installation
Here we will install Neutron on the controller01 node.

### MySQL Databse
```
mysql -p 
```
and then we run the following SQL commands: 
```
MariaDB [(none)]> create database neutron
    -> ;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> GRANT all privileges on neutron.* to 'neutron'@'localhost' identified by 'Cisco.123';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> GRANT all privileges on neutron.* to 'neutron'@'%' identified by 'Cisco.123';Query OK, 0 rows affected (0.00 sec)
```
Now we configure the Neutron services
```
openstack user create --domain default --password-prompt neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network
openstack endpoint create --region RegionOne network public http://controller01:9696
openstack endpoint create --region RegionOne network internal http://controller01:9696
openstack endpoint create --region RegionOne network admin http://controller01:9696

```

### Provider network configuration
We install the following on the controller node: 
```
apt-get install -y neutron-server neutron-plugin-ml2 neutron-plugin-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent python-neutronclient
```
The /etc/neutron/neutron.conf file looks as follows:
```
[DEFAULT]
verbose = True
core_plugin = ml2
service_plugins =
rpc_backend = rabbit
auth_strategy = keystone
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
nova_url = http://controller01:8774/v2

[matchmaker_redis]

[matchmaker_ring]
[quotas]
[agent]
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf

[keystone_authtoken]

auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = Cisco.123

[database]
connection = mysql+pymysql://neutron:Cisco.123@controller01/neutron

[nova]
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
region_name = RegionOne
project_name = service
username = nova
password = Cisco.123

[oslo_concurrency]
lock_path = $state_path/lock
[oslo_policy]
[oslo_messaging_amqp]
[oslo_messaging_qpid]
[oslo_messaging_rabbit]
rabbit_host = controller01
rabbit_userid = openstack
rabbit_password = Cisco.123
[qos]
```
We then edit ```/etc/neutron/plugins/ml2/ml2_conf.ini``` to look like: 

```
[ml2]
type_drivers = flat,vlan
tenant_network_types =
mechanism_drivers = linuxbridge
extension_drivers = port_security

[ml2_type_flat]
flat_networks = public

[ml2_type_vlan]
# network_vlan_ranges = public
[ml2_type_gre]
[ml2_type_vxlan]
[ml2_type_geneve]
[securitygroup]
enable_ipset = True
```
The ```/etc/neutron/plubins/ml2/linuxbridge_agent.ini``` looks like: 
```
[linux_bridge]
physical_interface_mappings = public:bond0
[vxlan]
enable_vxlan = False
[agent]
prevent_arp_spoofing = True
[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
and the ```/etc/neutron/dhcp_agent.ini``` file looks like: 
```
DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
verbose = True
[AGENT]

```
The ```/etc/metadata_agent.ini``` looks as follows:
```
[DEFAULT]
auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_region = RegionOne
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = Cisco.123
nova_metadata_ip = controller01
metadata_proxy_shared_secret = 6a5e0722b889dea9f8b1
[AGENT]
```

You'll notice that the ```metadata_proxy_shared_secret``` value is just a randomly 
generated string.  

We're finally ready to populate the neutron database: 
```
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```
Once this is successful, restart the nova service:
```
service nova-api restart
service neutron-server restart
service neutron-plugin-linuxbridge-agent restart
service neutron-dhcp-agent restart
service neutron-metadata-agent restart
#service neutron-l3-agent restart
```
and remove the extra sqlite file:
```
rm -f /var/lib/neutron/neutron.sqlite
```

## Install the Dashboard
```
apt-get install -y openstack-dashboard
```

To avoid bugs since we're using the provider network, you need to change the dashboard settings
in ```/etc/openstack-dashboard/local_settings.py```
To look like: 
```
OPENSTACK_NEUTRON_NETWORK = {
#    'enable_router': True,
#    'enable_quotas': True,
#    'enable_ipv6': True,
    'enable_router': False,
    'enable_quotas': False,
    'enable_ipv6': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
#    'enable_lb': True,
#    'enable_firewall': True,
#    'enable_vpn': True,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': True,
    ...
```
}

## Creating Networks

We create some networks on the controller node: 
```
neutron net-create public --shared --provider:physical_network public --provider:network_type flat
neutron subnet-create public 192.168.2.0/24 --name public --allocation-pool start=192.168.2.230,end=192.168.2.254 --dns-nameserver 10.93.234.38 --gateway 192.168.2.210
```

Another network for internal
```
neutron net-create cisco-1 --share --provider:physical_network public --provider:network_type vlan --provider:segmentation_id 1
neutron subnet-create cisco-1 10.93.234.0/24 --name cisco --allocation-pool start=10.93.234.80,end=10.93.234.90 --dns-nameserver 10.93.234.38 --gateway 10.93.234.1

Launching an instance:
```
nova boot --flavor m1.tiny --image cirros --nic net-id=public --security-group default --key-name lucky test


## Installing Cinder
Now that we have the ability to create ephemeral nodes, we'll add some block storage so that things can persist. 
First we have to install cinder on the controller node.
```
mysql -p
MariaDB [(none)]> create database cinder;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> grant all privileges on cinder.* to 'cinder'@'localhost' identified by 'Cisco.123';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> grant all privileges on cinder.* to 'cinder'@'%' identified by 'Cisco.123';
Query OK, 0 rows affected (0.00 sec)
```
Now create the user: 
```
openstack user create --domain default --password-prompt cinder
# Enter a password
openstack role add --project service --user cinder admin
openstack service create --name cinder --description "OpenStack Block Storage" volume
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
openstack endpoint create --region RegionOne volume public http://controller01:8776/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne volume admin http://controller01:8776/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne volume internal http://controller01:8776/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev2 internal http://controller01:8776/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev2 public http://controller01:8776/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev2 admin http://controller01:8776/v2/%\(tenant_id\)s
```
Now we install the components
```
apt-get install -y cinder-api cinder-scheduler python-cinderclient
```
We now edit the ```/etc/cinder/cinder.conf``` file.  This looks like: 
```
[DEFAULT]
rootwrap_config = /etc/cinder/rootwrap.conf
api_paste_confg = /etc/cinder/api-paste.ini
iscsi_helper = tgtadm
volume_name_template = volume-%s
volume_group = cinder-volumes
verbose = True
auth_strategy = keystone
state_path = /var/lib/cinder
lock_path = /var/lock/cinder
volumes_dir = /openstack/cinder/volumes
my_ip = 192.168.2.210
rpc_backend = rabbit

[database]
connection = mysql+pymysql://cinder:Cisco.123@controller01/cinder

[keystone_authtoken]
auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = cinder
password = Cisco.123

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp

[oslo_messaging_rabbit]
rabbit_host = controller01
rabbit_userid = openstack
rabbit_password = Cisco.123
```
Now make sure you create the ```cinder directory```
```
mkdir /openstack/cinder
chown cinder:cinder /openstack/cinder
```
Now migrate it over: 
```
su -s /bin/sh -c "cinder-manage db sync" cinder
```
Then restart the services
```
service nova-api restart
service cinder-scheduler restart
service cinder-api restart
rm -f /var/lib/cinder/cinder.sqlite
```

## Storage Node
The storage node needs to be updated with the latest level of Ubuntu and prepared like the other
compute nodes.  You can also add this node into the cluster as a compute node or just leave
it as a dedicated storage node.  

Our storage node has several drives on it. 
```
apt-get install cinder-volume python-mysqldb
```
Now create some volumes
```
pvcreate /dev/sdb
pvcreate /dev/sdc
pvcreate /dev/sdd
pvcreate /dev/sde
vgcreate cinder-volumes /dev/sdb /dev/sdc /dev/sdd /dev/sde
```
Edit ```/etc/lvm/lvm.conf``` and add the filter line:
```
filter = [ "a/sda/", "a/sdb/", "a/sdc/", "a/sdd/", "a/sde/", "r/.*/"]
```

Confinguring the ```/etc/cinder/cinder.conf``` file looks almost like the controller node
but with a few differences:
```
[DEFAULT]
rootwrap_config = /etc/cinder/rootwrap.conf
api_paste_confg = /etc/cinder/api-paste.ini
iscsi_helper = tgtadm
volume_name_template = volume-%s
volume_group = cinder-volumes
verbose = True
auth_strategy = keystone
state_path = /var/lib/cinder
lock_path = /var/lock/cinder
volumes_dir = /openstack/cinder/volumes
my_ip = 192.168.2.212
rpc_backend = rabbit
enabled_backends = lvm
glance_host = controller01

[database]
connection = mysql+pymysql://cinder:Cisco.123@controller01/cinder

[keystone_authtoken]
auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = cinder
password = Cisco.123

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp

[oslo_messaging_rabbit]
rabbit_host = controller01
rabbit_userid = openstack
rabbit_password = Cisco.123

[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
iscsi_protocol = iscsi
iscsi_helper = tgtadm
```
Now restart the services and pray you didn't make a type-o:
```
service tgt restart
service cinder-volume restart
rm -rf /var/lib/cinder/cinder.sqlite 
```

You should now be able to see the volume services running. From the controller node
run:
```
cinder service-list
+------------------+--------------+------+---------+-------+----------------------------+-----------------+
|      Binary      |     Host     | Zone |  Status | State |         Updated_at         | Disabled Reason |
+------------------+--------------+------+---------+-------+----------------------------+-----------------+
| cinder-scheduler | controller01 | nova | enabled |   up  | 2015-12-07T06:48:56.000000 |        -        |
|  cinder-volume   |  data02@lvm  | nova | enabled |   up  | 2015-12-07T06:48:55.000000 |        -        |
+------------------+--------------+------+---------+-------+----------------------------+-----------------+
```
If there are problems, please make sure your clocks are in sync.  
