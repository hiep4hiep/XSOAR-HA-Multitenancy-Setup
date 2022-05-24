# XSOAR-HA-Multitenancy-Setup
This is the guide to set up XSOAR in HA and Multitenancy environment
As we have already known, the newly released Cortex XSOAR 6.1 (Feb 15, 2021) supports some significant changes regarding architecture and functionality. One of the most relevant changes to enterprise deployment of XSOAR is High availability design. Good news for enterprises who require a high resilient capability, XSOAR 6.1 can do it much better than the older version.
You can find more information in the release note (https://docs.paloaltonetworks.com/cortex/cortex-xsoar/6-1/cortex-xsoar-release-notes/cortex-soar-release-information/cortex-soar-new-features.html#id3a34611b-f25c-4a0d-a80c-d850a1bc194a)

![image](https://user-images.githubusercontent.com/41276379/169926950-19a97e27-c027-4839-b4e2-cc5fd41bbc63.png)

In this guide, I put all together the new deployment architecture for High availability and Multi-tenancy. If you don’t have multi-tenancy requirement, just ignore the “host” part and multi-tenant flag when installing the system.

## Architecture guide
Now, it seems to be a lot of components, right? Let’s explain the functionality of each component and internal communication from top to bottom of the diagram:

User: will access XSOAR via Load Balancer IP address instead of directly to each XSOAR application server. That will be the way XSOAR servers deliver high availability for end user side, just as simple as Front end 2-3 tiers web application. 

Load Balancer: can be any L4 – L7 LB with health check and some basic to advance load balancing methods. In my lab, I use the native LB of the cloud provider that I used to host the lab.

XSOAR App: in this HA design, the application and database will be separated. XSOAR App can be 2 or 3 servers behind a Load balancer. 
The most important part is where XSOAR data (e.g license, settings, playbooks…) can be shared amongst the XSOAR apps? The answer is via NFS mounted folder. And here it is.
<img width="745" alt="image" src="https://user-images.githubusercontent.com/41276379/169926997-f5e4a989-6d90-47a1-ba54-aae41f65b446.png">

NFS: technically, NFS can be installed on one XSOAR app and exported (shared) to another XSOAR app to consume. But it is recommended to install a standalone NFS with regular backup to host the content of “/var/lib/demisto” folder for all XSOAR app. 
Then XSOAR app servers will need to mount it’s /var/lib/demisto folder to the NFS so that they can share the content. If XSOAR admin create a playbook on XSOAR app 1, it will be consumed by XSOAR app 2 as well, without any human intervention.

XSOAR host: this one is designed specifically for multi-tenancy deployment where XSOAR host will be the server putting close to end customer environment. 
XSOAR host will have multiple tenants configured and most of the workload will be running on the host, of course. That’s why in Palo Alto Networks document, they recommend the sizing should be number of tenants multiplies with recommended XSOAR CPU & memory (e.g 3 tenants x 16 CPU & 32GB RAM = 48 CPU & 96GB RAM)

<img width="763" alt="image" src="https://user-images.githubusercontent.com/41276379/169927020-27ec13b7-ac2e-4b9b-8963-4f4df24c1bb2.png">

XSOAR host connects to XSOAR app by TCP port 443, direction in the above diagram.

Elastic Search: ES is the main database (which stores incident data, context data…) and is much better in performance comparing with the legacy BoltDB. ES support clustering natively with minimum of 2 nodes (recommended 3 for highest resiliency). 

<img width="752" alt="image" src="https://user-images.githubusercontent.com/41276379/169927052-12616bd8-cb8c-414b-a06e-210df6b3327d.png">

You can read more about ES clustering on ElasticSearch website, but they key things to remember is that one cluster contains Master and Data node. And in this deployment, make sure all nodes are master-eligible and data node, that’s it.
XSOAR app and host will read/write to ElasticSearch via API (HTTP port 9200 by default).

## Start installing NFS
We will install the components from the bottom up because XSOAR app needs the NFS and Database ready.
There is plenty of options for NFS, either on Windows, Linux or any public cloud native NFS services. In this guide, we will install a new one on Linux.

<img width="749" alt="image" src="https://user-images.githubusercontent.com/41276379/169927097-85823afb-1909-4875-b57d-9d03ca1c55cc.png">


**Step 1**: have yourself a clean Linux server. In this guide, I use Ubuntu 18.04
**Step 2**: install NFS server
```
sudo apt update
sudo apt install nfs-kernel-server
```

**Step 3**: create a folder to share
XSOAR app with need to have read & write access to /var/lib/demisto (which is mounted to NFS folder). You can create any folder on NFS to share for this purpose, but I recommended to create an exact name to be easier to manage.
```
sudo mkdir /var/lib/demisto -p
```

**Step 4**: change owner to nobody:nogroup
As NFS will translate any root operation on client to nobody:nogroup credential, we need to chown the directory:
```
sudo chown nobody:nogroup /var/lib/demisto
```

**Step 5**: share the folder for XOAR app server access
Sudo vi /etc/exports
Then add one line to the file for the export purpose
```
/var/lib/demisto    172.17.2.4(rw,sync,no_root_squash,no_subtree_check) 172.17.2.6(rw,sync,no_root_squash,no_subtree_check)
```

In this setup, 172.17.2.4 and 172.17.2.6 are XSOAR App 01 and XSOAR App 02 server. Make sure the no_root_squash is added the the attribute list because it is important for XSOAR app credential to write to this folder.

**Step 6**: restart the NFS service and check status
`systemctl restart nfs-kernel-server`
Check your nfs-kernel-server status is running and use netstat to check if TCP 2049 is now listening on NFS server.

## Installing ElasticSearch cluster
This is apparently no need the full Elastic Stack (ELK) here because we only need ES cluster for the database. Cortex XSOAR will read/write to ElasticSearch via API so no need ingestion or report function.
As described on ElasticSearch documentation, we will need at least 3 nodes (servers) to deliver resiliency to the cluster. The setting of these 3 server will be similar, so let’s jump to the 1st one.
For your information, ElasticSearch includes 4 different types of nodes:
· Data nodes — stores data and executes data-related operations such as search and aggregation
· Master nodes — in charge of cluster-wide management and configuration actions such as adding and removing nodes
· Client nodes — forwards cluster requests to the master node and data-related requests to data nodes
· Ingest nodes — for pre-processing documents before indexing
And please be noted that we will use only Master node and Data node in our cluster with XSOAR.

Step 1: have yourself a clean Linux server. In this guide, I use Ubuntu 18.04
Step 2: Install Java
```
sudo apt-get update
sudo apt-get install default-jre
```
And check java version
```
#java --version
openjdk 11.0.10 2021-01-19
OpenJDK Runtime Environment (build 11.0.10+9-Ubuntu-0ubuntu1.18.04)
OpenJDK 64-Bit Server VM (build 11.0.10+9-Ubuntu-0ubuntu1.18.04, mixed mode, sharing)
```
Step 3: Import ElasticSearch PGP key
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add –
```

Step 4: Install apt-transport-https to get from ES server
```
sudo apt-get install apt-transport-https
```

Step 5: Add ES 7.x repo
```
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list
```

Step 6: Install ElasticSearch
```
sudo apt-get update && sudo apt-get install elasticsearch
```
Step 7: configure ElasticSearch cluster parameter
Please make sure you finish this step before starting your ElasticSearch service.
```
sudo vi /etc/elasticsearch/elasticsearch.yml
```

Add these lines to your configuration file on Node-1
```
#Set cluster name identical in all of the 3 servers
cluster.name: xsoar-db
#Node name will be node-1, node-2 and node-3
node.name: node-1
#Enable all node to be eligible for master
node.master: true
# Enable all node to be data
node.data: true
#
# ---------------------------------- Network -----------------------
#
#IP address of the node
network.host: 172.17.4.4
#
#Leave default port 9200
http.port: 9200
#
# --------------------------------- Discovery ----------------------
#Add list of 3 nodes IP here for discovery when ES starts
discovery.seed_hosts: ["172.17.4.4", "172.17.4.5", "172.17.4.3"]
#
#Select master nodes
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
#
```

Add these lines to your configuration file on Node-2

```
#Set cluster name identical in all of the 3 servers
cluster.name: xsoar-db
#Node name will be node-1, node-2 and node-3
node.name: node-2
#Enable all node to be eligible for master
node.master: true
# Enable all node to be data
node.data: true
#
# ---------------------------------- Network -----------------------
#
#IP address of the node
network.host: 172.17.4.5
#
#Leave default port 9200
http.port: 9200
#
# --------------------------------- Discovery ----------------------
#Add list of 3 nodes IP here for discovery when ES starts
discovery.seed_hosts: ["172.17.4.4", "172.17.4.5", "172.17.4.3"]
#
#Select master nodes
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
#
```

Add these lines to your configuration file on Node-3

```
#Set cluster name identical in all of the 3 servers
cluster.name: xsoar-db
#Node name will be node-1, node-2 and node-3
node.name: node-3
#Enable all node to be eligible for master
node.master: true
# Enable all node to be data
node.data: true
#
# ---------------------------------- Network -----------------------
#
#IP address of the node
network.host: 172.17.4.3
#
#Leave default port 9200
http.port: 9200
#
# --------------------------------- Discovery ----------------------
#Add list of 3 nodes IP here for discovery when ES starts
discovery.seed_hosts: ["172.17.4.4", "172.17.4.5", "172.17.4.3"]
#
#Select master nodes
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
#
```
Step 8: Start ElasticSearch
```
sudo service elasticsearch start
```

Step 9: Verify the service
Make sure the service is up and running
```
systemctl status elasticsearch
```

Then verify your cluster status
```
curl -XGET 'http://localhost:9200/_cluster/state?pretty'
```
You should get similar output
```
{
  "cluster_name" : "xsoar-db",
  "cluster_uuid" : "v0cPjnYDTFirhLLJFWAnrw",
  "version" : 404,
  "state_uuid" : "mNoaeOarR46GrjYflLEA4A",
  "master_node" : "M6c0B5AkT_y50si1Rtrb6A",
  "blocks" : { },
  "nodes" : {
    "nvydIDDtS4O5ODAOZgFGJQ" : {
      "name" : "node-2",
      "ephemeral_id" : "M5jnrwd4SauGcZgtTs7bfg",
      "transport_address" : "172.17.4.5:9300",
      "attributes" : {
        "ml.machine_memory" : "8349163520",
        "ml.max_open_jobs" : "20",
        "xpack.installed" : "true",
        "ml.max_jvm_size" : "1073741824",
        "transform.node" : "true"
      }
    },
    "M6c0B5AkT_y50si1Rtrb6A" : {
      "name" : "node-1",
      "ephemeral_id" : "7ar0kYPtS_CxSSrmC4Q7YQ",
      "transport_address" : "172.17.4.4:9300",
      "attributes" : {
        "ml.machine_memory" : "8349163520",
        "xpack.installed" : "true",
        "transform.node" : "true",
        "ml.max_open_jobs" : "20",
        "ml.max_jvm_size" : "1073741824"
      }
    }
  },
```

## Install XSOAR Application
Next part will be installing the XSOAR Application server. At this stage, I think everything is straightforward to you as we have already had:
- NFS: to store XSOAR content and setting data on /var/lib/demisto shared folder
- ElasticSearch: as XSOAR’s database to store mostly incident data

So before install XSOAR application, we will need to make sure XSOAR server can access to these services. 
**Step 1**: have yourself a clean Linux server. In this guide, I use Ubuntu 18.04
**Step 2**: Setup NFS client
We need to do this because XSOAR App server will be NFS client to mount and access shared folder on NFS server.
```
sudo apt update
sudo apt install nfs-common
```
**Step 3**: Create /var/lib/demisto folder and mount to NFS
```
sudo mkdir -p /var/lib/demisto
sudo mount 172.17.2.7:/var/lib/demisto /var/lib/demisto
```
(the 1st :/var/lib/demisto is the source path from NFS server, the 2nd /var/lib/demisto is the local folder on XSOAR server)
And check if it’s there
```
xsoar-app1:~$ df -h
Filesystem                   Size  Used Avail Use% Mounted on
udev                         3.9G     0  3.9G   0% /dev
tmpfs                        797M  784K  796M   1% /run
/dev/sda1                     29G   11G   19G  37% /
tmpfs                        3.9G     0  3.9G   0% /dev/shm
tmpfs                        5.0M     0  5.0M   0% /run/lock
tmpfs                        3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda15                   105M  6.1M   99M   6% /boot/efi
172.17.2.7:/var/lib/demisto   29G  6.8G   23G  24% /var/lib/demisto
```

Make it survives boot by adding to /etc/fstab

`172.17.2.7:/var/lib/demisto    /var/lib/demisto   nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0`


**Step 4**: Install the XSOAR application
Now it’s good to install the XSOAR application. Just follow below simple steps with internet connection and the server will be installed in about 10-15 mins.
```
wget -O demisto.sh "[direct download link]"

chmod +x demisto.sh
```

For multi-tenancy deployment
```
sudo ./demisto.sh -- -y -multi-tenant -elasticsearch-url=http://172.16.4.3:9200,http://172.16.4.4:9200,http://172.16.4.5:9200
```

For single-tenancy deployment
```
sudo ./demisto.sh -- -y -elasticsearch-url=http://172.16.4.3:9200,http://172.16.4.4:9200,http://172.16.4.5:9200
```

(if you have another load balancer for Elasticsearch, put the LB IP address as -elasticsearch-url value)

**Step 5**: Verify the installation
```
xsoar-app1:~$ systemctl status demisto
● demisto.service - Demisto Server Service
   Loaded: loaded (/etc/systemd/system/demisto.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2021-03-14 22:53:16 UTC; 6h ago
 Main PID: 1273 (server)
    Tasks: 23 (limit: 4915)
   CGroup: /system.slice/demisto.service
 ├─1273 /usr/local/demisto/server
 └─3936 docker run -i --rm --name demistoserver_pyexec-6ef6ce82-023d-4f39-8dce-e827679d5826-demistopython1.3-alpine--1 --env HTTP_PROXY= --env http_proxy= --env HTTPS_PROXY= --env https_proxy= --log-drive
 ```

If the status is Stopped (not running), most likely the problem came from permission issue of the /var/lib/demisto.
You can fix with
```
sudo chown -R demisto:demisto /var/lib/demisto
sytemctl start demisto
```


## Multi tenancy set up
In multi-tenancy deployment, there are some terms that we need to clarify:
<img width="407" alt="image" src="https://user-images.githubusercontent.com/41276379/169927453-42e921bb-3db4-45d8-95b8-a897690dbfda.png">

- Master: it sit on the XSOAR app server literally. From Master account, you can manage all your multi-tenancy deployment, including host and tenant.
- Host: is one or multiple servers deployed at end customer side. 
For example, if you are a MSSP and you have 3 managed customers at 3 different locations, you will need to install at least 1 Host server at each customer’s location.
In another scenario, if you manage 3 customers in your location, you can just install 1 Host (2 for high availability) and create 3 tenants on that host only.
- Tenant: is a virtual unit that define perimeter between managed customer. Tenant is created on each Host or HA group of hosts.
- HA Group is a logical group of multiple host as a cluster. HA Group can provide redundancy and performance improvement by load balancing the request to different member of the HA Group. It's really good, undoubtedly. So no reason why we don't use this feature, even if you have single Host server, it's good to have HA Group in place for the future expansion.

<img width="770" alt="image" src="https://user-images.githubusercontent.com/41276379/169927479-f9e12ca4-6c46-4c28-87e1-7731ea397336.png">
**Step 1:** If you have only one App Server, you can ignore this step. 
This step is for company who has 2+ App servers working in HA by a Load balancer (refer to this architecture guide). The XSOAR Host will connect back to XSOAR Apps by hostname/IP address:443. So in HA deployment, we need to instruct the XSOAR Host to connect to the Load Balancer domain name or IP.
To do this, log in to each of the XSOAR App host > Settings > About > Troubleshooting > Modify the Base URL and External Hostname to your loadbalancer IP sitting in front of the XSOAR Apps.
<img width="561" alt="image" src="https://user-images.githubusercontent.com/41276379/169927516-5803d44b-8223-4296-8657-bb4860760ae4.png">


**Step 2:** Log in to your XSOAR App server > Settings > Account Management > Host > New Host/HA Group
If you have multiple host under a HA group (for redundancy and load sharing), choose HA Group. Else, choose New Host.
<img width="755" alt="image" src="https://user-images.githubusercontent.com/41276379/169927560-064b6a95-ed3f-41ca-8036-5398cc1ac12a.png">

**Step 3:** Fill in installation information required then click Create and Download Installer
<img width="416" alt="image" src="https://user-images.githubusercontent.com/41276379/169927586-e855b1a8-35d0-4afe-bc9e-031ae6dc7ec4.png">

You will get a demisto-xxxx.sh file downloaded to your computer.

**Step 4**: Install host
- You just need a clean installed Linux server with internet connection and network connection back to your main XSOAR app server (TCP 443).
- But there will be something to prepare as below sub steps if you choose to have multiple Host in the same HA Group. 
<img width="636" alt="image" src="https://user-images.githubusercontent.com/41276379/169927647-7bb9ca5c-c665-4060-9834-acb3ea505ea8.png">

- But there will be something to prepare as below sub steps if you choose to have multiple Host in the same HA Group. 
Step 4.1: Setup NFS client
We need to do this because XSOAR Host server will be NFS client to mount and access shared folder on NFS server.
```
sudo apt update
sudo apt install nfs-common
```
Step 4.2: Create /var/lib/demisto folder and mount to NFS
```
sudo mkdir -p /var/lib/demisto2
sudo mount 172.17.2.7:/var/lib/demisto2 /var/lib/demisto
```

And check if it’s there
```
xsoar-host11:~$ df -h
Filesystem                   Size  Used Avail Use% Mounted on
udev                         3.9G     0  3.9G   0% /dev
tmpfs                        797M  784K  796M   1% /run
/dev/sda1                     29G   11G   19G  37% /
tmpfs                        3.9G     0  3.9G   0% /dev/shm
tmpfs                        5.0M     0  5.0M   0% /run/lock
tmpfs                        3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda15                   105M  6.1M   99M   6% /boot/efi
172.17.2.7:/var/lib/demisto2   29G  6.8G   23G  24% /var/lib/demisto
```

Step 4.3: now you can install with your .sh file downloaded from Master server
- Then upload the .sh file you got to a new Host server.
- Execute the .sh file to install
```
chmod +x demisto.sh
sudo ./demisto.sh -- -y
```
- After the host was installed, it will connect back to XSOAR app server automatically, you don’t need to do anything. If the host belongs to a HA Group, it can be automatically registered under that HA Group as well.
- If you do not see Host registered on XSOAR app server, the issue might be on network connectivity. So check if XSOAR app servers and XSOAR hosts can resolve each other’s hostname (use DNS or edit /etc/hosts file), and also check if any Firewall is blocking the connection between Host and Master server.

**Step 5**: Verify the Host installation
If everything is put correctly, you can see host under HA group on XSOAR or standalone host if you choose to deploy host before.
<img width="756" alt="image" src="https://user-images.githubusercontent.com/41276379/169927729-d4f95b8f-1fa7-408b-8c5f-194a09ddac93.png">
If the status is Offline and cannot see other parameter, check your hostname accessibility between the Master server and Hosts.
Also check your Main Hosts tab and you can see 2 XSOAR App that we installed will be shown:
<img width="758" alt="image" src="https://user-images.githubusercontent.com/41276379/169927769-cae46152-c890-4508-8915-c6278a82c8a2.png">

**Step 6:** Create tenant
There’s a bit confusing here as they do not have Tenant wording in Cortex XSOAR, but it is actually “Account”. So have in mind “Account” is not a user/admin/analyst account, but it is a Tenant. 
Go to Account Management > Accounts > Add Account
<img width="355" alt="image" src="https://user-images.githubusercontent.com/41276379/169927795-613b78c2-53b9-4308-8607-34fb16cc18a6.png">
Then check your result
<img width="754" alt="image" src="https://user-images.githubusercontent.com/41276379/169927824-16235b48-1108-4f38-bf25-bcb998d01e75.png">

After creating accounts, you can switch to different Tenant to manage by clicking the Top left at Main Account.
<img width="766" alt="image" src="https://user-images.githubusercontent.com/41276379/169927862-3202031b-a66c-4082-82ad-da5fd6a6a8e0.png">
