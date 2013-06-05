 
0. What is it ?
This guide includes steps to create multi-node HA openstack cloud, with Ceph as Glance and Cinder backend, Swift as object store, openvswitch as quantum plugin.



1. Requirements
4 type of nodes: Controller, Network, Compute and Swift

Architecture: 



IP addresses and disks allocation:



Hostname HW model	Role	em-1(external)	em-2(mgmt)	em-3(vm traffic)	em-4(storage)	iDRAC	CPU	Memory	Disk	RAID setup
R710-1	R710	Swift	198.154.120.134	10.10.10.1	

198.154.120.156	2 x QUAD CORE E5540 2.53GHz 	64 GB	6 x 2TB SAS	2 disks in RAID 1 for OS, 4 disks in 4 x RAID 0, for swift storage
R710-2	R710	Controller_bak	198.154.120.133	10.10.10.2	
10.30.30.2	198.154.120.155	2 x QUAD CORE E5540 2.53GHz 	64 GB	6 x 2TB SAS	2 disks in RAID 1 for OS, 4 disks in  4 x RAID 0 for ceph
R710-3	R710	Controller	198.154.120.132	10.10.10.3	
10.30.30.3	198.154.120.154	2 x QUAD CORE E5540 2.53GHz 	64 GB	6 x 2TB SAS	2 disks in RAID 1 for OS, 4 disks in 4 x RAID 0 for ceph
R610-4	R610	Network	198.154.120.131	10.10.10.4	10.20.20.4	10.30.30.4	198.154.120.153	2 x QUAD CORE L5520 2.26GHz	24 GB	2 x 64GB SSD	RAID 1
R610-5	R610	Network_bak	198.154.120.130	10.10.10.5	10.20.20.5	10.30.30.5	198.154.120.152	2 x QUAD CORE L5520 2.26GHz	24 GB	2 x 64GB SSD	RAID 1
R710L-6	R710L	Compute	198.154.120.135	10.10.10.6	10.20.20.6	10.30.30.6	198.154.120.151	2 x QUAD CORE E5540 2.53GHz 	192 GB	6 x 3TB SAS	RAID 1+0
R710-7	R710	Compute	198.154.120.136	10.10.10.7	10.20.20.7	10.30.30.7	198.154.120.150	2 x QUAD CORE E5540 2.53GHz 	64 GB	6 x 2TB SAS	RAID 1+0
R710-8	R710	Compute	198.154.120.137	10.10.10.8	10.20.20.8	10.30.30.8	198.154.120.149	2 x QUAD CORE E5540 2.53GHz 	64 GB	6 x 2TB SAS	RAID 1+0
R710L-9	R710L	Compute	198.154.120.138	10.10.10.9	10.20.20.9	10.30.30.9	198.154.120.148	2 x QUAD CORE E5540 2.53GHz 	192 GB	6 x 3TB SAS	RAID 1+0


2. Network Node
2.1. Preparing the Node
Install Ubuntu 13.04
Add ceph hosts entries to /etc/hosts
10.30.30.3 R710-3
10.30.30.2 R710-2
10.30.30.5 R610-5
Update your system
apt-get update -y
apt-get upgrade -y
apt-get dist-upgrade -y
Setup ntp service
apt-get install -y ntp
Add controllers as ntp servers, then restart ntp service. 
echo "server 10.10.10.3" >> /etc/ntp.conf
echo "server 10.10.10.2" >> /etc/ntp.conf
service ntp restart
Install other services
apt-get install -y vlan bridge-utils
Enable IP_Forwarding
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sysctl -p


2.2. Networking
Edit /etc/network/interfaces, following example is for R610-5 node, change IPs accordingly for R610-4 node. Also R610-4 does not need em4 storage IP since it has no ceph component running
auto em1
iface em1 inet static
        address 198.154.120.131
        netmask 255.255.255.224
        gateway 198.154.120.129
        dns-nameservers 8.8.8.8

 
#Openstack management
auto em2
iface em2 inet static
        address 10.10.10.5
        netmask 255.255.255.0
#VM traffic
auto em3
iface em3 inet static
        address 10.20.20.5
        netmask 255.255.255.0

 
#Storage network for ceph
auto em4
iface em4 inet static
        address 10.30.30.5
        netmask 255.255.255.0
Restart networking service
service networking restart
2.3. OpenVSwitch (Part1)
Install the openVSwitch:
apt-get install -y openvswitch-switch openvswitch-datapath-dkms
Create the bridges:
#br-int will be used for VM integration 
ovs-vsctl add-br br-int

#br-ex is used to make to VM accessible from the external network
ovs-vsctl add-br br-ex

#br-em3 is used to establish VM internal traffic
ovs-vsctl add-br br-em3
ovs-vsctl add-port br-em3 em3


2.4. Quantum
Install the Quantum openvswitch agent, l3 agent, dhcp agent and metadata-agent
apt-get -y install quantum-plugin-openvswitch-agent quantum-dhcp-agent quantum-l3-agent quantum-metadata-agent
Edit /etc/quantum/quantum.conf
[DEFAULT]
auth_strategy = keystone
rabbit_host = 10.10.10.101
rabbit_password=manager175

[keystone_authtoken]
auth_host = 10.10.10.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = quantum
admin_password = manager175
signing_dir = /var/lib/quantum/keystone-signing
Edit  /etc/quantum/api-paste.ini
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 10.10.10.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = quantum
admin_password = manager175
Edit the OVS plugin configuration file /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini with::
[DATABASE]
sql_connection = mysql://quantum:manager175@10.10.10.100/quantum
 
[OVS]
tenant_network_type = vlan
network_vlan_ranges = physnet1:1000:1100
integration_bridge = br-int
bridge_mappings = physnet1:br-em3
 
[SECURITYGROUP]
firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
Update /etc/quantum/metadata_agent.ini with:
[DEFAULT]
auth_url = http://10.10.10.200:35357/v2.0
auth_region = RegionOne
admin_tenant_name = service
admin_user = quantum
admin_password = manager175

nova_metadata_ip = 10.10.10.200
nova_metadata_port = 8775
metadata_proxy_shared_secret = howard
Edit /etc/sudoers to give quantum user full access like:
Defaults:quantum !requiretty
quantum ALL=NOPASSWD: ALL
Restart all the services:
cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done


2.5. OpenVSwitch (Part2)
Edit the em1in /etc/network/interfaces to become like this:
 

auto em1
iface em1 inet manual
up ifconfig $IFACE 0.0.0.0 up
up ip link set $IFACE promisc on
down ip link set $IFACE promisc off
down ifconfig $IFACE down
Add em1 to br-ex
#Internet connectivity will be lost after this step but this won't affect OpenStack's work
ovs-vsctl add-port br-ex em1
Add external IP to br-ex to get internet access back, add following to /etc/network/interfaces
auto br-ex
iface br-ex inet static
        address 198.154.120.130
        netmask 255.255.255.224
        gateway 198.154.120.129
        dns-nameservers 8.8.8.8
Restart networking and quantum services
service networking restart
cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done


2.6. HAProxy
Install package on both network node
apt-get install -y keepalived haproxy
Disable auto-start by editing /etc/default/haproxy
ENABLED=0
Edit /etc/haproxy/haproxy.cfg, the content of the file is same on both network node
global
        log 127.0.0.1   local0
        log 127.0.0.1   local1 notice
        #log loghost    local0 info
        maxconn 4096
        #chroot /usr/share/haproxy
        user haproxy
        group haproxy
        daemon
        #debug
        #quiet

defaults
 log  global
 maxconn  8000
 option  redispatch
 retries  3
 timeout  http-request 10s
 timeout  queue 1m
 timeout  connect 10s
 timeout  client 1m
 timeout  server 1m
 timeout  check 10s

listen dashboard_cluster
 bind 10.10.10.200:80
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server R710-3 10.10.10.3:80 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:80 check inter 2000 rise 2 fall 5
listen dashboard_cluster_internet
 bind 198.154.120.143:80
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server R710-3 10.10.10.3:80 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:80 check inter 2000 rise 2 fall 5
listen glance_api_cluster
 bind 10.10.10.200:9292
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server R710-3 10.10.10.3:9292 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:9292 check inter 2000 rise 2 fall 5
listen glance_api_internet_cluster
 bind 198.154.120.143:9292
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server R710-3 10.10.10.3:9292 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:9292 check inter 2000 rise 2 fall 5
listen glance_registry_cluster
 bind 10.10.10.200:9191
 balance  source
 option  tcpka
 option  tcplog
 server R710-3 10.10.10.3:9191 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:9191 check inter 2000 rise 2 fall 5

listen glance_registry_internet_cluster
 bind 198.154.120.143:9191
 balance  source
 option  tcpka
 option  tcplog
 server R710-3 10.10.10.3:9191 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:9191 check inter 2000 rise 2 fall 5
listen keystone_admin_cluster
 bind 10.10.10.200:35357
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server R710-3 10.10.10.3:35357 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:35357 check inter 2000 rise 2 fall 5
 server control03 192.168.220.43:35357 check inter 2000 rise 2 fall 5
listen keystone_internal_cluster
 bind 10.10.10.200:5000
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server R710-3 10.10.10.3:5000 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:5000 check inter 2000 rise 2 fall 5
listen keystone_public_cluster
 bind 198.154.120.143:5000
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server R710-3 10.10.10.3:5000 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:5000 check inter 2000 rise 2 fall 5
listen memcached_cluster
 bind 10.10.10.200:11211
 balance  source
 option  tcpka
 option  tcplog
 server R710-3 10.10.10.3:11211 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:11211 check inter 2000 rise 2 fall 5
listen nova_compute_api1_cluster
 bind 10.10.10.200:8773
 balance  source
 option  tcpka
 option  tcplog
 server R710-3 10.10.10.3:8773 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:8773 check inter 2000 rise 2 fall 5

listen nova_compute_api1_internet_cluster
 bind 198.154.120.143:8773
 balance  source
 option  tcpka
 option  tcplog
 server R710-3 10.10.10.3:8773 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:8773 check inter 2000 rise 2 fall 5

listen nova_compute_api2_cluster
 bind 10.10.10.200:8774
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server R710-3 10.10.10.3:8774 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:8774 check inter 2000 rise 2 fall 5
listen nova_compute_api2_internet_cluster
 bind 198.154.120.143:8774
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server R710-3 10.10.10.3:8774 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:8774 check inter 2000 rise 2 fall 5
listen nova_compute_api3_cluster
 bind 10.10.10.200:8775
 balance  source
 option  tcpka
 option  tcplog
 server R710-3 10.10.10.3:8775 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:8775 check inter 2000 rise 2 fall 5
listen nova_compute_api3_internet_cluster
 bind 198.154.120.143:8775
 balance  source
 option  tcpka
 option  tcplog
 server R710-3 10.10.10.3:8775 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:8775 check inter 2000 rise 2 fall 5
listen cinder_api_cluster
 bind 10.10.10.200:8776
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server R710-3 10.10.10.3:8776 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:8776 check inter 2000 rise 2 fall 5
listen cinder_api_internet_cluster
 bind 198.154.120.143:8776
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server R710-3 10.10.10.3:8776 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:8776 check inter 2000 rise 2 fall 5

listen novnc_cluster
 bind 10.10.10.200:6080
 balance  source
 option  tcpka
 option  tcplog
 server R710-3 10.10.10.3:6080 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:6080 check inter 2000 rise 2 fall 5
listen novnc__internet_cluster
 bind 198.154.120.143:6080
 balance  source
 option  tcpka
 option  tcplog
 server R710-3 10.10.10.3:6080 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:6080 check inter 2000 rise 2 fall 5
listen quantum_api_cluster
 bind 10.10.10.200:9696
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server R710-3 10.10.10.3:9696 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:9696 check inter 2000 rise 2 fall 5
listen quantum_api_internet_cluster
 bind 198.154.120.143:9696
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server R710-3 10.10.10.3:9696 check inter 2000 rise 2 fall 5
 server R710-2 10.10.10.2:9696 check inter 2000 rise 2 fall 5
Stop haproxy if it's running, let pacemaker to manage it later
service haproxy stop


2.7. Corosync and Pacemaker
Install packages
apt-get install pacemaker corosync
Generate Corosync keys on one node (R610-5)
corosync-keygen
 
#Copy generated key to another node
scp /etc/corosync/authkey R610-4:/etc/corosync/authkey
Edit /etc/corosync/corosync.conf on both node, replace "bindnetaddr" with real node em2 and em4 IP address
rrp_mode: active
        interface {
                # The following values need to be set based on your environment 
                ringnumber: 0
                bindnetaddr: 10.10.10.5
                mcastaddr: 226.94.1.3
                mcastport: 5405
        }
        interface {
                # The following values need to be set based on your environment 
                ringnumber: 1
                bindnetaddr: 10.30.30.5
                mcastaddr: 226.94.1.4
                mcastport: 5405
        }
Enable autostart, then start Corosync service
#Edit /etc/default/corosync
START=yes
 
service corosync start
Check Corosync status
crm_mon
Download HAproxy OCF script
cd /usr/lib/ocf/resource.d/heartbeat
wget  https://raw.github.com/russki/cluster-agents/master/haproxy
chmod 755 haproxy 
Configure cluster resources for mysql
 

crm configure
 
property stonith-enabled=false
property no-quorum-policy=ignore
rsc_defaults resource-stickiness=100
rsc_defaults failure-timeout=0
rsc_defaults migration-threshold=10
property pe-warn-series-max="1000"
property pe-input-series-max="1000"
property pe-error-series-max="1000"
property cluster-recheck-interval="5min"
 
primitive vip-mgmt ocf:heartbeat:IPaddr2 params ip=10.10.10.200 cidr_netmask=24 op monitor interval=5s 
primitive vip-internet ocf:heartbeat:IPaddr2 params ip=198.154.120.143 cidr_netmask=24 op monitor interval=5s
primitive haproxy  ocf:heartbeat:haproxy params conffile="/etc/haproxy/haproxy.cfg" op monitor interval="5s"
colocation haproxy-with-vips INFINITY: haproxy vip-mgmt vip-internet
order haproxy-after-IP mandatory: vip-mgmt vip-internet   haproxy 


verify
commit
 
 #Check pacemaker resource running
 crm_mon -1


2.8. Ceph (on R610-5 node only)
We use R610-5 node as 3rd Ceph monitor node

Install Ceph repository and package
wget -q -O- 'https://ceph.com/git/?p=ceph.git;a=blob_plain;f=keys/release.asc' | sudo apt-key add -
echo deb http://ceph.com/debian-cuttlefish/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
apt-get update -y

apt-get install ceph
Create ceph-c monitor directory
#R610-5
mkdir /var/lib/ceph/mon/ceph-c
 







3. Controller Nodes
3.1. Preparing the nodes
Install Ubuntu 13.04
During disk partitioning selection, leave around 200GB space in the volume group, we need some space for Mysql and Rabbitmq DRBD resources.

Add ceph hosts entries to /etc/hosts
10.30.30.3 R710-3
10.30.30.2 R710-2
10.30.30.5 R610-5
Update your system
apt-get update -y
apt-get upgrade -y
apt-get dist-upgrade -y
Setup ntp service
apt-get install -y ntp
Add another controller as ntp server, then restart ntp service. 
#use 10.10.10.3 for controller_bak node
echo "server 10.10.10.2" >> /etc/ntp.conf
service ntp restart
Install other services
apt-get install -y vlan bridge-utils
Enable IP_Forwarding
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sysctl -p
3.2. Networking
Edit /etc/network/interfaces, following example is for R710-3 node, change IPs accordingly for R710-2 node
auto em1
iface em1 inet static
        address 198.154.120.132
        netmask 255.255.255.224
        gateway 198.154.120.129
        dns-nameservers 8.8.8.8

#Openstack management
auto em2
iface em2 inet static
        address 10.10.10.3
        netmask 255.255.255.0

#Storage network for ceph
auto em4
iface em4 inet static
        address 10.30.30.3
        netmask 255.255.255.0
Restart networking service
service networking restart
3.3. MySQL
Install MySQL
apt-get install -y mysql-server python-mysqldb


Make sure on both controller node, mysql has same UID and GID, if they are different, change them to the same.


Configure mysql to accept all incoming requests 
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
Disable mysql auto-start by editting /etc/init/mysql.conf
#Comment following line:
 
#start on runlevel [2345]
Stop mysql to let pacemaker to manage
service mysql stop


3.4. RabbitMQ
Install RabbitMQ
apt-get install rabbitmq-server


Make sure on both controller node, rabbitmq has same UID and GID, if they are different, change them to the same.
Disable RabbitMQ server auto-restart by editing 
update-rc.d -f  rabbitmq-server remove
Stop mysql to let pacemaker to manage
service rabbitmq-server stop


3.5. DRBD
Install packages
apt-get install drbd8-utils xfsprogs
Disable DRBD auto-start
update-rc.d -f drbd remove
Prepare partitions, create a 100G LV for mysql, 10G LV for rabbitmq
lvcreate lvcreate R710-3-vg -n drbd0 -L 100G
lvcreate lvcreate R710-3-vg -n drbd1 -L 10G
 
#Replace VG name to R710-2-vg on node R710-2
Load  DRBD module
modprobe drbd
 
#Add drbd to /etc/modules
echo "drbd" >> /etc/modules
Create mysql DRBD resource file /etc/drbd.d/mysql.res
resource drbd-mysql {
        device /dev/drbd0;
        meta-disk internal;
        on R710-2 {
                address 10.30.30.2:7788;
                disk /dev/mapper/R710--2--vg-drbd0;
        }
        on R710-3 {
                address 10.30.30.3:7788;
                disk /dev/mapper/R710--3--vg-drbd0;
        }
        syncer {
                rate 40M;
        }
		net {
    	after-sb-0pri discard-zero-changes;
    	after-sb-1pri discard-secondary;
 		}
}
Create rabbitmq DRBD resource file /etc/drbd.d/rabbitmq.res
resource drbd-rabbitmq{
        device /dev/drbd1;
        meta-disk internal;
        on R710-2 {
                address 10.30.30.2:7789;
                disk /dev/mapper/R710--2--vg-drbd1;
        }
        on R710-3 {
                address 10.30.30.3:7789;
                disk /dev/mapper/R710--3--vg-drbd1;
        }
        syncer {
                rate 40M;
        }
		net {
   		after-sb-0pri discard-zero-changes;
    	after-sb-1pri discard-secondary;
 		}
}
After did configuration above on both nodes, bring up DRBD resources
#[Both node]Once the configuration file is saved, we can test its correctness as follows:
drbdadm dump drbd-mysql
drbdadm dump drbd-rabbitmq
 
#[Both node]Create the metadata as follows:
drbdadm create-md drbd-mysql
drbdadm create-md drbd-rabbitmq
 
#[Both node]Bring resources up:
drbdadm up drbd-mysql
drbdadm up drbd-rabbitmq
 
#[Both node]Check that both DRBD nodes have made communication and we'll see that the data is inconsistent as no initial synchronization has been made. For this we do the following:
drbd-overview 

#And the result will be similar to:
  0:drbd-mysql Connected Secondary/Secondary Inconsistent/Inconsistent C r-----
Initial DRBD Synchronization
#Do this on 1st node only:
drbdadm -- --overwrite-data-of-peer primary drbd-mysql
drbdadm -- --overwrite-data-of-peer primary drbd-rabbitmq
 
#And this should show something similar to:
 0:drbd-mysql Connected Secondary/Secondary UpToDate/UpToDate C r-----
Create filesystem
#Do this on 1st node only:
mkfs -t xfs /dev/drbd0
mkfs -t xfs /dev/drbd1
Move/Copy mysql and rabbitmq files to DRBD resources
#Do following on 1st node only
 
#mysql
mkdir /mnt/mysql
chown mysql:mysql /mnt/mysql
mount /dev/drbd0 /mnt/mysql
mv /var/lib/mysql/* /mnt/mysql
umount /mnt/mysql
 
#rabbitmq
mkdir /mnt/rabbitmq
chown rabbitmq:rabbitmq /mnt/rabbitmq
scp -p /var/lib/rabbitmq/.erlang.cookie R710-2:/var/lib/rabbitmq/
ssh R710-2 chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
mount /dev/drbd1 /mnt/rabbitmq
cp -a /var/lib/rabbitmq/.erlang.cookie /mnt/rabbitmq
umount /mnt/rabbitmq
Change resources back to secondary to let pacemaker to manage
#Check output of drbd-overview, do following only after the synchronization is finished.
drbdadm secondary drbd-mysql
drbdadm secondary drbd-rabbitmq


3.6. Pacemaker and Corosync
Install packages
apt-get install pacemaker corosync
Generate Corosync keys on one node
corosync-keygen
 
#Copy generated key to another node
scp /etc/corosync/authkey R710-2:/etc/corosync/authkey
Edit /etc/corosync/corosync.conf on both node, replace "bindnetaddr" with real node em2 and em4 IP address
rrp_mode: active
        interface {
                # The following values need to be set based on your environment 
                ringnumber: 0
                bindnetaddr: 10.10.10.2
                mcastaddr: 226.94.1.1
                mcastport: 5405
        }
        interface {
                # The following values need to be set based on your environment 
                ringnumber: 1
                bindnetaddr: 10.30.30.2
                mcastaddr: 226.94.1.2
                mcastport: 5405
        }
Enable autostart, then start Corosync service
#Edit /etc/default/corosync
START=yes
 
service corosync start
Check Corosync status
crm_mon
Configure cluster resources for mysql
crm configure
 
property stonith-enabled=false
property no-quorum-policy=ignore
rsc_defaults resource-stickiness=100

primitive drbd-mysql ocf:linbit:drbd \
        params drbd_resource="drbd-mysql" \
        op monitor interval="50s" role="Master" timeout="30s" \
        op monitor interval="60s" role="Slave" timeout="30s"

primitive fs-mysql ocf:heartbeat:Filesystem \
        params device="/dev/drbd0" directory="/mnt/mysql" fstype="xfs" \
        meta target-role="Started"

primitive mysql ocf:heartbeat:mysql \
        params config="/etc/mysql/my.cnf" datadir="/mnt/mysql" binary="/usr/bin/mysqld_safe" pid="/var/run/mysqld/mysqld.pid" socket="/var/run/mysqld/mysqld.sock" log="/var/log/mysql/mysql.log" additional_parameters="--bind-address=10.10.10.100" \
        op start interval="0" timeout="120s" \
        op stop interval="0" timeout="120s" \
        op monitor interval="15s" \
        meta target-role="Started"

primitive vip-mysql ocf:heartbeat:IPaddr2 \
        params ip="10.10.10.100" cidr_netmask="24"

group g-mysql fs-mysql vip-mysql mysql

ms ms-drbd-mysql drbd-mysql \
        meta master-max="1" master-node-max="1" clone-max="2" clone-node-max="1" notify="true" is-managed="true" target-role="Started"

colocation c-fs-mysql-on-drbd inf: g-mysql ms-drbd-mysql:Master

order o-drbd-before-fs-mysql inf: ms-drbd-mysql:promote g-mysql:start


 
verify
commit
 
 
#Check pacemaker resource running
 crm_mon
Configure cluster resources for mysql
crm configure
 

primitive drbd-rabbitmq ocf:linbit:drbd \
        params drbd_resource="drbd-rabbitmq" \
        op monitor interval="50s" role="Master" timeout="30s" \
        op monitor interval="60s" role="Slave" timeout="30s"
 
primitive fs-rabbitmq ocf:heartbeat:Filesystem \
        params device="/dev/drbd1" directory="/mnt/rabbitmq" fstype="xfs" \
        meta target-role="Started"
 
primitive rabbitmq ocf:rabbitmq:rabbitmq-server \
        params mnesia_base="/mnt/rabbitmq"
 
primitive vip-rabbitmq ocf:heartbeat:IPaddr2 \
        params ip="10.10.10.101" cidr_netmask="24"
 
group g-rabbitmq fs-rabbitmq vip-rabbitmq rabbitmq
 
ms ms-drbd-rabbitmq drbd-rabbitmq \
        meta notify="true" master-max="1" master-node-max="1" clone-max="2" clone-node-max="1"
 
colocation c-fs-rabbitmq-on-drbd inf: g-rabbitmq ms-drbd-rabbitmq:Master
 
order o-drbd-before-fs-rabbitmq inf: ms-drbd-rabbitmq:promote g-rabbitmq:start
 
verify
commit
#Check pacemaker resource running
 crm_mon
Configure rabbitmq guest password
#Check rabbitmq is running on which node by:
crm_mon -1
 
#Change the password on the active rabbitmq node:
rabbitmqctl change_password guest manager175


3.7. Create Databases
Create Databases
#Connet to mysql via its VIP:
mysql -u root -p -h 10.10.10.100

#Keystone
CREATE DATABASE keystone;
GRANT ALL ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'manager175';

#Glance
CREATE DATABASE glance;
GRANT ALL ON glance.* TO 'glance'@'%' IDENTIFIED BY 'manager175';

#Quantum
CREATE DATABASE quantum;
GRANT ALL ON quantum.* TO 'quantum'@'%' IDENTIFIED BY 'manager175';

#Nova
CREATE DATABASE nova;
GRANT ALL ON nova.* TO 'nova'@'%' IDENTIFIED BY 'manager175';

#Cinder
CREATE DATABASE cinder;
GRANT ALL ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'manager175';

quit;


3.8. Ceph
2 controller nodes are Ceph monitor(MON) and storage(OSD) nodes

Install Ceph repository and package on both controller nodes
wget -q -O- 'https://ceph.com/git/?p=ceph.git;a=blob_plain;f=keys/release.asc' | sudo apt-key add -
echo deb http://ceph.com/debian-cuttlefish/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
apt-get update -y

apt-get install ceph python-ceph
Setup password free ssh connection from R710-3 to other 2 ceph nodes
#R710-3
ssh-keygen -N '' -f ~/.ssh/id_rsa
ssh-copy-id R710-2
ssh-copy-id R610-5
Prepare directories and disks on both controller nodes
#R710-3
mkdir /var/lib/ceph/osd/ceph-11
mkdir /var/lib/ceph/osd/ceph-12
mkdir /var/lib/ceph/osd/ceph-13
mkdir /var/lib/ceph/osd/ceph-14
mkdir /var/lib/ceph/mon/ceph-a
parted /dev/sdb mklabel msdos
parted /dev/sdc mklabel msdos
parted /dev/sdd mklabel msdos
parted /dev/sde mklabel msdos


#R710-2
mkdir /var/lib/ceph/osd/ceph-21
mkdir /var/lib/ceph/osd/ceph-22
mkdir /var/lib/ceph/osd/ceph-23
mkdir /var/lib/ceph/osd/ceph-24
mkdir /var/lib/ceph/mon/ceph-b
parted /dev/sdb mklabel msdos
parted /dev/sdc mklabel msdos
parted /dev/sdd mklabel msdos
parted /dev/sde mklabel msdos
Create /etc/ceph.conf on R710-3, then distribute it to other 2 nodes
[global]
        auth cluster required = cephx
        auth service required = cephx
        auth client required = cephx
[osd]
        osd journal size = 1000
        osd mkfs type = xfs
        
[mon.a]
        host = R710-3
        mon addr = 10.30.30.3:6789
[mon.b]
        host = R710-2
        mon addr = 10.30.30.2:6789
[mon.c]
        host = R610-5
        mon addr = 10.30.30.5:6789
[osd.11]
        host = R710-3
        devs = /dev/sdb
[osd.12]
        host = R710-3
        devs = /dev/sdc
[osd.13]
        host = R710-3
        devs = /dev/sdd
[osd.14]
        host = R710-3
        devs = /dev/sde
[osd.21]
        host = R710-2
        devs = /dev/sdb
[osd.22]
        host = R710-2
        devs = /dev/sdc
[osd.23]
        host = R710-2
        devs = /dev/sdd
[osd.24]
        host = R710-2
        devs = /dev/sde
 
[client.volumes]
    keyring = /etc/ceph/ceph.client.volumes.keyring
[client.images]
    keyring = /etc/ceph/ceph.client.images.keyring
 
#Copy ceph.conf to other 2 ceph nodes
scp /etc/ceph/ceph.conf R710-2:/etc/ceph
scp /etc/ceph/ceph.conf R610-5:/etc/ceph
Initialize ceph cluster from R710-3 node
cp /etc/ceph
mkcephfs -a -c /etc/ceph/ceph.conf -k ceph.keyring --mkfs
service ceph -a start
Check if ceph health is OK
ceph -s
 
#expected output:
   health HEALTH_OK
   monmap e1: 3 mons at {a=10.30.30.3:6789/0,b=10.30.30.2:6789/0,c=10.30.30.5:6789/0}, election epoch 30, quorum 0,1,2 a,b,c
   osdmap e72: 8 osds: 8 up, 8 in
    pgmap v5128: 1984 pgs: 1984 active+clean; 25057 MB data, 58472 MB used, 14835 GB / 14892 GB avail
   mdsmap e1: 0/0/1 up
Create pools for voluems and images
#R710-3
ceph osd pool create volumes 128
ceph osd pool create images 128


3.9. Keystone
Install the keystone packages
apt-get install -y keystone
Configure admin_token and database connection in  /etc/keystone/keystone.conf. (10.10.10.100 is the VIP of Mysql HA cluster)
 

[DEFAULT]
admin_token = manager175

[ssl]
enable = False

[signing]
token_format = UUID
[sql]
connection = mysql://keystone:manager175@10.10.10.100/keystone
Restart the keystone service then synchronize the database:
service keystone restart
keystone-manage db_sync
Configure keystone users, tenants, roles, services and endpoints by 2 scripts (manual creation is also ok, but it takes too much time)
Retrieve scripts:

wget https://github.com/abckey/OpenStack-Grizzly-Install-Guide/raw/master/KeystoneScripts/keystone_basic.sh
wget https://github.com/abckey/OpenStack-Grizzly-Install-Guide/raw/master/KeystoneScripts/keystone_endpoints.sh


Modify ADMIN_PASSWORD variable to your own value
Modify SERVICE_TOKEN variable to your own value
Modify  USER_PROJECT variable to your operation user and project name 

Modify the HOST_IP and EXT_HOST_IP variables to the HA Proxy O&M VIP and external VIP before executing the scripts. In this example, it's 10.10.10.200 and 198.154.120.143. 
Modify the SWIFT_PROXY_IP and EXT_SWIFT_PROXY_IP variables to the Swift proxy server O&M IP and external IP.  In this example, it's 10.10.10.1 and 198.154.120.134. 
Modify the MYSQL_USER and MYSQL_PASSWORD variables according your setup
Run scripts:

sh -x keystone_basic.sh
sh -x keystone_endpoints.sh
Create a simple credential file and load it so you won't be bothered later.
vi keystonerc
 
#Paste the following:
export OS_USERNAME=admin
export OS_PASSWORD=manager175
export OS_TENANT_NAME=admin
export OS_AUTH_URL="http://10.10.10.200:5000/v2.0/"
export PS1='[\u@\h \W(keystone_admin)]\$ '
 
# Source it:
source keystonerc
To test Keystone, we use a simple CLI command
keystone user-list


3.10. Glance
Install Glance packages
apt-get install -y glance
Update /etc/glance/glance-api.conf with
[DEFAULT]
sql_connection = mysql://glance:manager175@10.10.10.100/glance

[keystone_authtoken]
auth_host = 10.10.10.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = manager175
 
[paste_deploy]
flavor= keystone
Update the /etc/glance/glance-registry.conf with
[DEFAULT]
sql_connection = mysql://glance:manager175@10.10.10.100/glance
[keystone_authtoken]
auth_host = 10.10.10.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = manager175

[paste_deploy]
flavor= keystone
Create ceph authentication and keyring for glance
#R710-3
ceph auth get-or-create client.images mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images'

ceph auth get-or-create client.images |sudo tee /etc/ceph/ceph.client.images.keyring
chown glance:glance /etc/ceph/ceph.client.images.keyring
scp -p //etc/ceph/ceph.client.images.keyring R710-2:/etc/ceph/
ssh R710-2 chown glance:glance /etc/ceph/ceph.client.images.keyring
Restart glance-api and glance-registry services
service glance-api restart; service glance-registry restart
Synchronize the glance database:
glance-manage db_sync
To test Glance, upload a cirros image from internet:
glance image-create --name cirros-qcow2--is-public true --container-format bare --disk-format qcow2 --copy-from https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
Now list the image to see what you have just uploade
glance image-list

#Also you can check if it's stored on ceph rbd images pool:
rados -p images ls 


3.11. Quantum
Install the Quantum server and the OpenVSwitch package collection
apt-get install -y quantum-server
Update the OVS plugin configuration file /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini with
[DATABASE]
sql_connection = mysql://quantum:manager175@10.10.10.100/quantum
 
[OVS]
tenant_network_type = vlan
network_vlan_ranges = physnet1:1000:1100
integration_bridge = br-int
 
[SECURITYGROUP]
firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
Edit /etc/quantum/quantum.conf
[DEFAULT]
rabbit_host = 10.10.10.101
rabbit_password=manager175


[keystone_authtoken]
auth_host = 10.10.10.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = quantum
admin_password = manager175
signing_dir = /var/lib/quantum/keystone-signing
Restart the quantum server
service quantum-server restart


3.12. Nova
Start by installing nova server related components
apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy nova-doc nova-conductor
Update the  /etc/nova/api-paste.ini filelike this:
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 10.10.10.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = manager175
signing_dir = /tmp/keystone-signing-nova
Update the /etc/nova/nova.conf like this, replace 10.10.10.3 with the management IP of the controller node
[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/var/lock/nova
force_dhcp_release=True
iscsi_helper=tgtadm
libvirt_use_virtio_for_bridges=True
connection_type=libvirt
root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
#verbose=True
#debug=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
volumes_path=/var/lib/nova/volumes
enabled_apis=ec2,osapi_compute,metadata

#Message queue and DB
rabbit_host=10.10.10.101
rabbit_password=manager175
nova_url=http://10.10.10.200:8774/v1.1/
sql_connection=mysql://nova:manager175@10.10.10.100/nova


# Auth
use_deprecated_auth=false
auth_strategy=keystone


# Imaging service
glance_api_servers=10.10.10.200:9292
image_service=nova.image.glance.GlanceImageService


# Vnc configuration
novncproxy_base_url=http://198.154.120.143:6080/vnc_auto.html
novncproxy_port=6080
#vncserver_proxyclient_address=10.10.10.3
#vncserver_listen=0.0.0.0
novncproxy_host=10.10.10.3


#Memcached
memcached_servers=10.10.10.3:11211,10.10.10.2:11211

# Network settings
network_api_class=nova.network.quantumv2.api.API
quantum_url=http://10.10.10.200:9696
quantum_auth_strategy=keystone
quantum_admin_tenant_name=service
quantum_admin_username=quantum
quantum_admin_password=manager175
quantum_admin_auth_url=http://10.10.10.200:35357/v2.0
libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
#If you want Quantum + Nova Security groups
firewall_driver=nova.virt.firewall.NoopFirewallDriver
security_group_api=quantum
#If you want Nova Security groups only, comment the two lines above and uncomment line -1-.
#-1-firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver


#Metadata
service_quantum_metadata_proxy = True
quantum_metadata_proxy_shared_secret = howard


# Compute #
compute_driver=libvirt.LibvirtDriver


# Cinder #
volume_api_class=nova.volume.cinder.API
osapi_volume_listen_port=5900
cinder_catalog_info=volume:cinder:internalURL
Synchronize your database
nova-manage db sync
Restart nova-* services::
cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done
Check for the smiling faces on nova-* services to confirm your installation
nova-manage service list


3.13. Cinder
Install the required packages:
apt-get install -y cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms
Configure the iscsi services:
sed -i 's/false/true/g' /etc/default/iscsitarget
Restart services
service iscsitarget start
service open-iscsi start
Configure /etc/cinder/api-paste.ini like this:
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
service_protocol = http
service_host = 10.10.10.200
service_port = 5000
auth_host = 10.10.10.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = cinder
admin_password = manager175
signing_dir = /var/lib/cinder
Edit the /etc/cinder/cinder.conf to
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
volumes_dir = /var/lib/cinder/volumes

sql_connection = mysql://cinder:manager175@10.10.10.100/cinder
#iscsi_helper=ietadm
#iscsi_ip_address=10.10.10.3
rabbit_host = 10.10.10.101
rabbit_password=manager175

volume_driver=cinder.volume.drivers.rbd.RBDDriver
rbd_pool=volumes
rbd_user=volumes
rbd_secret_uuid={uuid of secret}  #The uuid will be generated from 1st compute node in compute node config section
Add env CEPH_ARGS="--id volumes" after “stop on runlevel [!2345]”  line in /etc/init/cinder-volume.conf
...
start on runlevel [2345]
stop on runlevel [!2345]
env CEPH_ARGS="--id volumes"
chdir /var/run
...
Create ceph authentication and keyring for cinder
#R710-3
ceph auth get-or-create client.volumes mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rx pool=images'

ceph auth get-or-create client.volumes | sudo tee /etc/ceph/ceph.client.volumes.keyring
chown cinder:cinder /etc/ceph/ceph.client.volumes.keyring
scp -p /etc/ceph/ceph.client.volumes.keyring R710-2:/etc/ceph/
ssh R710-2 chown cinder:cinder /etc/ceph/ceph.client.volumes.keyring
Then, synchronize your database
cinder-manage db sync
Restart the cinder services:
cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i restart; done


3.14. Horizon
Install horizon packages
apt-get install -y openstack-dashboard memcached
#Remove ubuntu theme if you like:
dpkg --purge openstack-dashboard-ubuntu-theme 
Modify the /etc/openstack-dashboard/local_settings.py file like:
OPENSTACK_HOST = "10.10.10.200"

CACHES = {
    'default': {
        'BACKEND' : 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION' : '10.10.10.200:11211'
    }
}
Modify /etc/memcached.conf, replace 127.0.0.1 with the controller management IP address. (here takes R710-3 as example)
-l 10.10.10.3
Restart memcached and httpd
service apache2 restart
service memcached restart
Try to log into dashboard webUI with admin or normal project users
http://198.154.120.143/horizon



4. Compute Nodes
4.1. Preparing the Node
Install Ubuntu 13.04
Add ceph hosts entries to /etc/hosts
10.30.30.3 R710-3
10.30.30.2 R710-2
10.30.30.5 R610-5
Update your system
apt-get update -y
apt-get upgrade -y
apt-get dist-upgrade -y
Setup ntp service
apt-get install -y ntp
Add controllers as ntp servers, then restart ntp service. 
echo "server 10.10.10.3" >> /etc/ntp.conf
echo "server 10.10.10.2" >> /etc/ntp.conf
service ntp restart
Install other services
apt-get install -y vlan bridge-utils
Enable IP_Forwarding
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sysctl -p
4.2. Networking
Edit /etc/network/interfaces, following example is for R710-7 node, change IPs accordingly for other compute nodes
auto em1
iface em1 inet static
        address 198.154.120.136
        netmask 255.255.255.224
        gateway 198.154.120.129
        dns-nameservers 8.8.8.8

#Openstack management
auto em2
iface em2 inet static
        address 10.10.10.7
        netmask 255.255.255.0

#VM traffic
auto em3
iface em3 inet static
        address 10.20.20.7
        netmask 255.255.255.0
 
#Storage network for ceph
auto em4
iface em4 inet static
        address 10.30.30.7
        netmask 255.255.255.0
Restart networking service
service networking restart


4.3. KVM
Make sure that your hardware enables virtualization in BIOS, for DELL R610/R710 similar nodes, following CPU configs are recommanded
Logical processor 	Enabled
Virtualization technology	Enabled
Trubo Mode	Enabled
C1E	Disabled
C states	Disabled
Double check from OS that your hardware enables virtualization
apt-get install -y cpu-checker
kvm-ok
Install and start libvirt and kvm
apt-get install -y kvm libvirt-bin pm-utils
Edit the cgroup_device_acl array in the /etc/libvirt/qemu.conf file to:
cgroup_device_acl = [
"/dev/null", "/dev/full", "/dev/zero",
"/dev/random", "/dev/urandom",
"/dev/ptmx", "/dev/kvm", "/dev/kqemu",
"/dev/rtc", "/dev/hpet","/dev/net/tun"
]
Delete default virtual bridge
 

virsh net-destroy default
virsh net-undefine default
Enable live migration by updating /etc/libvirt/libvirtd.conf file:
 

listen_tls = 0
listen_tcp = 1
auth_tcp = "none"
Edit libvirtd_opts variable in /etc/init/libvirt-bin.conf file:
 

env libvirtd_opts="-d -l"
Edit /etc/default/libvirt-bin file
libvirtd_opts="-d -l"
Restart the libvirt service to load the new values
 

service libvirt-bin restart


4.4. OpenVSwitch
Install and start the openVSwitch
apt-get install -y openvswitch-switch openvswitch-datapath-dkms
Create the bridges and add port
 

ovs-vsctl add-br br-int
ovs-vsctl add-br br-em3
ovs-vsctl add-port br-em3 em3


4.5. Quantum
Install the Quantum openvswitch agent:
apt-get -y install quantum-plugin-openvswitch-agent
Edit the OVS plugin configuration file /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini with
[DATABASE]
sql_connection = mysql://quantum:manager175@10.10.10.100/quantum
 
[OVS]
tenant_network_type = vlan
network_vlan_ranges = physnet1:1000:1100
integration_bridge = br-int
bridge_mappings = physnet1:br-em3

[SECURITYGROUP]
firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
Edit /etc/quantum/quantum.conf
[DEFAULT]
auth_strategy = keystone
rabbit_host = 10.10.10.3:5672,10.10.10.2:5672
rabbit_password=manager175
rabbit_ha_queues=True

[keystone_authtoken]
auth_host = 10.10.10.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = quantum
admin_password = manager175
signing_dir = /var/lib/quantum/keystone-signing
Restart the service
service quantum-plugin-openvswitch-agent restart
4.6. Ceph
Updage librdb from Ceph repository
wget -q -O - 'https://ceph.com/git/?p=ceph.git;a=blob_plain;f=keys/release.asc' | apt-key add -
echo "deb http://ceph.com/debian-cuttlefish/ precise main" >> /etc/apt/sources.list.d/ceph.list
apt-get upgrade librbd
Copy ceph volume key to compute nodes( Do this from 1st controller node)
#R710-3
ceph auth get-key client.volumes  | ssh 10.10.10.7 tee /home/sysadmin/client.volumes.key
ceph auth get-key client.volumes  | ssh 10.10.10.8 tee /home/sysadmin/client.volumes.key
...
On one compute nodes, add ceph volumes key to libvirt (here we do this on R710-7)
#R710-7
cd /home/sysadmin
 
cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <usage type='ceph'>
    <name>client.volumes secret</name>
  </usage>
</secret>
EOF

sudo virsh secret-define --file secret.xml
<uuid of secret is output here>
sudo virsh secret-set-value --secret {uuid of secret} --base64 $(cat client.volumes.key) 


#Dump defined secret to xml, to be imported to other compute nodes
 virsh secret-dumpxml <uuid  of secret> > dump.xml
 
#Copy the dump.xml to other compute nodes
scp dump.xml 10.10.10.8:/home/sysadmin
...
On other compute nodes, add the ceph volumes key
#R710-8 and other compute nodes
virsh secret-define --file dump.xml
virsh secret-set-value --secret {uuid of secret} --base64 $(cat client.volumes.key) 


4.7. Nova
Install nova's required components for the compute node:
apt-get install -y nova-compute-kvm nova-novncproxy
Now modify authtoken section in the /etc/nova/api-paste.ini file to this:
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 10.10.10.200
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = manager175
signing_dir = /tmp/keystone-signing-nova
Edit /etc/nova/nova-compute.conf file
[DEFAULT]
libvirt_type=kvm
compute_driver=libvirt.LibvirtDriver
libvirt_ovs_bridge=br-int
libvirt_vif_type=ethernet
libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
libvirt_use_virtio_for_bridges=True
Edit /etc/nova/nova.conf file, replace 10.10.10.7 with management IP of the compute node
[DEFAULT]
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/run/lock/nova
verbose=True
#debug=True
api_paste_config=/etc/nova/api-paste.ini
compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
 
# Message queue and DB connection
rabbit_host = 10.10.10.101
rabbit_password=manager175
sql_connection=mysql://nova:manager175@10.10.10.100/nova
root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

# Auth
use_deprecated_auth=false
auth_strategy=keystone

# Imaging service
glance_api_servers=10.10.10.200:9292
image_service=nova.image.glance.GlanceImageService

# Vnc configuration
novncproxy_base_url=http://198.154.120.143:6080/vnc_auto.html
vncserver_proxyclient_address=10.10.10.7
vncserver_listen=10.10.10.7

# Network settings
network_api_class=nova.network.quantumv2.api.API
quantum_url=http://10.10.10.200:9696
quantum_auth_strategy=keystone
quantum_admin_tenant_name=service
quantum_admin_username=quantum
quantum_admin_password=manager175
quantum_admin_auth_url=http://10.10.10.200:35357/v2.0
libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
#If you want Quantum + Nova Security groups
firewall_driver=nova.virt.firewall.NoopFirewallDriver
security_group_api=quantum

#Metadata
service_quantum_metadata_proxy = True
quantum_metadata_proxy_shared_secret = howard

# Compute #
compute_driver=libvirt.LibvirtDriver

# Cinder #
volume_api_class=nova.volume.cinder.API
osapi_volume_listen_port=5900
cinder_catalog_info=volume:cinder:internalURL
Restart nova-* service
cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done
Check for the smiling faces on nova-* services to confirm your installation, run this on controller node:
nova-manage service list


 

5. Swift Node
 In this case, we install a All-in-One Swift node with 4 internal disks simulating 4 zones. Swift Proxy or OS itself has no HA protection

5.1. Preparing the Node
Install Ubuntu 13.04
Update your system
apt-get update -y
apt-get upgrade -y
apt-get dist-upgrade -y
 Setup ntp service
apt-get install -y ntp
Add controllers as ntp servers, then restart ntp service. 
echo "server 10.10.10.3" >> /etc/ntp.conf
echo "server 10.10.10.2" >> /etc/ntp.conf
service ntp restart


5.2. Networking
Edit /etc/network/interfaces
auto em1
iface em1 inet static
        address 198.154.120.134
        netmask 255.255.255.224
        gateway 198.154.120.129
        dns-nameservers 8.8.8.8

#Openstack management
auto em2
iface em2 inet static
        address 10.10.10.1
        netmask 255.255.255.0
Restart networking service
service networking restart


5.3. Swift Storage
Install swift related packages
apt-get install swift swift-account swift-container swift-object python-swiftclient python-webob
Edit /etc/swift/swift.conf file
[swift-hash]
swift_hash_path_suffix = howard
For those 4 disks, setup the XFS filesystem, setup mount points, create need folders
mkfs.xfs -i size=1024 /dev/sdb
mkfs.xfs -i size=1024 /dev/sdc
mkfs.xfs -i size=1024 /dev/sdd
mkfs.xfs -i size=1024 /dev/sde
 
echo "/dev/sdb /srv/node/sdb xfs noatime,nodiratime,nobarrier,logbufs=8 0 0" >> /etc/fstab
echo "/dev/sdc /srv/node/sdc xfs noatime,nodiratime,nobarrier,logbufs=8 0 0" >> /etc/fstab
echo "/dev/sdd /srv/node/sdd xfs noatime,nodiratime,nobarrier,logbufs=8 0 0" >> /etc/fstab
echo "/dev/sde /srv/node/sde xfs noatime,nodiratime,nobarrier,logbufs=8 0 0" >> /etc/fstab
 
mkdir -p /srv/node/sdb
mkdir -p /srv/node/sdc
mkdir -p /srv/node/sdd
mkdir -p /srv/node/sde
 
mount /srv/node/sdb
mount /srv/node/sdc
mount /srv/node/sdd
mount /srv/node/sde
 
for x in 1 2 3 4;do mkdir -p /srv/$x/node/sdx$x;done
 
ln -s /srv/node/sdb /srv/1
ln -s /srv/node/sdc /srv/2
ln -s /srv/node/sdd /srv/3
ln -s /srv/node/sde /srv/4
 
mkdir -p /etc/swift/object-server /etc/swift/container-server /etc/swift/account-server
rm -f /etc/swift/*server.conf
mkdir -p /var/cache/swift /var/cache/swift2 /var/cache/swift3 /var/cache/swift4 /etc/swift/keystone-signing-swift
chown -R swift:swift /etc/swift/ /srv/* /var/run/swift/ /var/cache/swift* 
 
#Add following to /etc/rc.local before "exit 0"
mkdir -p /var/cache/swift /var/cache/swift2 /var/cache/swift3 /var/cache/swift4
chown -R swift:swift /var/cache/swift*
mkdir -p /var/run/swift
chown -R swift:swift /var/run/swift
Create /etc/rsyncd.conf file
uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = 127.0.0.1
[account6012]
max connections = 25
path = /srv/1/node/
read only = false
lock file = /var/lock/account6012.lock
[account6022]
max connections = 25
path = /srv/2/node/
read only = false
lock file = /var/lock/account6022.lock
[account6032]
max connections = 25
path = /srv/3/node/
read only = false
lock file = /var/lock/account6032.lock
[account6042]
max connections = 25
path = /srv/4/node/
read only = false
lock file = /var/lock/account6042.lock

[container6011]
max connections = 25
path = /srv/1/node/
read only = false
lock file = /var/lock/container6011.lock
[container6021]
max connections = 25
path = /srv/2/node/
read only = false
lock file = /var/lock/container6021.lock
[container6031]
max connections = 25
path = /srv/3/node/
read only = false
lock file = /var/lock/container6031.lock
[container6041]
max connections = 25
path = /srv/4/node/
read only = false
lock file = /var/lock/container6041.lock
[object6010]
max connections = 25
path = /srv/1/node/
read only = false
lock file = /var/lock/object6010.lock
[object6020]
max connections = 25
path = /srv/2/node/
read only = false
lock file = /var/lock/object6020.lock
[object6030]
max connections = 25
path = /srv/3/node/
read only = false
lock file = /var/lock/object6030.lock
[object6040]
max connections = 25
path = /srv/4/node/
read only = false
lock file = /var/lock/object6040.lock
Enable and restart rsyncd
#Edit /etc/default/rsync
RSYNC_ENABLE=true
 
service rsync restart
Setup rsyslog for individual logging, create /etc/rsyslog.d/10-swift.conf:
# Uncomment the following to have a log containing all logs together
#local1,local2,local3,local4,local5.*   /var/log/swift/all.log
# Uncomment the following to have hourly proxy logs for stats processing
#$template HourlyProxyLog,"/var/log/swift/hourly/%$YEAR%%$MONTH%%$DAY%%$HOUR%"
#local1.*;local1.!notice ?HourlyProxyLog
 
local1.*;local1.!notice /var/log/swift/proxy.log
local1.notice           /var/log/swift/proxy.error
local1.*                ~
local2.*;local2.!notice /var/log/swift/storage1.log
local2.notice           /var/log/swift/storage1.error
local2.*                ~
local3.*;local3.!notice /var/log/swift/storage2.log
local3.notice           /var/log/swift/storage2.error
local3.*                ~
local4.*;local4.!notice /var/log/swift/storage3.log
local4.notice           /var/log/swift/storage3.error
local4.*                ~
local5.*;local5.!notice /var/log/swift/storage4.log
local5.notice           /var/log/swift/storage4.error
local5.*                ~
Edit /etc/rsyslog.conf and make the following change:
$PrivDropToGroup adm
Change right and restart rsyslog 
mkdir -p /var/log/swift/hourly
chown -R syslog.adm /var/log/swift
chmod -R g+w /var/log/swift

service rsyslog restart
Create /etc/swift/account-server/1.conf
[DEFAULT]
devices = /srv/1/node
mount_check = false
disable_fallocate = true
bind_port = 6012
user = swift
log_facility = LOG_LOCAL2
recon_cache_path = /var/cache/swift
eventlet_debug = true
[pipeline:main]
pipeline = recon account-server
[app:account-server]
use = egg:swift#account
[filter:recon]
use = egg:swift#recon
[account-replicator]
vm_test_mode = yes
[account-auditor]
[account-reaper]
Create /etc/swift/account-server/2.conf
[DEFAULT]
devices = /srv/2/node
mount_check = false
disable_fallocate = true
bind_port = 6022
user = swift
log_facility = LOG_LOCAL3
recon_cache_path = /var/cache/swift2
eventlet_debug = true
[pipeline:main]
pipeline = recon account-server
[app:account-server]
use = egg:swift#account
[filter:recon]
use = egg:swift#recon
[account-replicator]
vm_test_mode = yes
[account-auditor]
[account-reaper]
Create /etc/swift/account-server/3.conf
[DEFAULT]
devices = /srv/3/node
mount_check = false
disable_fallocate = true
bind_port = 6032
user = swift
log_facility = LOG_LOCAL4
recon_cache_path = /var/cache/swift3
eventlet_debug = true
[pipeline:main]
pipeline = recon account-server
[app:account-server]
use = egg:swift#account
[filter:recon]
use = egg:swift#recon
[account-replicator]
vm_test_mode = yes
[account-auditor]
[account-reaper]
Create /etc/swift/account-server/4.conf
[DEFAULT]
devices = /srv/4/node
mount_check = false
disable_fallocate = true
bind_port = 6042
user = swift
log_facility = LOG_LOCAL5
recon_cache_path = /var/cache/swift4
eventlet_debug = true
[pipeline:main]
pipeline = recon account-server
[app:account-server]
use = egg:swift#account
[filter:recon]
use = egg:swift#recon
[account-replicator]
vm_test_mode = yes
[account-auditor]
[account-reaper]
Create /etc/swift/container-server/1.conf
[DEFAULT]
devices = /srv/1/node
mount_check = false
disable_fallocate = true
bind_port = 6011
user = swift
log_facility = LOG_LOCAL2
recon_cache_path = /var/cache/swift
eventlet_debug = true
[pipeline:main]
pipeline = recon container-server
[app:container-server]
use = egg:swift#container
[filter:recon]
use = egg:swift#recon
[container-replicator]
vm_test_mode = yes
[container-updater]
[container-auditor]
[container-sync]
Create /etc/swift/container-server/2.conf
[DEFAULT]
devices = /srv/2/node
mount_check = false
disable_fallocate = true
bind_port = 6021
user = swift
log_facility = LOG_LOCAL3
recon_cache_path = /var/cache/swift2
eventlet_debug = true
[pipeline:main]
pipeline = recon container-server
[app:container-server]
use = egg:swift#container
[filter:recon]
use = egg:swift#recon
[container-replicator]
vm_test_mode = yes
[container-updater]
[container-auditor]
[container-sync]
Create /etc/swift/container-server/3.conf
[DEFAULT]
devices = /srv/3/node
mount_check = false
disable_fallocate = true
bind_port = 6031
user = swift
log_facility = LOG_LOCAL4
recon_cache_path = /var/cache/swift3
eventlet_debug = true
[pipeline:main]
pipeline = recon container-server
[app:container-server]
use = egg:swift#container
[filter:recon]
use = egg:swift#recon
[container-replicator]
vm_test_mode = yes
[container-updater]
[container-auditor]
[container-sync]
Create /etc/swift/container-server/4.conf
[DEFAULT]
devices = /srv/4/node
mount_check = false
disable_fallocate = true
bind_port = 6041
user = swift
log_facility = LOG_LOCAL5
recon_cache_path = /var/cache/swift4
eventlet_debug = true
[pipeline:main]
pipeline = recon container-server
[app:container-server]
use = egg:swift#container
[filter:recon]
use = egg:swift#recon
[container-replicator]
vm_test_mode = yes
[container-updater]
[container-auditor]
[container-sync]
Create /etc/swift/object-server/1.conf
[DEFAULT]
devices = /srv/1/node
mount_check = false
disable_fallocate = true
bind_port = 6010
user = swift
log_facility = LOG_LOCAL2
recon_cache_path = /var/cache/swift
eventlet_debug = true
[pipeline:main]
pipeline = recon object-server
[app:object-server]
use = egg:swift#object
[filter:recon]
use = egg:swift#recon
[object-replicator]
vm_test_mode = yes
[object-updater]
[object-auditor]
Create /etc/swift/object-server/2.conf
[DEFAULT]
devices = /srv/2/node
mount_check = false
disable_fallocate = true
bind_port = 6020
user = swift
log_facility = LOG_LOCAL3
recon_cache_path = /var/cache/swift2
eventlet_debug = true
[pipeline:main]
pipeline = recon object-server
[app:object-server]
use = egg:swift#object
[filter:recon]
use = egg:swift#recon
[object-replicator]
vm_test_mode = yes
[object-updater]
[object-auditor]
Create /etc/swift/object-server/3.conf
[DEFAULT]
devices = /srv/3/node
mount_check = false
disable_fallocate = true
bind_port = 6030
user = swift
log_facility = LOG_LOCAL4
recon_cache_path = /var/cache/swift3
eventlet_debug = true
[pipeline:main]
pipeline = recon object-server
[app:object-server]
use = egg:swift#object
[filter:recon]
use = egg:swift#recon
[object-replicator]
vm_test_mode = yes
[object-updater]
[object-auditor]
Create /etc/swift/object-server/4.conf
[DEFAULT]
devices = /srv/4/node
mount_check = false
disable_fallocate = true
bind_port = 6040
user = swift
log_facility = LOG_LOCAL5
recon_cache_path = /var/cache/swift4
eventlet_debug = true
[pipeline:main]
pipeline = recon object-server
[app:object-server]
use = egg:swift#object
[filter:recon]
use = egg:swift#recon
[object-replicator]
vm_test_mode = yes
[object-updater]
[object-auditor]
Create rings
cd /etc/swift
swift-ring-builder object.builder create 14 3 1
swift-ring-builder object.builder add z1-127.0.0.1:6010/sdx1 1
swift-ring-builder object.builder add z2-127.0.0.1:6020/sdx2 1
swift-ring-builder object.builder add z3-127.0.0.1:6030/sdx3 1
swift-ring-builder object.builder add z4-127.0.0.1:6040/sdx4 1
swift-ring-builder object.builder rebalance
swift-ring-builder container.builder create 14 3 1
swift-ring-builder container.builder add z1-127.0.0.1:6011/sdx1 1
swift-ring-builder container.builder add z2-127.0.0.1:6021/sdx2 1
swift-ring-builder container.builder add z3-127.0.0.1:6031/sdx3 1
swift-ring-builder container.builder add z4-127.0.0.1:6041/sdx4 1
swift-ring-builder container.builder rebalance
swift-ring-builder account.builder create 14 3 1
swift-ring-builder account.builder add z1-127.0.0.1:6012/sdx1 1
swift-ring-builder account.builder add z2-127.0.0.1:6022/sdx2 1
swift-ring-builder account.builder add z3-127.0.0.1:6032/sdx3 1
swift-ring-builder account.builder add z4-127.0.0.1:6042/sdx4 1
swift-ring-builder account.builder rebalance


5.4. Swift Proxy
Install packages
apt-get instal swift-proxy memcached python-keystoneclient 
Create /etc/swift/proxy-server.conf
[DEFAULT]
bind_port = 8080
user = swift
workers = 8
log_facility = LOG_LOCAL1
#eventlet_debug = true
[pipeline:main]
pipeline = healthcheck cache tempauth  authtoken keystone proxy-server 
#pipeline = healthcheck cache tempauth proxy-logging proxy-server
[app:proxy-server]
use = egg:swift#proxy
allow_account_management = true
account_autocreate = true
[filter:tempauth]
use = egg:swift#tempauth
user_admin_admin = admin .admin .reseller_admin
user_test_tester = testing .admin
user_test2_tester2 = testing2 .admin
user_test_tester3 = testing3
[filter:healthcheck]
use = egg:swift#healthcheck
[filter:cache]
use = egg:swift#memcache
[filter:proxy-logging]
use = egg:swift#proxy_logging
memcache_servers = 127.0.0.1:11211
[filter:keystone]
use = egg:swift#keystoneauth
operator_roles = admin, SwiftOperator, Member
is_admin = true
cache = swift.cache
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
admin_tenant_name = service
admin_user = swift
admin_password = manager175
auth_host = 10.10.10.200
auth_port = 5000
auth_protocol = http
signing_dir = /etc/swift/keystone-signing-swift
admin_token = manager175
Make sure all the config files are owned by the swift user
chown -R swift:swift /etc/swift
Start all swift services
swift-init proxy start
swift-init all start
Check if swift works on controller by swift CLI:
source /home/sysadmin/keystonerc
swift stat
 You will see something similar to:

[root@R710-1 ~(keystone_admin)]# swift stat
Account: AUTH_22daa9971308497bbac6dc0fa51179fb
Containers: 0
Objects: 0
Bytes: 0
Accept-Ranges: bytes
X-Timestamp: 1366796485.27588
Content-Type: text/plain; charset=utf-8


 

6. Create networks and router to start VM launching
To start your first VM, we first need to create an internal network, router and an external network.

On keystone node, source keystonerc file
source /home/sysadmin/keystonerc
Retreive howard tenant id
keystone tenant-list
+----------------------------------+---------+---------+
|                id                |   name  | enabled |
+----------------------------------+---------+---------+
| e0da799e414f43a09aa5de7f317cf213 |  admin  |   True  |
| 56c5fbd23bf8440d8898797fe8537839 |  howard |   True  |
| 4f6c6b1a141f44a4ad0523a5ddb3f862 | service |   True  |
+----------------------------------+---------+---------+
Create a new network for the tenant howard
quantum net-create  --tenant-id 56c5fbd23bf8440d8898797fe8537839 net-1
 
Created a new network:
+---------------------------+--------------------------------------+
| Field | Value |
+---------------------------+--------------------------------------+
| admin_state_up | True |
| id | 554e786e-8a37-4dcc-840b-ced0eb4fd985 |
| name | net-1 |
| provider:network_type | vlan |
| provider:physical_network | physnet1 |
| provider:segmentation_id | 1000 |
| router:external | False |
| shared | False |
| status | ACTIVE |
| subnets | |
| tenant_id | 56c5fbd23bf8440d8898797fe8537839 |
+---------------------------+--------------------------------------+
Create a new subnet inside the new tenant network
quantum subnet-create --tenant-id 56c5fbd23bf8440d8898797fe8537839 --name subnet-1 net-1 192.168.100.0/24
Created a new subnet:
+------------------+------------------------------------------------------+
| Field | Value |
+------------------+------------------------------------------------------+
| allocation_pools | {"start": "192.168.100.2", "end": "192.168.100.254"} |
| cidr | 192.168.100.0/24 |
| dns_nameservers | |
| enable_dhcp | True |
| gateway_ip | 192.168.100.1 |
| host_routes | |
| id | 91dcb379-1d79-44da-9d14-87df64e2e5e8 |
| ip_version | 4 |
| name | subnet-1 |
| network_id | 554e786e-8a37-4dcc-840b-ced0eb4fd98 |
| tenant_id | 56c5fbd23bf8440d8898797fe8537839 |
+------------------+------------------------------------------------------+
Create the router
quantum router-create  --tenant-id  56c5fbd23bf8440d8898797fe8537839  router-net-1
Created a new router:
+-----------------------+--------------------------------------+
| Field | Value |
+-----------------------+--------------------------------------+
| admin_state_up | True |
| external_gateway_info | |
| id | 6ec77746-fd71-4ee1-862e-0cbc6f893c8c |
| name | router-net-1 |
| status | ACTIVE |
| tenant_id | 56c5fbd23bf8440d8898797fe8537839 |
+-----------------------+--------------------------------------+
Add the subnet-1 to the router
quantum router-interface-add 6ec77746-fd71-4ee1-862e-0cbc6f893c8c subnet-1
Added interface to router 6ec77746-fd71-4ee1-862e-0cbc6f893c8c
Create the external network
quantum net-create external-net --router:external=True
Created a new network:
+---------------------------+--------------------------------------+
| Field | Value |
+---------------------------+--------------------------------------+
| admin_state_up | True |
| id | 5235abad-f911-4a1d-98ad-d9cb3d2fcf84 |
| name | external-net |
| provider:network_type | vlan |
| provider:physical_network | physnet1 |
| provider:segmentation_id | 1001 |
| router:external | True |
| shared | False |
| status | ACTIVE |
| subnets | |
| tenant_id | e0da799e414f43a09aa5de7f317cf213 |
+---------------------------+--------------------------------------+
Create the subnet for floating IPs
 

quantum subnet-create --name subnet-external external-net --allocation-pool start=108.166.166.66,end=108.166.166.66 108.166.166.64/26 -- --enable_dhcp=False 
Created a new subnet:
+------------------+----------------------------------------------------+
| Field | Value |
+------------------+----------------------------------------------------+
| allocation_pools | {"start": "108.166.166.66", "end": "108.166.166.66"} |
| cidr | 108.166.166.64/26 |
| dns_nameservers | |
| enable_dhcp | False |
| gateway_ip | 108.166.166.65 |
| host_routes | |
| id | 329a3754-9a07-4842-8cd8-acdf4e5206a9 |
| ip_version | 4 |
| name | subnet-external |
| network_id | 5235abad-f911-4a1d-98ad-d9cb3d2fcf84 |
| tenant_id | e0da799e414f43a09aa5de7f317cf213 |
+------------------+----------------------------------------------------+
Set the router gateway towards the external network:
 

quantum router-gateway-set  6ec77746-fd71-4ee1-862e-0cbc6f893c8c external-net
Set gateway for router 6ec77746-fd71-4ee1-862e-0cbc6f893c8c
For DHCP agent HA purpose, let's add the net-1 to DHCP agent on both network nodes
#Check dhcp agent list
quantum agent-list
+--------------------------------------+--------------------+--------+-------+----------------+
| id                                   | agent_type         | host   | alive | admin_state_up |
+--------------------------------------+--------------------+--------+-------+----------------+
| 35b8b317-4108-4504-9a89-063231d008aa | Open vSwitch agent | R610-4 | :-)   | True           |
| 423fdb06-26b1-415b-b605-de9c313daf0f | Open vSwitch agent | R610-5 | :-)   | True           |
| 4efc2293-b42f-415e-89ad-5ce5265b1df4 | DHCP agent         | R610-4 | :-)   | True           |
| 553a32bf-b8aa-41e9-b1fc-d3fcccbc602a | L3 agent           | R610-4 | :-)   | True           |
| 88de90f3-d7d6-485e-ac00-b764a5d14e7f | DHCP agent         | R610-5 | :-)   | True           |
| 9fe37082-6154-461d-b955-b8cccf2aedc0 | Open vSwitch agent | R710-8 | :-)   | True           |
| d3fb2959-311c-403e-b91c-735c5e80ed65 | Open vSwitch agent | R710-7 | :-)   | True           |
| f9648c1a-e8e9-4f39-92c7-649f2f296440 | L3 agent           | R610-5 | :-)   | True           |
+--------------------------------------+--------------------+--------+-------+----------------+
 
#Then let's add net-1 to DHCP agents on R610-4 and R610-5 hosts
quantum dhcp-agent-network-add  4efc2293-b42f-415e-89ad-5ce5265b1df4 net-1
quantum dhcp-agent-network-add  88de90f3-d7d6-485e-ac00-b764a5d14e7f net-1
That's it ! Log on to your dashboard, create your secure key and modify your security groups then create your first VM, later you could also add floating IP to your running instances!!


7. Network service recovery from a network node failure
Only quantum-l3-agent service in the solution has a single point of failure, quantum-dhcp-agent services are running in active-active mode on 2 network nodes. This section describes how to recover l3-agent service from a network node failure.

Check all agent list and status
#Check dhcp agent list
quantum agent-list
+--------------------------------------+--------------------+--------+-------+----------------+
| id                                   | agent_type         | host   | alive | admin_state_up |
+--------------------------------------+--------------------+--------+-------+----------------+
| 35b8b317-4108-4504-9a89-063231d008aa | Open vSwitch agent | R610-4 | :-)   | True           |
| 423fdb06-26b1-415b-b605-de9c313daf0f | Open vSwitch agent | R610-5 | :-)   | True           |
| 4efc2293-b42f-415e-89ad-5ce5265b1df4 | DHCP agent         | R610-4 | :-)   | True           |
| 553a32bf-b8aa-41e9-b1fc-d3fcccbc602a | L3 agent           | R610-4 | :-)   | True           |
| 88de90f3-d7d6-485e-ac00-b764a5d14e7f | DHCP agent         | R610-5 | :-)   | True           |
| 9fe37082-6154-461d-b955-b8cccf2aedc0 | Open vSwitch agent | R710-8 | :-)   | True           |
| d3fb2959-311c-403e-b91c-735c5e80ed65 | Open vSwitch agent | R710-7 | :-)   | True           |
| f9648c1a-e8e9-4f39-92c7-649f2f296440 | L3 agent           | R610-5 | :-)   | True           |
+--------------------------------------+--------------------+--------+-------+----------------+
We can see L3 agent on R610-4 and R610-5 are both alive, this is normal status.

Check our router is running on which L3 agent:
quantum l3-agent-list-hosting-router router-net-1
+--------------------------------------+--------+----------------+-------+
| id                                   | host   | admin_state_up | alive |
+--------------------------------------+--------+----------------+-------+
| f9648c1a-e8e9-4f39-92c7-649f2f296440 | R610-5 | True           | :-)   |
+--------------------------------------+--------+----------------+-------+
We can see now our router "router-net-1" is running on R610-5 node.

If the R610-5 node is down, we should see the alive status in output above is "XXX" instead of ":-）“, then we need to switch it over to another running node: R610-4:
# Firstly, remove the router from R610-5
quantum  l3-agent-router-remove <L3 Agent ID of R610-5> router-net-1
 
# Secondly,add the router to R610-4
quantum l3-agent-router-add <L3 Agent ID of R610-4> router-net-1
 
#L3 Agent IDs can be read from output of "quantum agent-list"
 
#Then check again:
quantum l3-agent-list-hosting-router router-net-1
+--------------------------------------+--------+----------------+-------+
| id                                   | host   | admin_state_up | alive |
+--------------------------------------+--------+----------------+-------+
| f9648c1a-e8e9-4f39-92c7-649f2f296440 | R610-4 | True           | :-)   |
+--------------------------------------+--------+----------------+-------+
You should see the router is now alive on R610-4.
