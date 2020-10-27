### Use the Openstack CLI to Create a Wordpress site with separate web, db, and storage servers

**Source your rc file:**
```
source ./<your_rc_file>
```

**Find the UUID of the CentOS 8 image:**
```
openstack image list | grep "CentOS 8"
```

**Find the UUID of the `small` flavor:**
```
openstack flavor list | grep small
```

**Find the UUID for the INTERNAL_NET network:**
```
openstack network list | grep INTERNAL_NET
```

**Create the following scripts in your home directory:**
```
vim ~/wp-db.sh
```
```
#!/bin/bash
dnf install -y mariadb-server vim bash-completion
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
systemctl enable mariadb
systemctl start mariadb
echo "[client]" > ~/.my.cnf
echo "user=root" >> ~/.my.cnf
echo "password=" >> ~/.my.cnf
mysql -e "CREATE DATABASE wordpress"
mysql -e "CREATE USER 'wordpress_user'@'%' IDENTIFIED BY 'password12345'"
mysql -e "GRANT ALL ON wordpress.* TO 'wordpress_user'@'%'"
mysql -e "FLUSH PRIVILEGES"
```
```
vim ~/wp-storage.sh
```
```
#!/bin/bash
dnf install -y dnf nfs-utils vim bash-completion
systemctl enable nfs-server
mkdir -p /var/www/html/
echo "/var/www/html 192.168.0.0/24(rw,sync,no_root_squash)" >> /etc/exports
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
systemctl start nfs-server
systemctl enable nfs-server
```

We will pass these scripts using cloud-init at the creation time of the instances to automate the software installations.

**Create the `wp-db` instance:**
```
openstack server create --image <CentOS_8_IMAGE_UUID> --flavor <SMALL_FLAVOR_UUID> --security-group SSH --security-group mysql --key-name <your_keypair_name> --nic net-id=<INTERNAL_NET_UUID> --user-data ~/wp-db.sh wp-db
```

**List the Floating IP Addresses available:**
```
openstack floating ip list
```

**Assign the first available Floating IP to the `wp-db` instance:**
```
Note: Available FLoating IPs will have `None` in the `Fixed IP Address` field
```
```
openstack server add floating ip wp-db <X.X.X.X>
```

**Create the `wp-storage` instance:**
```
openstack server create --image <CentOS_8_IMAGE_UUID> --flavor <SMALL_FLAVOR_UUID> --security-group SSH --security-group nfs --key-name <your_keypair_name> --nic net-id=<INTERNAL_NET_UUID> --user-data ~/wp-storage.sh wp-storage
```

**List the Floating IP Addresses available:**
```
openstack floating ip list
```

**Assign the first available Floating IP to the `wp-storage` instance:**
```
Note: Available FLoating IPs will have `None` in the `Fixed IP Address` field
```
```
openstack server add floating ip wp-storage <X.X.X.X>
```

**Create the following script in your home directory:**
```
vim ~/wp-web1.sh
```
```
#!/bin/bash
dnf install -y httpd mariadb-common php-fpm php-mysqlnd wget php-json tar bash-completion vim nfs-utils nfs4-acl-tools
systemctl enable httpd && systemctl start httpd
echo "<INTERNAL_NET_IP_OF_wp-storage_SERVER>:/var/www/html /var/www/html nfs defaults 0 0" >> /etc/fstab
mount -a
wget https://www.wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz
cp -R wordpress/* /var/www/html/
cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
sed -i 's/database_name_here/wordpress/' /var/www/html/wp-config.php
sed -i 's/username_here/wordpress_user/' /var/www/html/wp-config.php
sed -i 's/password_here/password12345/' /var/www/html/wp-config.php
sed -i 's/localhost/<INTERNAL_NET_IP_OF_wp-db_SERVER>/' /var/www/html/wp-config.php
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```
```
Be sure to replace:
<INTERNAL_NET_IP_OF_wp-storage_SERVER>
<INTERNAL_NET_IP_OF_wp-db_SERVER>
in the script above
```

**Create the `wp-web1` instance:**
```
openstack server create --image <CentOS_8_IMAGE_UUID> --flavor <SMALL_FLAVOR_UUID> --security-group SSH --security-group web --key-name <your_keypair_name> --nic net-id=<INTERNAL_NET_UUID> --user-data ~/wp-web1.sh wp-web1
```

**List the Floating IP Addresses available:**
```
openstack floating ip list
```

**Assign the first available Floating IP to the `wp-web1` instance:**
```
Note: Available FLoating IPs will have `None` in the `Fixed IP Address` field
```
```
openstack server add floating ip wp-web1 <X.X.X.X>
```

It might take some time for the cloud-init script to finish depending on the speed of your hardware. Eventually you will be able to hit the Floating IP of your `wp-web1` instance in your web browser and see the Wordpress welcome screen.

![](screenshots/19.png)


[<-- Back to LABs](../README.md)
[<-- Back to Main](../../README.md)