# MSSQL-AG-on-Linux-in-AWS
SQL Server always on AG on linux deployed in AWS
2 node cluster + config only node (sqlhalx1, sqlhalx2, sqlhaconfig)

**HA Setup with pacemaker steps**

1.	Setup AG (3 node cluster Primary, Replica and Configuraiton Only – express edition)
2.	Setup pacemaker cluster
3.	Add SQL Server AG to pacemaker
4.	Test failover

**Setup AG**
•	Update computer name for each host in /etc/hostname file. Reboot the node

•	Validate that all the nodes intended to be part of the availability group configuration can communicate with each other. (A ping to the hostname should reply with the corresponding IP address.) Ensure The hosts file on every server contains the IP addresses and names of all servers that will participate in the availability group. Add listener IP and Listener name here as well

•	Set sqlcmd path and reset “sa” password 

echo 'export PATH="$PATH:/opt/mssql-tools/bin:/opt/mssql/bin/"' >> ~/.bash_profile
echo 'export PATH="$PATH:/opt/mssql-tools/bin:/opt/mssql/bin/"' >> ~/.bashrc
source ~/.bashrc

sudo systemctl stop mssql-server  
sudo /opt/mssql/bin/mssql-conf set-sa-password  
sudo systemctl start mssql-server

•	Enable AOAG on primary and secondary

sudo /opt/mssql/bin/mssql-conf set hadr.hadrenabled 1
sudo systemctl restart mssql-server

•	(Optional) You can optionally enable extended events (XE) to help with root-cause diagnosis when you troubleshoot an availability group. Run the following command on each instance of SQL Server:

ALTER EVENT SESSION AlwaysOn_health ON SERVER WITH (STARTUP_STATE=ON);
GO

•	The SQL Server service on Linux uses certificates to authenticate communication between the mirroring endpoints. The following Transact-SQL script creates a master key and a certificate. It then backs up the certificate and secures the file with a private key. Update the script with strong passwords. Run the script on Primary instance to create the certificate.

CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<password>';
CREATE CERTIFICATE dbm_certificate WITH SUBJECT = 'dbm';
BACKUP CERTIFICATE dbm_certificate  TO FILE = '/var/opt/mssql/data/dbm_certificate.cer' 
WITH PRIVATE KEY (FILE = '/var/opt/mssql/data/dbm_certificate.pvk', ENCRYPTION BY PASSWORD = '<password>');


•	Copy certificate files to all secondary and config only node
scp -i /usr/bin/demo-pg.pem /var/opt/mssql/data/dbm_cer* ec2-user@sqlhalx2:/tmp

•	move to /var/opt/mssql/data/ and make “mssql” user the owner on secondary and config node

mv /tmp/db* /var/opt/mssql/data
cd /var/opt/mssql/data
chown mssql:mssql dbm_certificate.*

•	Create the certificate on secondary node and config only node

CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<password>';
CREATE CERTIFICATE dbm_certificate
    FROM FILE = '/var/opt/mssql/data/dbm_certificate.cer'
    WITH PRIVATE KEY (
           FILE = '/var/opt/mssql/data/dbm_certificate.pvk',
           DECRYPTION BY PASSWORD = '<password>' );


•	Create Endpoints in all nodes. ROLE=ALL runs on primary and secondary and WITNESS runs on config only node

CREATE ENDPOINT [Hadr_endpoint]
    AS TCP (LISTENER_PORT = 5022)
    FOR DATABASE_MIRRORING (
        ROLE = ALL,
        AUTHENTICATION = CERTIFICATE dbm_certificate,
        ENCRYPTION = REQUIRED ALGORITHM AES
        );

ALTER ENDPOINT [Hadr_endpoint] STATE = STARTED;

CREATE ENDPOINT [Hadr_endpoint]
    AS TCP (LISTENER_PORT = 5022)
    FOR DATABASE_MIRRORING (
        ROLE = WITNESS,
        AUTHENTICATION = CERTIFICATE dbm_certificate,
        ENCRYPTION = REQUIRED ALGORITHM AES
        );

ALTER ENDPOINT [Hadr_endpoint] STATE = STARTED;

•	Before creating AG, make sure @@serverproperty returns node hostname, if not, change it using following commands and restart sql server 

sp_dropserver 'ip-10-0-114-78'
sp_addserver 'sqlhalx1', 'LOCAL'

sudo systemctl stop mssql-server  
sudo systemctl start mssql-server

•	Create AG with 2 replicas and one config only node on primary node
CREATE AVAILABILITY GROUP [ag1]
   WITH (CLUSTER_TYPE = EXTERNAL)
   FOR REPLICA ON
      N'sqlhalx1' WITH (
         ENDPOINT_URL = N'tcp://sqlhalx1:5022',
         AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
         FAILOVER_MODE = EXTERNAL,
         SEEDING_MODE = AUTOMATIC
         ),
      N'sqlhalx2' WITH (
         ENDPOINT_URL = N'tcp://sqlhalx2:5022',
         AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
         FAILOVER_MODE = EXTERNAL,
         SEEDING_MODE = AUTOMATIC
         ),
      N'sqlhaconfig' WITH (
         ENDPOINT_URL = N'tcp://sqlhaconfig:5022',
         AVAILABILITY_MODE = CONFIGURATION_ONLY
         );
ALTER AVAILABILITY GROUP [ag1] GRANT CREATE ANY DATABASE;

•	Create login for pacemaker on all SQL Server nodes and grant these permissions 
USE [master]
GO
CREATE LOGIN pcuser with PASSWORD= '<password>';
GRANT ALTER, CONTROL, VIEW DEFINITION ON AVAILABILITY GROUP::ag1 TO pcuser;
GRANT VIEW SERVER STATE TO pcuser;

•	Run below commands on secondary replicas to join in the AG

ALTER AVAILABILITY GROUP [ag1] JOIN WITH (CLUSTER_TYPE = EXTERNAL);

ALTER AVAILABILITY GROUP [ag1] GRANT CREATE ANY DATABASE;

•	On primary create Db and backup

CREATE DATABASE [db1];
ALTER DATABASE [db1] SET RECOVERY FULL;
BACKUP DATABASE [db1]    TO DISK = N'/var/opt/mssql/data/db1.bak';

•	Add DB to AG

ALTER AVAILABILITY GROUP [ag1] ADD DATABASE [db1];

•	Verify DB is created in secondary

SELECT name FROM sys.databases WHERE name = 'db1';
GO
SELECT DB_NAME(database_id) AS 'database', synchronization_state_desc FROM sys.dm_hadr_database_replica_states;

**Setup Pacemaker Cluster**

**Prereq**

•	Install Pacemaker and agents

sudo yum install pacemaker pcs fence-agents-all resource-agents

•	Set the password for the default user that is created when installing Pacemaker and Corosync packages. Use the same password on all nodes.

sudo passwd hacluster

•	To allow nodes to rejoin the cluster after the restart, enable and start pcsd service and Pacemaker. Run the following command on all nodes.
sudo systemctl enable pcsd
sudo systemctl start pcsd
sudo systemctl enable pacemaker

•	Disable source destination check
aws ec2 modify-instance-attribute --instance-id i-xxxxx --no-source-dest-check
aws ec2 modify-instance-attribute --instance-id i-xxxxx --no-source-dest-check

•	In ~/.aws/config file in [default] section make sure you have below setting

profile=default 

**Create Pacemaker Cluster**
	            
•	Authorize Configuring Pacemaker Cluster Nodes on primary

sudo pcs host auth sqlhalx1 sqlhalx2 sqllxconfig -u hacluster

•	Create PCS Cluster on primary. Provide all node names

sudo pcs cluster setup pmclus sqlhalx1 sqlhalx2 sqllxconfig --force

•	Enable and Start PCS Cluster

sudo pcs cluster start --all
sudo pcs cluster enable --all

•	Install SQL Server resource agent for SQL Server. Run the following commands on all nodes. Reboot

sudo yum install mssql-server-ha

•	SQL Server login is created for pacemaker earlier, now save the credentials for the SQL Server login on all nodes

echo 'pcuser' >> ~/pacemaker-passwd
echo '<password>' >> ~/pacemaker-passwd
sudo mv ~/pacemaker-passwd /var/opt/mssql/secrets/passwd
sudo chown root:root /var/opt/mssql/secrets/passwd
sudo chmod 400 /var/opt/mssql/secrets/passwd # Only readable by root

•	configure and start stonith fencing using fence_aws agent
    

sudo pcs stonith create clusterfence fence_aws region=ap-southeast-2 pcmk_host_map="sqlhalx1:<Instance Id>;sqlhalx2:<Instance Id>;sqllxconfig:<Instance Id>" power_timeout=240 pcmk_reboot_timeout=480 pcmk_reboot_retries=4 --force
    
sudo pcs property set stonith-enabled=true

•	create cluster resource agent (mssql_ag)

sudo pcs resource create ag1 ocf:mssql:ag ag_name=ag1 promotable notify=true meta failure-time=60s --force 

•	create cluster resource agent (aws-vpc-move-ip)

   sudo pcs resource create lxlnsql ocf:heartbeat:aws-vpc-move-ip ip=<IP address> interface="eth0" routing_table=<rtb-xxx> op monitor timeout="30s" interval="60s" --force

•	Add Cluster Colocation Constraints.

sudo pcs constraint colocation add lxlnsql with master ag1-clone INFINITY with-rsc-role=Master --force

•	add cluster order/promote constraints
    
sudo pcs constraint order promote ag1-clone then start lxlnsql --force
    
•	set pcs cluster properties
    

sudo pcs resource update ag1 meta failure-timeout=60s
sudo pcs property set start-failure-is-fatal=true  
sudo pcs property set cluster-recheck-interval=75s

•	MOVE and clear colocation constraint

sudo pcs resource move ag1-clone --master sqlhalx2 && sleep 30 && sudo pcs resource clear ag1-clone

