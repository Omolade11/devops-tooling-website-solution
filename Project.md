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





