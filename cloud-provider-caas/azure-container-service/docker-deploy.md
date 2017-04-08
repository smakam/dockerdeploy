**Summary:**  
The steps here deploys a voting application(https://github.com/docker/example-voting-app) in Azure container service. In this example, we will use Docker Swarm as the orchestrator.

**Azure container service:**  
Azure container service(ACS) is a managed container offering from Azure cloud. Following are some features of ACS:

 - Can be used with docker swarm, kubernetes or mesos.
 - ACS helps with cluster creation only. Orchestration and container creation is managed by individual orchestrators.
 - The interaction with the container cluster is done using native orchestration interface. For example, docker commands are used with docker swarm, kubectl is used with kubernetes.
 - Docker swarm supported currently in ACS is pre-swarm mode. This means we don't have services and also we cannot use compose v3 format. 
 - There is automatic load balancer integration for services that needs to be exposed externally.
 - Azure container registry integration available for ACS.

**Pre-requisites:**  

 - Azure cloud account
 - ssh key pair

**Step 1:**  
**Create cluster:**  
Using ACS service, we can create the cluster. Following are the inputs needed:

 - Orchestrator. Supported options are Docker swarm, Kubernetes, Mesos. I have used Docker Swarm for this example.
 - Master. We need to specify the number of Masters, dns name, username, public key for ssh. I have used Master count 1. For Master node, there is no option to specify node type. dnsname is used to access the cluster later on.
 - Agent. We need to specify the number and type of agent nodes. I have used agent count of 2 and type as standard A2.

ACS will create all necessary infrastructure including master node, agent node, load balancer and associated networking and storage constructs. ACS will also install Docker Swarm and the necessary Docker components to make a Swarm cluster between the master and agent nodes. 

**Step 2:**  
**Accessing the cluster:**  
We can ssh directly to master node and execute commands or setup a tunnel between the localhost and master node.  First, we need to find the domainname for the cluster and its visible as "FQDN" in container service under resource group.  
**Setting up a tunnel:**  
Use the following command to create tunnel and access it:

    ssh -i ~/.ssh/id_rsa -fNL 2375:localhost:2375 -p 2200 docker@<fqdn>
    export DOCKER_HOST=:2375

The above command creates a tunnel mapping Docker commands from localhost to master node. Azure uses port 2200 to expose connection to master node. 

Following command shows the 2 node cluster:

    $ docker info
    Containers: 5
     Running: 5
     Paused: 0
     Stopped: 0
    Images: 10
    Role: primary
    Strategy: spread
    Filters: health, port, dependency, affinity, constraint
    Nodes: 2
     swarm-agent-EBB8638B000002: 10.0.0.6:2375
      └ Status: Healthy
      └ Containers: 2
      └ Reserved CPUs: 0 / 1
      └ Reserved Memory: 0 B / 3.528 GiB
      └ Labels: executiondriver=<not supported>, kernelversion=3.19.0-65-generic, operatingsystem=Ubuntu 14.04.4 LTS, storagedriver=aufs
      └ Error: (none)
      └ UpdatedAt: 2017-03-26T05:33:16Z
     swarm-agent-EBB8638B000003: 10.0.0.7:2375
      └ Status: Healthy
      └ Containers: 3
      └ Reserved CPUs: 0 / 1
      └ Reserved Memory: 0 B / 3.528 GiB
      └ Labels: executiondriver=<not supported>, kernelversion=3.19.0-65-generic, operatingsystem=Ubuntu 14.04.4 LTS, storagedriver=aufs
      └ Error: (none)
      └ UpdatedAt: 2017-03-26T05:33:37Z

Following command shows Docker version running:

    $ docker --version
    Docker version 17.03.0-ce, build 3a232c8

**Step 3:**  
**Deploying votingapp:**  
Since ACS does not use Swarm mode and uses older Swarm, we need to use voting app in Docker compose version 2 format. I have used the docker compose file specified here(https://github.com/smakam/dockerdeploy/blob/master/azure-container-service/docker-voting-azure.yml).

I used the following command to deploy the votingapp:

    docker-compose -f docker-voting-azure.yml up -d

Following output shows the running services:

    $ docker ps
    CONTAINER ID        IMAGE                                         COMMAND                  CREATED             STATUS              PORTS                      NAMES
    6dce4dde6189        postgres:9.4                                  "docker-entrypoint..."   51 minutes ago      Up 51 minutes       10.0.0.7:32769->5432/tcp   swarm-agent-EBB8638B000003/azurecontainerservice_db_1
    5908d47f7f8e        docker/example-voting-app-worker:latest       "/bin/sh -c 'dotne..."   51 minutes ago      Up 51 minutes                                  swarm-agent-EBB8638B000002/azurecontainerservice_worker_1
    486009c882cf        docker/example-voting-app-vote:latest         "gunicorn app:app ..."   51 minutes ago      Up 51 minutes       10.0.0.7:5000->80/tcp      swarm-agent-EBB8638B000003/azurecontainerservice_vote_1
    38fb4084f986        tmadams333/example-voting-app-result:latest   "node server.js"         51 minutes ago      Up 51 minutes       10.0.0.6:5001->80/tcp      swarm-agent-EBB8638B000002/azurecontainerservice_result_1
    ff5d2745942b        redis:alpine                                  "docker-entrypoint..."   51 minutes ago      Up 51 minutes       10.0.0.7:32768->6379/tcp   swarm-agent-EBB8638B000003/azurecontainerservice_redis_1

**Step 4:**  
**Accessing the service:**  
With Swarm mode, it is easier to access the service using any node in the cluster. In older Swarm, there is no service IP and the services get exposed in individual agent nodes. 
For the votingapp, we need to expose ports 5000 and 5001 externally. First, we need to go to swarm-agent-lb and create health probes for ports 5000 and 5001. Then, we need to add load balancer rule for ports 5000 and 5001 mapping them to the corresponding health probe. Now the vote and result services can be accessed using swarm-agent-lb public IP at ports 5000 and 5001. 

**Reference:**  
https://docs.microsoft.com/en-in/azure/container-service/container-service-intro
https://azure.microsoft.com/en-in/services/container-service/

