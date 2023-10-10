# Deployment Openstack Wallaby With Kolla-Ansible
## Documentation

[Ansible](https://docs.ansible.com/)

[Docker](https://docs.docker.com/)

[Openstack Wallaby Deployment](https://docs.openstack.org/Wallaby/deploy/index.html)

[Window Server Image](https://1drv.ms/u/s!Ah2QX3LCAkYQi5B4zGVbbQm6u3kAJA?e=fvtate)

[Example config](https://1drv.ms/f/s!Ah2QX3LCAkYQi5EAYUC4I1TBJ1ZaKg)

  
## Host machine

Edit netplan

```
# This is the network config written by 'subiquity'
network:
  ethernets:
    eno1: {}
    eno2: {}
    eno3:
      addresses: [10.10.24.200/24]
      gateway4: 10.10.24.1
      nameservers:
        addresses:
        - 8.8.8.8
  bonds:
    bond0:
      interfaces: [eno1,eno2]
      parameters:
        mode: 802.3ad
  version: 2

```

----------
# In All Node

Verify connectivity
```
 ping -c 4 8.8.8.8
```

Update the package index
```
sudo apt-get update
```

Install SSH
```
apt install ssh 
```


Set timezone
```
timedatectl set-timezone Asia/Ho_Chi_Minh
```

----------
# In storage node

###  Create Cinder volume
Use **pvcreate** and **vgcreate** to create the volume group

⚠️ ALL DATA ON /dev/sdb and /dev/sdc will be LOST!

```
apt install lvm2
```

```
pvcreate /dev/sdb /dev/sdc
vgcreate cinder-volumes /dev/sdb /dev/sdc
```

----------

# In Controller node

install sshpass

```
apt install sshpass
```

Generating ssh key
```
ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa
```

copy keygen

```
ssh-copy-id compute1@192.168.99.20
ssh-copy-id .....
```
note: copy public key to .ssh/authorized_keys 
edit /etc/hosts

```
192.168.99.10 controller
192.168.99.20 compute1
192.168.99.10 network01
192.168.99.10 storage01

```
**SET DOCKER PROXY**

Create a systemd drop-in directory for the Docker service
```
mkdir /etc/systemd/system/docker.service.d
```

Create a file http-proxy.conf and adds the HTTP_PROXY environment variable:
```
vi /etc/systemd/system/docker.service.d/http-proxy.conf
```
```
[Service]
Environment="HTTP_PROXY=http://10.61.11.42:3128/"
Environment="HTTPS_PROXY=http://10.61.11.42:3128/"
Environment="NO_PROXY=localhost"
```

Flush changes:
```
sudo systemctl daemon-reload
```

Verify that the configuration has been loaded:
```
sudo systemctl show --property Environment docker
```

Restart Docker:
```
sudo systemctl restart docker
```


## Deployment

### Install dependencies

Install Python build dependencies
```
sudo apt-get install python3-dev libffi-dev gcc libssl-dev
```

Install pip
```
sudo apt-get install python3-pip
```

Ensure the latest version of pip is installed
```
sudo pip3 install -U pip
```


### Install Ansible

```
sudo apt-get install ansible
```

### Install Kolla-ansible

Install kolla-ansible and its dependencies using pip
```
sudo pip3 install kolla-ansible
```

Create the /etc/kolla directory
```
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
```

Copy globals.yml and passwords.yml to /etc/kolla directory
```
cp -r /usr/local/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
```

Copy all-in-one and multinode inventory files to the current directory
```
cp /usr/local/share/kolla-ansible/ansible/inventory/* .
```

### Configure Ansible
Add the following options to the Ansible configuration file
```
vi /etc/ansible/ansible.cfg
```
```
[defaults]
host_key_checking=False
pipelining=True
forks=100
```

### Prepare initial configuration
Edit multinode file
```
vi multinode
```
```
[all:vars]
ansible_connection=ssh
ansible_become=true
ansible_ssh_port=22
#ansible_ssh_user=sysadmin
ansible_ssh_pass=1
ansible_sudo_pass=1

[control]
control01

[network]
network01

[monitoring]
control01
compute01

[storage]
storage01

[compute]
compute01

[deployment]
controller

```

Check whether the configuration of inventory is correct or not
```
ansible -i multinode all -m ping
```

Running random password generator
```
kolla-genpwd
```

Edit main configuration file for Kolla-Ansible 
```
vi /etc/kolla/globals.yml
```
```

kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "wallaby"
neutron_bridge_name: "br-ex"
network_interface: "bond0.2000"
neutron_external_interface: "bond0"
kolla_internal_vip_address: "192.168.99.10"
enable_chrony: "yes"
enable_neutron_provider_networks: "yes"
enable_haproxy: "yes"
nova_compute_virt_type: "kvm"

enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
enable_cinder_backup: "no"

enable_prometheus: "yes"
enable_grafana: "yes"
enable_gnocchi: "yes"
enable_collectd: "yes"
enable_senlin: "yes"

```

### Deployment
Install kolla-ansible Galaxy deps
```
kolla-ansible install-deps
```


Bootstrap servers with kolla deploy dependencies
```
kolla-ansible -i multinode bootstrap-servers
```

Do pre-deployment checks for hosts:
```
kolla-ansible -i multinode prechecks
```

Finally proceed to actual OpenStack deployment:

```
kolla-ansible -i multinode pull
```

```
kolla-ansible -i multinode deploy
```


### Using OpenStack
Install the OpenStack CLI client:
```
apt install python3-openstackclient
```
Generate admin-openrc file:
```
kolla-ansible post-deploy

. /etc/kolla/admin-openrc.sh

cp /etc/kolla/admin-openrc.sh admin-openrc.sh

source admin-openrc.sh
```

### Create example networks, images, and so on
```
/usr/local/share/kolla-ansible/init-runonce
```

or

Configuring neutron
```
ENABLE_EXT_NET=${ENABLE_EXT_NET:-1}
EXT_NET_CIDR=${EXT_NET_CIDR:-'10.10.90.0/24'}
EXT_NET_RANGE=${EXT_NET_RANGE:-'start=10.10.90.10,end=10.10.90.199'}
EXT_NET_GATEWAY=${EXT_NET_GATEWAY:-'10.10.90.1'}

openstack network create --external --provider-physical-network physnet1 \
   --provider-network-type vlan Vlan90

openstack subnet create --dhcp \
   --allocation-pool ${EXT_NET_RANGE} --network Vlan90 \
   --subnet-range ${EXT_NET_CIDR} --gateway ${EXT_NET_GATEWAY} Vlan90-subnet1

```

Get admin user and tenant IDs
```
ADMIN_USER_ID=$(openstack user list | awk '/ admin / {print $2}')
ADMIN_PROJECT_ID=$(openstack project list | awk '/ admin / {print $2}')
ADMIN_SEC_GROUP=$(openstack security group list --project ${ADMIN_PROJECT_ID} | awk '/ default / {print $2}')
```

Sec Group Config
```
openstack security group rule create --ingress --ethertype IPv4 \
   --protocol icmp ${ADMIN_SEC_GROUP}
openstack security group rule create --ingress --ethertype IPv4 \
   --protocol tcp --dst-port 22 ${ADMIN_SEC_GROUP}
openstack security group rule create --ingress --ethertype IPv4 \
   --protocol tcp --dst-port 8000 ${ADMIN_SEC_GROUP}
openstack security group rule create --ingress --ethertype IPv4 \
   --protocol tcp --dst-port 8080 ${ADMIN_SEC_GROUP}
 ```
 
Configuring nova public key and quotas
```
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
```

Increase the quota to allow 40 m1.small instances to be created
```
openstack quota set --instances 40 ${ADMIN_PROJECT_ID}
openstack quota set --cores 40 ${ADMIN_PROJECT_ID}
openstack quota set --ram 96000 ${ADMIN_PROJECT_ID}
```

Add default flavors, if they don't already exis
```
openstack flavor create --id 1 --ram 512 --disk 1 --vcpus 1 m1.tiny
openstack flavor create --id 2 --ram 2048 --disk 20 --vcpus 1 m1.small
openstack flavor create --id 3 --ram 4096 --disk 40 --vcpus 2 m1.medium
openstack flavor create --id 4 --ram 8192 --disk 80 --vcpus 4 m1.large
openstack flavor create --id 5 --ram 16384 --disk 160 --vcpus 8 m1.xlarge
```


Upload cirros to image service
```
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
```

```
glance image-create --name "cirros" \
  --file cirros-0.4.0-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility=public

```


### FAQ

**Cannot run apt update?**

Add "nameserver 8.8.8.8" to /etc/resolv.conf


**How to remove the failed deployment?**
```
kolla-ansible -i multinote destroy
```

**How to get horizon admin passwords?**
```
cat /etc/kolla/passwords.yml | grep keystone_admin
```


  
