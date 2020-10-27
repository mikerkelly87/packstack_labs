# Packstack Installation

Once the CentOS 8 OS has been installed on the VM or physical machine SSH into it as root and follow these steps...

**Install the network-scripts package then disable and stop the firewall along with Network Manager.**
```
dnf install vim bash-completion wget network-scripts -y
systemctl disable firewalld
systemctl stop firewalld
systemctl disable NetworkManager
systemctl stop NetworkManager
systemctl enable network
systemctl start network
```

**Enable the PowerTools Repo, install the Ussuri release Openstack Packages, then update all packages**
```
dnf config-manager --enable PowerTools
dnf install -y centos-release-openstack-ussuri
dnf update -y
```

**Install Packstack:**
```
dnf install -y openstack-packstack
```

**Reboot so the VM is running the latest kernel:**
```
reboot
```

**Next we still start the packstack all in one installer which will deploy Openstack:**
```
packstack --allinone --provision-demo=n --os-neutron-ovn-bridge-mappings=extnet:br-ex --os-neutron-ovn-bridge-interfaces=br-ex:ens4
```
Note: Replace `ens4` in the command above with whatever the name of your interface is named
  
This command is going to take a long time to finish (up to an hour or more depending on hardware specs of your host machine)

It will seem like it's stuck at "Applying X.X.X.X_controller.pp" forever, but unless it fails assume it's still running

This is a good opportunity to go trim your neck beard


**Once the installation is complete we have a functional Openstack cloud. It's time to create some cloud resources.**


**Create a directory to download cloud images to:**
```
mkdir /root/images
cd /root/images
```

**Download the images:**
```
wget http://download.cirros-cloud.net/0.5.1/cirros-0.5.1-x86_64-disk.img
wget http://cdimage.debian.org/cdimage/openstack/current/debian-10.5.0-openstack-amd64.qcow2
wget https://download.fedoraproject.org/pub/fedora/linux/releases/32/Cloud/x86_64/images/Fedora-Cloud-Base-32-1.6.x86_64.qcow2
wget https://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img
wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
wget http://download.opensuse.org/distribution/leap/15.2/appliances/openSUSE-Leap-15.2-JeOS.x86_64-OpenStack-Cloud.qcow2

# wget https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2
# wget https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.2.2004-20200611.2.x86_64.qcow2
```
I commented out the wget commands for downloading the CentOS images. While they will work they can take forever. If they are taking forever you might want to download them overnight or possibly find a torrent link and scp/sftp them to the `/root/images/` directory on the VM.

**Source the rc file so that the openstack CLI client will be able to communicate with and authenticate to your cloud:**
```
source /root/keystonerc_admin
```
This must be done every time you login to the VM to interact with your cloud. You can also copy this file to your local machine and download the `python3-openstackclient` package if you didn't want to SSH into the VM anymore.

**Upload the images to the Glance image service:**
```
openstack image create --disk-format qcow2 --container-format bare --public --file ./cirros-0.5.1-x86_64-disk.img "CirrOS"
openstack image create --disk-format qcow2 --container-format bare --public --file ./debian-10.5.0-openstack-amd64.qcow2 "Debian 10"
openstack image create --disk-format qcow2 --container-format bare --public --file ./Fedora-Cloud-Base-32-1.6.x86_64.qcow2 "Fedora 32"
openstack image create --disk-format qcow2 --container-format bare --public --file ./bionic-server-cloudimg-amd64.img "Ubuntu 18.04"
openstack image create --disk-format qcow2 --container-format bare --public --file ./focal-server-cloudimg-amd64.img "Ubuntu 20.04"
openstack image create --disk-format qcow2 --container-format bare --public --file ./openSUSE-Leap-15.2-JeOS.x86_64-OpenStack-Cloud.qcow2 "OpenSuse Leap 15"
openstack image create --disk-format qcow2 --container-format bare --public --file ./CentOS-7-x86_64-GenericCloud.qcow2 "CentOS 7"
openstack image create --disk-format qcow2 --container-format bare --public --file ./CentOS-8-GenericCloud-8.2.2004-20200611.2.x86_64.qcow2 "CentOS 8"
```
Obviously the CentOS uploads will fail if you chose to download those images at a later time.


**Create the external network:**
```
openstack network create GATEWAY_NET --external --provider-physical-network extnet --share --provider-network-type flat
```

**Create the external network's subnet:**
```
openstack subnet create --subnet-range 10.0.0.0/24 --gateway 10.0.0.1 --dns-nameserver 8.8.8.8 --dns-nameserver 8.8.4.4 --network GATEWAY_NET --allocation-pool start=10.0.0.220,end=10.0.0.250 GATEWAY_SUBNET_1
```
Replace all the IPs in the command above with the IP information of your home LAN. In my LAN, my home router gives out DHCP addresses lower in the range which is why I'm allocating IPs higher in the range (.220 - .250).

**Create the Neutron router and set the external gateway network as the external network we just created:**
```
openstack router create EXT_ROUTER
openstack router set --enable-snat --external-gateway GATEWAY_NET EXT_ROUTER
```

**Create an internal network**
```
openstack network create INTERNAL_NET
```

**Create the subnet for the internal network:**
```
openstack subnet create --subnet-range 192.168.70.0/24 --gateway 192.168.70.1 --dns-nameserver 8.8.8.8 --dns-nameserver 8.8.4.4 --network INTERNAL_NET --allocation-pool start=192.168.70.10,end=192.168.70.254 INTERNAL_SUBNET_1
```

**Add the internal network to the Neutron router:**
```
openstack router add subnet  EXT_ROUTER INTERNAL_SUBNET_1
```

**Create a security group named `webserver`:**
```
openstack security group create webserver
```

**Allow Ports 80, 443, 22, and ICMP traffic for pings on the `webserver` security group:**
```
openstack security group rule create webserver --protocol tcp --dst-port 80:80 --remote-ip 0.0.0.0/0
openstack security group rule create webserver --protocol tcp --dst-port 443:443 --remote-ip 0.0.0.0/0
openstack security group rule create webserver --protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0
openstack security group rule create webserver --protocol icmp --remote-ip 0.0.0.0/0
```

**Allocate 5 floating IPs:**
```
count=1; while [ $count -le 5 ];do openstack floating ip create GATEWAY_NET && $(( count++ )) 2>/dev/null;done
```

**Delete the default instance flavors that were created:**
```
for i in {1..5};do openstack flavor delete $i;done
```

**Create some instance flavors with some lower specs that make more sense for our environment:**
```
openstack flavor create --public small --id auto --ram 512 --disk 10 --vcpus 1 --rxtx-factor 1
openstack flavor create --public medium --id auto --ram 1024 --disk 15 --vcpus 1 --rxtx-factor 1
openstack flavor create --public large --id auto --ram 2048 --disk 20 --vcpus 2 --rxtx-factor 1
```
Note: You can adjust these specs to your liking, these just felt pretty sane.


**Create an SSH key for logging into instances and upload the keypair to Openstack:**
```
ssh-keygen -f /root/.ssh/packstack
openstack keypair create --public-key /root/.ssh/packstack.pub packstack
```


**Set environment variables so we can create a test instance running `Ubuntu 18.04` with the `small` flavor in the `INTERNAL_NET` network with the `webserver` security group assigned:**
```
IMAGE_ID=$(openstack image list | grep 18.04 | awk '{print $2}')
FLAVOR_ID=$(openstack flavor list | grep small | awk '{print $2}')
SECGROUP_ID=$(openstack security group list | grep webserver | awk '{print $2}')
NET_ID=$(openstack network list | grep INTERNAL_NET | awk '{print $2}')
```

**Create the instance named `ubuntu-test`:**
```
openstack server create --image $IMAGE_ID --flavor $FLAVOR_ID --security-group $SECGROUP_ID --key-name packstack --nic net-id=$NET_ID ubuntu-test
```

**Assign the `ubuntu-test` instance the first available floating IP:**
```
FLOATING_IP=$(openstack floating ip list | grep None | awk '{print $4}' | head -n1)

openstack server add floating ip ubuntu-test $FLOATING_IP
```

**Verify you can SSH into the `ubuntu-test` instance using the floating IP**
```
ssh -i /root/.ssh/packstack ubuntu@$FLOATING_IP
```

**Once you verified you can connect to the `ubuntu-test` instance you can delete it:**
```
openstack server delete ubuntu-test
```

**Now that you validated the cloud is functioning you can login to the Horizon dashboard by hitting the following URL in your web browser:**
```
http://<IP_OF_YOUR_VM>/dashboard
```
**The credentials will be in your `keystonerc_admin` file:**
```
cat /root/keystonerc_admin
```

[<-- Back to Main](../README.md)