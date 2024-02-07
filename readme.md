## IDEAS
To implement 3 Mariadb instances having its own storage and ensure all data are synchronized, we can use the model of Master-Slave Replication where Master is a place to allow write and/or read, Slave is to be responsible to read-related operations
To do that, we need the LB and proxy (MaxScale - RWSplit) stay in front of them to extract their rrequests to our service and the Monitor to detect when master is down, 1 of 2 remaining node places to be a master. Once the old master is recovered, it becomes the slave replicating all the new data during the period of malfunction

1/ Minikube
- Using minikube to spin up the K8s cluster

2/ 3 Mariadb instances with the kind of StatefulSet
- StatefulSet: Ensure each instance has its own storage to save data
- Using the mechanism of Master-Slave to syn up data between 3 instances
- All 3 instances are under of MaxScale
- MaxScale plays a role of LB, proxy to transfer accordingly the read and operation to slave and master. However, MaxScale has the functionality of automatic failover when the master is down.

3/ How to do (all of yaml files are attached):
- Create 3 instances with the StatefulSet of 3 replicas
    + On the configmap: create the configuration for 3 engines and the init database queries script (including the user  'repluser' on slave to sync up with master, the user MaxScale having the enough priviledges to work with the Master DB, the admin user for Monitor failover, the databse configuration with the model of Master-Slave replication) (mariadb-configmap)
    + Create the StatefulSet with the Service (mariadb-service)
- Create 1 instance of MaxScale with the Deployment
    + On the configmap: Create the MaxScale functionality of Monitor, and enable the atribute of auto_failover and auto-rejoin (mariadb-maxscale) and RWSplitRouter to balance the read-heavy and write flow
    + Create the Deployment of MaxScale under of Service mariadb-service-lb-rw

4/ How to test:
4.1/ Test the replication
- Using the admin account to access database
- Jump into the master-db pod, create the database (demo) with some samples 
- jump into one of salve databases, run the command SHOW DATAABASES; SELECT *FROM <TABLE>

4.2/ Test the routing from outside (Model: Client -> MaxScale -> DB Cluster)
- Because of using minikube, create the tunnel between the host to the K8s cluster with the service mariadb-service-lb-rw
- Run the command: 'mysql -h<IP-exposed> -P<Port-Exposed> -u<user> -p<pass>' (with admin credential)
- Run the command SHOW DATAABASES; SELECT *FROM <TABLE>
- Jump into the MaxScale pod, run command 'maxctrl show service "RWSplitRouter"' and see the writes are issued on Master

4.3/ Test the failover
- Currently, set the sts pod mariadb-sts-2 as master
- Run the command: 'maxctrl list servers;' to see which one is the master or slave
- Scale down the number of sts replicas to 2, then failover is occured to turn the mariadb-sts-0 or mariadb-sts-1 to master. Run the command: 'maxctrl list servers;' to see which one is the master or slave
- Scale up to 3 replicas again, the mariadb-sts-2 is spined up with the same GTID of 2 others. 

Reference
- https://cloudinfrastructureservices.co.uk/setup-mariadb-replication/
- https://www.bing.com/search?pglt=2211&q=maradb+replication&cvid=a598603375db4f02b93654985bdb3437&gs_lcrp=EgZjaHJvbWUqBggAEEUYOzIGCAAQRRg7MgYIARBFGDkyBggCEAAYQDIGCAMQABhAMgYIBBAAGEAyBggFEAAYQDIGCAYQLhhAMgYIBxBFGDwyBggIEEUYPNIBCDI5MTFqMGoxqAIAsAIA&FORM=ANNTA1&PC=U531
- https://github.com/mariadb-corporation/MaxScale
