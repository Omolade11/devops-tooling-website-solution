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

