# HA-WordPress

This guide outlines the steps to set up a high-availability WordPress website on Linode, using master-master replication to ensure that changes made to the website on one server are reflected on the other server. This configuration ensures that the website remains available in case of a server failure, as traffic can be routed to the other server seamlessly. The guide will cover the configuration of the databases, Apache, installation of WordPress, and synchronization of the /var/www directory on both Linodes. In addition, a node balancer will be set up to balance traffic between the two Linodes.

This guide assumes that you have a solid understanding of the Linux command line and have already set up two Linodes to serve as the web servers. Let's begin building your high-availability WordPress site on Linode!

## MySQL Master-Master Replication

### 1: Set up Linodes
Set up two Linodes by following the instructions provided by Linode. Make sure you have root access to both Linodes.

### 2: Update and Install MySQL Server
On both Linodes, update the package lists and install MySQL server using the following commands:

```
sudo apt update
sudo apt upgrade -y
sudo apt install mysql-server
```

### 3: Configure MySQL on both Linodes
1. Edit the MySQL configuration file on both Linodes using the following command:
```
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```
2. Comment out the bind-address and mysqlx-bind-address lines by adding a # character at the beginning of the line:
```
# bind-address            = 127.0.0.1
# mysqlx-bind-address     = 127.0.0.1
```
3. Add the following lines to the configuration file:
```
server_id = 1   # set to 2 for the second Linode
log_bin = /var/log/mysql/mysql-bin.log
```
4. Save and close the configuration file:
```
esc :wq
```
5. Restart the MySQL service on both Linodes:
```
exit
```
```
sudo systemctl restart mysql
```

### 4: Create Replication User on Linode 1
1. Connect to the MySQL server on Linode 1 using the following command:
```
sudo mysql -u root -p
```
2. Create a replication user with the following command:
```
CREATE USER 'rep1'@'%' IDENTIFIED BY 'password';
```
3. Set the authentication method for the replication user to mysql_native_password:
```
ALTER USER 'rep1'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
```
4. Grant replication slave privileges to the replication user:
```
GRANT REPLICATION SLAVE ON *.* TO 'rep1'@'%';
```
5. Flush the privileges:
```
FLUSH PRIVILAGES
```
6. Retrieve the binary log file name and position from the master using the following command:
```
SHOW MASTER STATUS \G;
```
Note down the LogFile and Position values for use in the configuration of the slave on Linode 2.

### 5: Configure Replication on Linode 2

1. Connect to the MySQL server on Linode 2 using the following command:
```
sudo mysql -u root -p
```
2. Create a replication user with the following command:
```
CREATE USER 'rep1'@'%' IDENTIFIED BY 'password';
```
3. Set the authentication method for the replication user to mysql_native_password:
```
ALTER USER 'rep1'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
```
4. Grant replication slave privileges to the replication user:
```
GRANT REPLICATION SLAVE ON *.* TO 'rep1'@'%';
```
5. Flush the privileges and retrieve binary log file:
```
FLUSH PRIVILAGES
SHOW MASTER STATUS \G;
```
6. Stop Slave
```
STOP SLAVE
```
7. Configure the replication settings for the slave by running the following command, replacing the placeholders with the appropriate values:
```
CHANGE MASTER TO MASTER_HOST='<linode1_ip_address>', MASTER_USER='rep1', MASTER_PASSWORD='password', MASTER_LOG_FILE='<LogFile>', MASTER_LOG_POS=<position(no quotes)>;
```
8. Start the slave process:
```
START SLAVE
```
### 6: Configure Replication on Linode 1
1. Connect to the MySQL server on Linode 1 using the following command:
```
sudo mysql -u root -p
```
2. Stop the slave process:
```
STOP SLAVE;
```
3. Configure the replication settings for the slave by running the following command, replacing the placeholders with the appropriate values:
```
CHANGE MASTER TO MASTER_HOST='<linode2_ip_address>', MASTER_USER='rep1', MASTER_PASSWORD='password', MASTER_LOG_FILE='<LogFile>', MASTER_LOG_POS=<position(no quotes)>;
```
4. Start the slave process:
```
START SLAVE
```
### 7: Reset Masters
On both Linodes, run the following commands:
```
reset master;
stop slave;
reset slave;
start slave;
```

## Configure Apache
The steps in this section will need to be performed on both of your Linodes.

