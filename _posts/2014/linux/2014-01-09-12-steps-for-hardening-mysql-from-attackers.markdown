---
author: AnirudhAnand
comments: true
date: 2014-01-09 16:04:33+00:00
layout: post
slug: 12-steps-for-hardening-mysql-from-attackers
title: 12 steps for Hardening MySQL from Attackers !!
wordpress_id: 782
categories:
- hardening
- linux
summary: The most important steps that are required to harden MYSQL from attackers
---

MySQL is a relational database management system, based on the Structured Query Language (SQL), which is designed to modify information in the database. It is very popular for its fast performance, high reliability, and ease of use. Relational databases are the common way of storing the increasing amount of information. The way of controlling and operating relational databases depends on TCP/IP, with all the advantages and disadvantages given naturally by using them.

The database systems are prone to many security risks like unauthorized activity, malware infections, data corruption or data loss, performance constraint etc. so these database systems and the programs and the functions within them should be secured.

**Hardening MySQL  **

Securing and hardening database systems (primarily on Linux) using MySQL, as this software is widespread and commonly used. **


**1) Securing the server**:

The application server and the database server should be on different systems, as mostly attacks are possible if physical access is obtained. If they are on the same machine, then any service running on the that machine should be given the lowest permission which will still let the service to run.Remember to install Antivirus and Antispam software, configure firewall, harden the production server and services, disable unnecessary services.


**2) Secure root account with strong password**:

When MySQL is initially installed, default root user account is created, hackers often try to gain access of its permission. So change the root username and password,

To change the admin's username, use the rename command in the MySQL console:
  
   `mysql> RENAME USER root TO new_user;            // default username is root`

To change (or update) a root password:
    
   `mysqladmin -u root -p oldpassword newpass`


**3) Confine or disable remote access**:

Through TCP wrappers, iptables, or any other firewall software or hardware, it can be ensure that only defined hosts can access the server. To restrain the MySQL from opening a network socket, add "skip-networking" in the [mysqld] section of `my.cnf` or `my.ini` ( the file is found in `/etc/mysql/my.cnf` ) .


**4) Disable unauthorized reading from local files**:

Disable the `LOAD DATA LOCAL INFILE` command as this command can load a file located in the local host or in the server host, in certain cases this command can be used to load the file in which the password is stored. Disabling this will prevent unauthorized reading from local file. In binary distributions, all clients and libraries are compiled with the -enable-local-infile option.

To disable it, add the following to the MySQL configuration file:

   `set-variable=local-infile=0`


**5) Remove unwanted database user accounts**:

When MySQL is installed, default anonymous user accounts are created, which doesn't have passwords, which makes it easier for an anonymous user to log in to the server.

To remove the account, execute the following command:

    
   `MySQL> DROP USER "";                            // MySQL version 5.0 and above`
    
For older versions:
    
   `# MySQL -u root -p`
    
   `MySQL> DELETE FROM mysql.user WHERE User = ”;`
    
   `MySQL> FLUSH PRIVILEGES;`


**6) Remove unused database**:

Default databases (test) comes pre-installed with MySQL, they should be removed if not necessary as otherwise unauthorized users can access sensitive information stored in this database. To remove this, use
    
   `MySQL> drop database test;`


**7) Lower system privileges**:

To protect the databases, the file directory in which the MySQL directory is stored should be owned by the user "MySQL" and the group "MySQL". Ensure that only the user "MySQL" and the user "root" have access to the directory var/lib/MySQL. Likewise, the MySQL binaries, which reside under the directory /usr/bin/ , should be owned by "root" or the specific system "MySQL" user. Other users shouldn't have the access to write to these files.

**8) Lower database privileges**:

Only users who really need root privileges should be granted them, all the other user's privileges should be reviewed. Anyone having access to the MySQL prompt can use the command "SHOW DATABASES", can use it to gather data before attacking the database. To disable the usage of this command add the following to /etc/my.cnf
    
   `skip-show-database`


**9) Enable logging in mysql server**:

To monitor critical events in a server enable the transaction logging by adding the following lines to the configuration file (_**/etc/my.cnf**_):

`log= /var/log/logfile`

Ensure only `root` and `mysql` have access to the logfile `hostname.err` and `logfileXY`. These files are stored in the mysql data directory.


**10) Change the root directory**:

`Chroot` is the process of changing the apparent disk root directory (and the current running process and its children) to another root directory. The write access of the MySQL processes can be limited by using the chroot environment.To make use of he database administrative tools convenient, add the following to the configuration file:
    
   `socket = /chroot/mysql/tmp/mysql.sock`


**11) Remove History**:

During the installation, there is a lot of sensitive information that can assist an intruder to assault a database, this information is stored in the server's history, they maybe of very good use during installation, but its not needed after the installation. The content of the history file (`~/.mysql_history`) should be removed, where all executed commands are stored (including password).

   `cat /dev/null > ~/mysql_history`


**12) MySQL should be run on non default port**:

MySQL runs on a default port, and the connections should be changed to a different port otherwise, the port will be detected as open by a port scanner and an unauthorized user can get more information about the mysql server. The port can be changed from the configuration file (/etc/mysql), set the port to a different port rather than the default one.

By following these steps we can make the database immune to many of the attacks. We can limit the attack possibilities from users who visit our web servers with unfair intentions.