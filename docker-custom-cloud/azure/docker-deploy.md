**Summary:**  
The steps here can be used to create Swarm mode cluster in Azure cloud using "Docker for Azure" and it also deploys example-voting-app(https://github.com/docker/example-voting-app) to Azure cloud. Docker CE version runs in the Azure cloud using these steps. 
"Docker for Azure" is Docker's attempt to make Docker containers Azure cloud friendly. Azure template is used to create the necessary resources. 

Following are some integrations that Docker has added as part of this offering:

 - Master and worker nodes along with Swarm mode cluster gets automatically created.
 - Firewalls are automatically created to expose the necessary ports needed for Swarm mode and ssh. Only master node is exposed to outside world. 
 - Load balancer gets automatically created when new Docker services are created and exposes the services. 
 - Logs are automatically directed to Azure logger service.
 - Azure infrastructure internals like virtual network, load balancer gets automatically created. 

**Video:**  
[![Docker custom cloud](https://github.com/smakam/dockerdeploy/blob/master/images/dockercustomcloud.jpg)](https://www.youtube.com/watch?v=l2Yy8poceuY")

**Pre-requisities:**
 - Azure cloud account 
 - Azure subscription ID
 - Azure ssh key needs to be created

**Step 1:**  
**Setup Azure Infra**  
"Service principal" is a concept used in Azure to that allows invoking Azure API with needed permissions. It is like IAM concept in AWS. The first step is to create Service principal and then use the credentials given by the Service principal to invoke Azure template. 
I used the following command to create the Service principal:

    docker run -ti docker4x/create-sp-azure sp-mydockerswarm rg-mydockerswarm southindia

"sp-mydockerswarm" is the service principal name, "rg-mydockerswarm" is the resource group, "southindia" is the region. This container will request for Azure authentication and also prompt for Azure subscription ID. 
The following output will be displayed on successful creation of service principal:  

    Your access credentials ==================================================
    AD ServicePrincipal App ID:       ....
    AD ServicePrincipal App Secret:   .....
    AD ServicePrincipal Tenant ID:    .....
    Resource Group Name:              rg-mydockerswarm
    Resource Group Location:          southindia

**Step 2:**  
**Create cluster**  
There is a link here(https://docs.docker.com/docker-for-azure/#docker-community-edition-ce-for-azure) to invoke the Azure template. We can choose either the "stable" or "edge" Docker CE versions. The "edge" version has latest features that are not fully tested. For this example, I have chosedn the "edge" version. The links corresponding to the version will start with appropriate Azure template. 
Following are some major inputs to be specified as part of Azure template:

 - App id, App secret, resource group name, resource group location. This is got from step 1 above. 
 - Number of master nodes. (I used 1)
 - Instance type of master nodes.(I used Standard.A2)
 - ssh key to ssh to master node.(I used existing key and pasted it)
 - Number of worker nodes (I used 2)
 - Instance type of worker nodes. (I used Standard.A2)

Following output shows the 3 node cluster created for me:

        $ docker node ls
    ID                           HOSTNAME             STATUS  AVAILABILITY  MANAGER STATUS
    kw7u4f1ao00wkitvlr2i3qds9 *  swarm-manager000000  Ready   Active        Leader
    obi4741qebrusidgc5wqzcefe    swarm-worker000001   Ready   Active        
    quvgd3cf3lckc64z3u9qgvfxi    swarm-worker000000   Ready   Active 

From outside world, ssh is allowed only to the master node. From master node, we can ssh to worker nodes using the private network. 
Azure uses a different port to expose ssh. Following command can be used to ssh to master node:

    ssh -i ~/.ssh/id_rsa -p 50000 docker@<masterip>
   
**Step 3:**  
**Deploy application:**  
I used the following command to scp the compose file from my machine to Master node:

    scp -i ~/.ssh/id_rsa -P 50000 docker-stack.yml docker@<ip>:

Following command deploys the application:

    docker stack deploy --compose-file docker-stack.yml vote

Since this application has 3 services that exposes ports 8080, 5000 and 5001, Load balancer gets automatically created in Azure which is mapped to the Docker services. The 3 services can be accessed using load balancer domainname/IP address.

Following output shows the running services:

    $ docker stack ps vote
    ID            NAME               IMAGE                                         NODE                 DESIRED STATE  CURRENT STATE              ERROR  PORTS
    rqzcku75xixi  vote_result.1      dockersamples/examplevotingapp_result:before  swarm-worker000001   Running        Running about an hour ago         
    fvksuitv82z1  vote_vote.1        dockersamples/examplevotingapp_vote:before    swarm-worker000000   Running        Running about an hour ago         
    zl0ali52t0ea  vote_db.1          postgres:9.4                                  swarm-manager000000  Running        Running about an hour ago         
    tu2g3onejhwb  vote_redis.1       redis:alpine                                  swarm-worker000001   Running        Running about an hour ago         
    hib7019yak6j  vote_visualizer.1  dockersamples/visualizer:stable               swarm-manager000000  Running        Running about an hour ago         
    qq1v4x0ll54q  vote_worker.1      dockersamples/examplevotingapp_worker:latest  swarm-manager000000  Running        Running about an hour ago         
    do6megki7u5u  vote_vote.2        dockersamples/examplevotingapp_vote:before    swarm-worker000001   Running        Running about an hour ago         
    ap1bpiwdk1lc  vote_redis.2       redis:alpine                                  swarm-worker000000   Running        Running about an hour ago        


**Some Internals:**  

 - Docker uses custom linux distribution which has the latest kernel for master and worker nodes.
 - Docker creates system containers in master and worker nodes that takes care of background tasks like tying in with logging, Load balancer. 
 - Upgrade is done using rolling upgrade of nodes - minimum 3 managers are needed
 - Logs from containers to a native cloud provider abstraction (a storage account in the created resource group)
 - cloudstor is used as volume plugin for persistent storage - available only in edge channel. I could not initially get this to work. The issue is caused by incorrect alias and I got it fixed by following the steps here(https://forums.docker.com/t/services-stuck-in-pending-when-using-cloudstor-azure-volumes/29938/10) 

**Reference:**  
https://docs.docker.com/docker-for-azure
