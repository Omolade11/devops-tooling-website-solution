# DevOps Tooling Website Solution
In the previous project [Implementing Web Solution](https://github.com/Omolade11/Web-Solution-using-WordPress-and-MySQL), I implemented a WordPress based solution that is ready to be filled with content and can be used as a full fledged website or blog. Moving further I will add some more value to my solution so that a member of a DevOps team could utilize.

In this project, we will implement a solution that consists of following components:
1. Infrastructure: AWS
2. Webserver Linux: Red Hat Enterprise Linux 8
3. Database Server: Ubuntu 20.04 + MySQL
4. Storage Server: Red Hat Enterprise Linux 8 + NFS Server
5. Programming Language: PHP
6. Code Repository: GitHub


On the diagram below, we can see a common pattern where several stateless Web Servers share a common database and also access the same files using [Network File Sytem (NFS)](https://en.wikipedia.org/wiki/Network_File_System) as a shared file storage. Even though the NFS server might be located on a completely separate hardware – for Web Servers it look like a local file system from where they can serve the same files.

![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/nfs%20diagram.png)

This project consists of the following servers:

1. Web server(RHEL)
2. Database server(Ubuntu + MySQL)
3. Storage/File server(RHEL + NFS server)

## Step 1 – Prepare NFS Server
1. Spin up a new EC2 instance with RHEL Linux 8 Operating System.
2. Launch the instance and attach 3 volumes to it.
![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-23%20at%2008.37.03.png)
![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-23%20at%2014.36.13.png)
![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-23%20at%2014.49.53.png)
3. ssh to the instance and let's begin to configure our nfs server. To verify, we will run `sudo lsblk`.
We will use gdisk utility to create a single partition on each of the 3 disks:

`sudo gdisk /dev/xvdf` type n then hit enter 5 times type p hit enter once type w hit enter once type y hit enter once

`sudo gdisk /dev/xvdg` type n then hit enter 5 times type p hit enter once type w hit enter once type y hit enter once

`sudo gdisk /dev/xvdh` type n then hit enter 5 times type p hit enter once type w hit enter once type y hit enter once

4. We will verify the newly configured partition on each of the 3 disks by using `lsblk` utility
![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-25%20at%2009.32.09.png)
5. We will install lvm2 package using `sudo yum install lvm2`

6. To check for available partitions, we will run `sudo lvmdiskscan`

7. Use pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM:
```
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```
8. We will verify that our Physical volume has been created successfully by running:
`sudo pvs`

9. We will use vgcreate utility to add all 3 PVs to a volume group (VG). Name the VG webdata-vg or anything we want :
`sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1`

10. Verify that our VG has been created successfully by running:
`sudo vgs`

11. We will use lvcreate utility to create 3 logical volumes. apps-lv, logs-lv and opt-lv:
```
sudo lvcreate -n apps-lv -L 7G webdata-vg
sudo lvcreate -n logs-lv -L 7G webdata-vg
sudo lvcreate -n opt-lv -L 7G webdata-vg
```
Verify that our Logical Volume has been created successfully by running:
`sudo lvs`

![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-25%20at%2011.28.03.png)

12. Verify the entire setup by running: 

`sudo vgdisplay -v #view complete setup - VG, PV, and LV`
`sudo lsblk`

13. Instead of formatting the disks as ext4, we will have to format them as xfs
```
sudo mkfs -t xfs /dev/webdata-vg/apps-lv
sudo mkfs -t xfs /dev/webdata-vg/logs-lv
sudo mkfs -t xfs /dev/webdata-vg/opt-lv
```

14. Create mount points on /mnt directory for the logical volumes as follow: Mount lv-apps on /mnt/apps – To be used by webservers Mount lv-logs on /mnt/logs – To be used by webserver logs Mount lv-opt on /mnt/opt – To be used by Jenkins server in Project 8
```
sudo mkdir -p /mnt/apps
sudo mkdir -p /mnt/logs
sudo mkdir -p /mnt/opt
```
```
sudo mount /dev/webdata-vg/apps-lv /mnt/apps
sudo mount /dev/webdata-vg/logs-lv /mnt/logs
sudo mount /dev/webdata-vg/opt-lv /mnt/opt
```
15. Once mount is completed run:

`sudo blkid` to get the UUID, edit the fstab file accordingly

Update /etc/fstab in this format using your own UUID and remember to remove the leading and ending quotes.


`sudo vi /etc/fstab`

```
UUID=06291ab3-bbfe-4461-9b55-b48979c5a42d /mnt/logs  xfs defaults 0 0
UUID=9f26a746-20b9-49fd-9363-0d0b5001a16b /mnt/apps  xfs defaults 0 0
UUID=3aa4c9e9-802d-49c1-af17-844200c9b287 /mnt/opt   xfs defaults 0 0

```
16. Verify the mount points

```
sudo mount -a  
sudo systemctl daemon-reload
```
17. Verify our setup by running `df -h`, output must look like this:
![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-25%20at%2012.50.35.png)

18. Install NFS server, configure it to start on reboot and make sure it is up and running:

```
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```
19. Export the mounts for webservers’ subnet CIDR to connect as clients. For simplicity, you will install all three Web Servers inside the same subnet, but in the production set-up, you would probably want to separate each tier inside its own subnet for a higher level of security.

Make sure we set up permission that will allow our Web servers to read, write and execute files on NFS:

```
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt
 
sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt
 
sudo systemctl restart nfs-server.service

```



20. Next we configure NFS to interact with clients present in the same subnet.
We can find the subnet ID and CIDR in the Networking tab of our instances

![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2006.39.45.png)
![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2006.45.17.png)

```
sudo vi /etc/exports

``` 
Edit the file like the image below:

![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2007.01.43.png) 

```
/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
```
 
21. We will run this command to export
`sudo exportfs -arv`

![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2007.05.10.png)

 
22. Check which port is used by NFS and open it using Security Groups (add new Inbound Rule) `rpcinfo -p | grep nfs`
![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2007.11.33.png)

Important note: In order for NFS server to be accessible from our client, we must also open following ports: TCP 111, UDP 111, UDP 2049 and TCP 2049:

![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2007.27.47.png)


## Step 2 – Configure the Database Server

1. launch an Ubuntu 20.04 server on AWS which will serve as our Database. Ensure its in the same subnet as the NFS-Server

Install MySQL server

```
sudo apt update

sudo apt upgrade

sudo apt install mysql-server

mysql --version

```


2. Create a database and name it tooling
```
sudo mysql 
create database tooling;

```
Create a database user with name webaccess and grant permission to the user on tooling db to be able to do anything only from the webservers subnet cidr


`create user 'webaccess'@'172.31.80.0/20' identified by 'password';`

Grant permission to webaccess user on tooling database to do anything only from the webservers subnet cidr:

`grant all privileges on tooling.* to 'webaccess'@'172.31.80.0/20';`

![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2008.05.54.png)

The ip address is the webserver's IPv4 CIDR. The webserver is created in the next step below.


## Step 3 – Prepare the Web Servers

1. Create a RHEL EC2 instance on AWS which serves as our web server. Also remember to have in it in same subnet.
2. Install NFS client

`sudo yum install nfs-utils nfs4-acl-tools -y`

3. Mount /var/www/ and target the NFS server’s export for apps

```
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www

```
4. We will verify that NFS was mounted successfully by running `df -h`. Make sure that the changes will persist on Web Server after reboot:
![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2008.26.26.png)

`sudo vi /etc/fstab`

add following line :

``` <NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0 ```

![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2008.43.42.png)

5. We will install [Remi’s repository](http://www.servermom.org/how-to-enable-remi-repo-on-centos-7-6-and-5/2790/), Apache and PHP

```
sudo yum install httpd -y
 
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
 
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
 
sudo dnf module reset php
 
sudo dnf module enable php:remi-7.4
 
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
 
sudo systemctl start php-fpm
 
sudo systemctl enable php-fpm
 
sudo setsebool -P httpd_execmem 1

```
6. We will verify that Apache files and directories are available on the Web Server in /var/www and also on the NFS server in /mnt/apps. If we see the same files – it means NFS is mounted correctly. 

``` ls /var/www ```

![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2008.57.03.png)

``` ls /mnt/apps ```
![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2009.10.28.png)

7. Locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs. Make sure the mount point will persist after reboot.


``` sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/logs /var/log/httpd ```

Afterward, run `sudo vi /etc/fstab`

add the following line :

`<NFS-Server-Private-IP-Address>:/mnt/logs /var/log/httpd nfs defaults 0 0 `
![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2009.42.36.png)

8. Fork the tooling source code from Darey.io Github Account to our Github account.
![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2009.48.46.png)

9. Deploy the tooling website’s code to the Webserver. Ensure that the html folder from the repository is deployed to /var/www/html.
``` sudo yum install git -y ```
![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2010.26.21.png)


Note 1: Do not forget to open TCP port 80 on the Web Server.

Note 2: If you encounter 403 Error – check permissions to your /var/www/html folder and also disable SELinux sudo setenforce 0

To make this change permanent – open following config file `sudo vi /etc/sysconfig/selinux` and set SELINUX=disabled then restart httpd.

![](https://github.com/Omolade11/devops-tooling-website-solution/blob/main/Images/Screenshot%202023-03-28%20at%2010.47.09.png)


```
sudo systemctl restart httpd
sudo systemctl status httpd
```




