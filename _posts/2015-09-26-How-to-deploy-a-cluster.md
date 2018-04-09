---
layout: post
title: Deprecated - How to deploy an cluster using OpenDaylight?
---

To deploy a cluster using OpenDaylight, you will need multiple VMs.

##Prerequisites
####OpenDaylight

* Maven 3.3.3 - [https://maven.apache.org/download.cgi](https://maven.apache.org/download.cgi)
* JDK 1.7 - [http://www.oracle.com/technetwork/java/javase/downloads/jdk7.html](http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html0)

####VM
In order to deploy VMs, I will use [Vagrant](https://www.vagrantup.com/).

* Vagrant 1.7.4 - [https://www.vagrantup.com/downloads.html](https://www.vagrantup.com/downloads.html)


####Container
In order to deploy containers, I will use [Docker](https://www.docker.com/).

* Docker version 1.8.2 - [https://docs.docker.com/installation/](https://docs.docker.com/installation/)

##Vagrant VM
I created a repo hosting a Vagrantfile spawning VMs with the following network characteristics:

* Adapter 1: NAT
* Adapter 2: Bridge "en0: Wi-Fi (AirPort)"
* Static IP address: 192.168.50.15X (X being the number of the node)
* Adapter type: paravirtualized 

These are the steps to follow:

``` 
$ git clone -b lithium https://github.com/adetalhouet/cluster-nodes.git 
$ cd cluster-nodes
$ export NUM_OF_NODES=3
$ vagrant up
```

After a few minutes, to make sure the VMs are correctly running, execute the following command in the cluster-nodes folder:

```
$ vagrant status
Current machine states:

node-1                    running (virtualbox)
node-2                    running (virtualbox)
node-3                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.

```

The credentials of the VMs are <br>
- user: vagrant <br>
- password: vagrant
<br><br>
Now we have tree VMs available at thos IP addresses:

* 192.168.50.151

```
$ ssh vagrant@192.168.50.151
vagrant@192.168.50.151's password: 
Welcome to Ubuntu 14.04.3 LTS (GNU/Linux 3.19.0-25-generic x86_64)

 * Documentation:  https://help.ubuntu.com/
vagrant@node-1:~$ 
```

* 192.168.50.152

```
$ ssh vagrant@192.168.50.152
vagrant@192.168.50.152's password: 
Welcome to Ubuntu 14.04.3 LTS (GNU/Linux 3.19.0-25-generic x86_64)

 * Documentation:  https://help.ubuntu.com/
Last login: Sat Sep 26 14:17:34 2015 from 192.168.50.1
vagrant@node-2:~$
```
* 192.168.50.153

```
$ ssh vagrant@192.168.50.153
vagrant@192.168.50.153's password: 
Welcome to Ubuntu 14.04.3 LTS (GNU/Linux 3.19.0-25-generic x86_64)

 * Documentation:  https://help.ubuntu.com/
Last login: Sat Sep 26 14:17:44 2015 from 192.168.50.1
vagrant@node-3:~$ 
```

##OpenDaylight distribution
Two options are feasible; in the end, what we need is the .zip containing the distribution. <br>

1. Depending on the distribution you want to have in your cluster, download the relevant project from the [OpenDaylight repository](https://git.opendaylight.org/gerrit/#/admin/projects/). <br> For example, if you want to test the Network Intent Composition project in a cluster, do the following:
    
    ```
    $ git clone https://git.opendaylight.org/gerrit/nic.git
    $ cd nic
    $ mvn clean install -DskipTests
    $ cd distribution/target/
    $ ls | grep nic-karaf
    nic-karaf-1.1.0-SNAPSHOT.jar
    nic-karaf-1.1.0-SNAPSHOT.tar.gz
    nic-karaf-1.1.0-SNAPSHOT.zip
    ```
    Here is the .zip file containing the distribution to deploy.
2. The second solution is to download the full, precompiled, OpenDaylight Lithium SR1 distribution from nexus:
    
    ```
    $ wget https://nexus.opendaylight.org/content/groups/public/org/opendaylight/integration/distribution-karaf/0.3.1-Lithium-SR1/distribution-karaf-0.3.1-Lithium-SR1.zip
    $ ls 
    distribution-karaf-0.3.1-Lithium-SR1.zip
    ```

##Deploying the cluster
To deploy the cluster using VMs, or containers, let's use the cluster-deployer develops by the OpenDaylight Integration team.

```
$ git clone https://git.opendaylight.org/gerrit/integration/test.git
$ cd test/tools/clustering/cluster-deployer/ 
$ ls -1
deploy.py
kill_controller.sh
remote_host.py
restart.py
templates/
```
You need the following: 

1. Your VMs/containers IP addresses: <br>
    192.168.50.151, 192.168.50.152, 192.168.50.153
    
2. Their credentials (must be the same for all the VMs/containers): <br>
    vagrant/vagrant
    
3. The path to the distribution to deploy:  <br>
    Either nic/distribution/target/nic-karaf-1.1.0-SNAPSHOT.zip or distribution-karaf-0.3.1-Lithium-SR1.zip
    
4. The cluster's configuration files: <br>
    Those can be find under the templates repository. A bunch of templates are available, we will focus only on the _multi-node-test_.
    
    ```
    $ cd templates/multi-node-test/
    $ ls -1
    akka.conf.template
    jolokia.xml.template
    module-shards.conf.template
    modules.conf.template
    org.apache.karaf.features.cfg.template
    org.apache.karaf.management.cfg.template
    ```

**Let's take a closer look at the configuration files**<br>

- <u>akka.conf.template</u>:
This one is responsible for the cluster management, using the [Akka framework](http://akka.io/). If you want to learn more about how it works, visit the Akka [cluster usage](http://doc.akka.io/docs/akka/snapshot/java/cluster-usage.html) page.

- <u>jolokia.xml.template</u>:
This file defines the feature jolokia, in order to deploy it in each node. Jolokia is an elegant and firewall-friendly JMX/JSON/HTTP bridge, which allows access to your MBeans over HTTP and returns their attributes in JSON.

- <u>module-shards.conf.template</u>:
I won't describe it as it contains a good explanation of what it does. <br>
As it is, it defines a bunch of shards that could be unnecessary for your use. Feel free to edit it as you want, but here is my advice: keep the _default_, _topology_, and _inventory_ shards.

```
$ cat module-shards.conf.template 
# This file describes which shards live on which members
# The format for a module-shards is as follows,
# {
#    name = "<friendly_name_of_the_module>"
#    shards = [
#        {
#            name="<any_name_that_is_unique_for_the_module>"
#            replicas = [
#                "<name_of_member_on_which_to_run>"
#            ]
#     ]
# }
#
# For Helium we support only one shard per module. Beyond Helium
# we will support more than 1
# The replicas section is a collection of member names. This information
# will be used to decide on which members replicas of a particular shard will be
# located. Once replication is integrated with the distributed data store then
# this section can have multiple entries.
#
--[cut]--
```

- <u>modules.conf.template</u>:
As the previous one, it contains a description:

```
$ cat modules.conf.template 
# This file should describe all the modules that need to be placed in a separate shard
# The format of the configuration is as follows
# {
#    name = "<friendly_name_of_module>"
#    namespace = "<the yang namespace of the module>"
#    shard-strategy = "module"
# }
#
# Note that at this time the only shard-strategy we support is module which basically
# will put all the data of a single module in two shards (one for config and one for
# operational data)
```

- <u>org.apache.karaf.management.cfg.template</u>:
This file contains a bunch of configuration for a node. There is no need to modify it.

- <u>org.apache.karaf.features.cfg.template</u>:
This one is responsible for the features you wish to have available in your environment. Here, you can also define features to be install by default.

```
$ cat org.apache.karaf.features.cfg.template
--[cut]--
#
# Comma separated list of features repositories to register by default
#
featuresRepositories = mvn:org.apache.karaf.features/standard/3.0.3/xml/features,mvn:org.apache.karaf.features/enterprise/3.0.3/xml/features,mvn:org.ops4j.pax.web/pax-web-features/3.1.4/xml/features,mvn:org.apache.karaf.features/spring/3.0.3/xml/features,mvn:org.opendaylight.integration/features-integration-index/0.3.1/xml/features

#
# Comma separated list of features to install at startup
#
featuresBoot=config,standard,region,package,kar,ssh,management,odl-restconf,odl-clustering-test-app,odl-netconf-connector-ssh
--[cut]--
```

###Example
If you want to use the OpenDaylight Lithium SR1 distribution, the _featuresRepositories_ is correctly configured. If you wish specifying default boot features, those can be added to the _featuresBoot_ definition.

Deploy your cluster:

```
$ pwd
test/tools/clustering/cluster-deployer
# the deploy script will prepare the configuration files view 
# above for the specified IP addresses, and it needs to store 
# them in a temp/ folder.
$ mkdir temp 
$ python deploy.py --clean --distribution=../../../../distribution-karaf-0.3.1-Lithium-SR1.zip --rootdir=/home/vagrant --hosts=192.168.50.151,192.168.50.152,192.168.50.153 --user=vagrant --password=vagrant --template=multi-node-test --clean
```

The deploy.sh script will log the following during the process if everything went fine: [log dump](https://gist.github.com/8af7eff94663b68573c6)
<br>
It will also start the controller on each nodes. You could, for instance, get into the node-3 and have a look at the karaf.log to know what happened. Here is a [hint](https://gist.github.com/c232b4232e40469eabd6).

FYI, here is a tree dump of my file explorer with only the relevant data:

```
$ tree
.
├── cluster-nodes
│   ├── LICENSE
│   ├── README.md
│   ├── Vagrantfile
│   └── puppet
│       ├── manifests
│       │   └── base.pp
│       └── scripts
│           └── bootstrap.sh
├── distribution-karaf-0.3.1-Lithium-SR1.zip
└── test
    └── tools
        └── clustering
            └── cluster-deployer
                ├── deploy.py
                ├── kill_controller.sh
                ├── remote_host.py
                ├── remote_host.pyc
                ├── restart.py
                ├── temp
                │   ├── akka.conf
                │   ├── jolokia.xml
                │   ├── module-shards.conf
                │   ├── modules.conf
                │   ├── org.apache.karaf.features.cfg
                │   └── org.apache.karaf.management.cfg
                └── templates
                    └── multi-node-test
                        ├── akka.conf.template
                        ├── jolokia.xml.template
                        ├── module-shards.conf.template
                        ├── modules.conf.template
                        ├── org.apache.karaf.features.cfg.template
                        └── org.apache.karaf.management.cfg.template
```

## Verify the deployment
To verify the deployment, we're going to use Jolokia to read an MBean on one of our nodes: <br>

Here is the request made:

- GET
- node-1 (192.168.50.151)
- shard: inventory
- data store: config

Here is what we get, among other things:

- the shard leader: "member-2-shard-inventory-config
- the peer addresses: "member-2-shard-inventory-config: akka.tcp://opendaylight-cluster-data@192.168.50.152:2550/user/shardmanager-config/member-2-shard-inventory-config, member-3-shard-inventory-config: akka.tcp://opendaylight-cluster-data@192.168.50.153:2550/user/shardmanager-config/member-3-shard-inventory-config"
- the state of the node in the cluster: "Follower"

```
$ curl -v --user "admin":"admin" -H "Accept: application/json" -H "Content-type: application/json" -X GET http://192.168.50.151:8181/jolokia/read/org.opendaylight.controller:Category=Shards,name=member-1-shard-inventory-config,type=DistributedConfigDatastore | python -m json.tool
*   Trying 192.168.50.151...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 192.168.50.151 (192.168.50.151) port 8181 (#0)
* Server auth using Basic with user 'admin'
> GET /jolokia/read/org.opendaylight.controller:Category=Shards,name=member-1-shard-inventory-config,type=DistributedConfigDatastore HTTP/1.1
> Host: 192.168.50.151:8181
> Authorization: Basic YWRtaW46YWRtaW4=
> User-Agent: curl/7.43.0
> Accept: application/json
> Content-type: application/json
> 
< HTTP/1.1 200 OK
< Content-Type: text/plain;charset=UTF-8
< Cache-Control: no-cache
< Pragma: no-cache
< Date: Sat, 26 Sep 2015 15:33:50 GMT
< Expires: Sat, 26 Sep 2015 14:33:50 GMT
< Content-Length: 1415
< Server: Jetty(8.1.15.v20140411)
< 
{ [1415 bytes data]
100  1415  100  1415    0     0   154k      0 --:--:-- --:--:-- --:--:--  172k
* Connection #0 to host 192.168.50.151 left intact
{
    "request": {
        "mbean": "org.opendaylight.controller:Category=Shards,name=member-1-shard-inventory-config,type=DistributedConfigDatastore",
        "type": "read"
    },
    "status": 200,
    "timestamp": 1443281630,
    "value": {
        "AbortTransactionsCount": 0,
        "CommitIndex": 2,
        "CommittedTransactionsCount": 0,
        "CurrentTerm": 1,
        "FailedReadTransactionsCount": 0,
        "FailedTransactionsCount": 0,
        "FollowerInfo": [],
        "FollowerInitialSyncStatus": true,
        "InMemoryJournalDataSize": 184,
        "InMemoryJournalLogSize": 1,
        "LastApplied": 2,
        "LastCommittedTransactionTime": "1970-01-01 00:00:00.000",
        "LastIndex": 2,
        "LastLeadershipChangeTime": "2015-09-26 15:17:53.227",
        "LastLogIndex": 2,
        "LastLogTerm": 1,
        "LastTerm": 1,
        "Leader": "member-2-shard-inventory-config",
        "LeadershipChangeCount": 1,
        "PeerAddresses": "member-2-shard-inventory-config: akka.tcp://opendaylight-cluster-data@192.168.50.152:2550/user/shardmanager-config/member-2-shard-inventory-config, member-3-shard-inventory-config: akka.tcp://opendaylight-cluster-data@192.168.50.153:2550/user/shardmanager-config/member-3-shard-inventory-config",
        "PendingTxCommitQueueSize": 0,
        "RaftState": "Follower",
        "ReadOnlyTransactionCount": 0,
        "ReadWriteTransactionCount": 0,
        "ReplicatedToAllIndex": 1,
        "ShardName": "member-1-shard-inventory-config",
        "SnapshotCaptureInitiated": false,
        "SnapshotIndex": 1,
        "SnapshotTerm": 1,
        "StatRetrievalError": null,
        "StatRetrievalTime": "301.2 \u03bcs",
        "VotedFor": "member-2-shard-inventory-config",
        "WriteOnlyTransactionCount": 0
    }
}
```

##References
  * [Deployment of 3 Nodes Cluster](https://wiki.opendaylight.org/view/CrossProject:Integration_Group:Controller-Cluster_Deployment#Deployment_of_3_Node_Cluster)
  * [Cluster deployer](https://git.opendaylight.org/gerrit/gitweb?p=integration/test.git;a=tree;f=tools/clustering/cluster-deployer;h=a11fc3d5526d96b4a202411df7727310ea493580;hb=HEAD)
  * [OpenDaylight wiki page presenting the clustering architecture](https://wiki.opendaylight.org/view/OpenDaylight_Controller:MD-SAL:Architecture:Clustering)