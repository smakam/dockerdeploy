**Summary:**  
The steps here can be used to create Swarm mode cluster in Google cloud using "Docker for GCE" and it also deploys example-voting-app(https://github.com/docker/example-voting-app) to Google cloud. "Docker for GCE" is currently in beta and we need to request for access at https://beta.docker.com/. Docker CE version runs in the Google cloud using the  steps below.  "Docker for GCE" is Docker's attempt to make Docker containers Google cloud friendly. Google deployment template is used to create the necessary resources. 

Following are some integrations that Docker has added as part of this offering:

 - Master and worker nodes along with Swarm mode cluster gets automatically created.
 - Firewalls are automatically created to expose the necessary ports needed for Swarm mode and ssh. Only master node is exposed to outside world. 
 - Load balancer gets automatically created when new Docker services are created and exposes the services. 
 - Google infrastructure internals like virtual network, load balancer gets automatically created. 

**Pre-requisities:**
 - Google cloud account 
 - Sign up for beta. The beta will be tied to your Google account after approval
 - gcloud sdk installation. There is no GUI based deployment available yet in the beta version.

**Step 1:**  
**Setup Google cloud Infra**  
Enable Google Cloud Deployment Manager api and Google cloud RuntimeConfig API from Google cloud console. 

**Step 2:**  
**Create cluster**  
"Docker for GCE" deployment can be done either using gcloud CLI or gcloud shell. GUI option is not yet available. In both cases, Google deployment manager is used to involve the template to create cluster. 
Following are some major inputs to be specified as part of Google deployment template:

 - Number of master nodes. (I used 1)
 - Instance type of master nodes.(default uses g1.small)
 - Number of worker nodes (I used 2)
 - Instance type of worker nodes. (default uses g1.small)
 - Location of the template

Following is the command that I used to create the cluster:

    gcloud deployment-manager deployments create docker \
    >     --config https://docker-for-gcp-templates.storage.googleapis.com/v8/Docker.jinja \
    >     --properties managerCount:1,workerCount:2

Following output shows the 3 node cluster created for me:

        $ docker node ls
    ID                           HOSTNAME              STATUS  AVAILABILITY  MANAGER STATUS
    dqug3urahgvkdv76s3f3ss0kw    docker-worker-z4q7rk  Ready   Active        
    elbgzbsyrppu2obbzlle9vsk3 *  docker-manager-1      Ready   Active        Leader
    pm082i4iagxf0arg40zmg87w8    docker-worker-x9vbd0  Ready   Active

From outside world, ssh is allowed only to the master node. From master node, we can ssh to worker nodes using the private network. 
   
**Step 3:**  
**Deploy application:**  
I used the following command to scp the compose file from my machine to Master node:

    gcloud compute copy-files docker-stack.yml docker@docker-manager-1:

Following command deploys the application:

    docker stack deploy --compose-file docker-stack.yml vote

Since this application has 3 services that exposes ports 8080, 5000 and 5001, Load balancer gets automatically created in Google cloud which is mapped to the Docker services. The 3 services can be accessed using load balancer domainname/IP address.
Following output shows the running services:

        $ docker stack ps vote
        ID            NAME               IMAGE                                         NODE                  DESIRED STATE  CURRENT STATE              ERROR                      PORTS
    bdhyqussif4d  vote_worker.1      dockersamples/examplevotingapp_worker:latest  docker-manager-1      Running        Running about an hour ago                             
    x7xglt1g1zfh  vote_db.1          postgres:9.4                                  docker-manager-1      Running        Running about an hour ago                             
    ixfjmb2isz1w  vote_redis.1       redis:alpine                                  docker-worker-x9vbd0  Running        Running about an hour ago                             
    vfkvhbauvogn  vote_visualizer.1  dockersamples/visualizer:stable               docker-manager-1      Running        Running about an hour ago                             
    ozm3v8ijjilq  vote_worker.1      dockersamples/examplevotingapp_worker:latest  docker-manager-1      Shutdown       Failed about an hour ago   "task: non-zero exit (1)"  
    0szne5rw8u5u  vote_result.1      dockersamples/examplevotingapp_result:before  docker-worker-x9vbd0  Running        Running about an hour ago                             
    ne8yed9r1k9j  vote_vote.1        dockersamples/examplevotingapp_vote:before    docker-worker-z4q7rk  Running        Running about an hour ago                             
    tkeh3x2ahbpt  vote_redis.2       redis:alpine                                  docker-worker-z4q7rk  Running        Running about an hour ago                             
    nu5yjudwywb6  vote_vote.2        dockersamples/examplevotingapp_vote:before    docker-manager-1      Running        Running about an hour ago                             
           
**Deleting the cluster:**  
This is done in 2 steps:

    gcloud compute instances delete --delete-disks=boot $(gcloud compute instances list --filter='networkInterfaces[0].network ~ docker-network' --uri)
    gcloud deployment-manager deployments delete docker

**Some Internals:**  

 - Docker uses custom linux distribution which has the latest kernel for master and worker nodes.
 - Docker creates system containers in master and worker nodes that takes care of background tasks like tying in with network, Load balancer. Some of the work being done in Infrakit is being used to integrate with GCE infrastructure. 
 - Upgrade is done using rolling upgrade of nodes - minimum 3 managers are needed
 - Logs are not yet integrated with Google cloud logs. This is planned for future. For now, the logs can be accessed directly from nodes using docker log commands.
 - Worker count can be changed dynamically. Master count cannot be changed live.

**Reference:**  
https://beta.docker.com/docs/gcp/

