**Summary:**  
The steps here deploys a voting application(https://github.com/docker/example-voting-app) in Docker cloud. Docker cloud supports both the Swarm modes. In this example, we have used Docker swarm mode which is provided as a beta service within Docker cloud currently.

**Docker cloud:**  
Docker cloud is a hosted service from Docker to manage Containers. Earlier, Docker cloud used its own orchestration engine, networking and stack file format. With Swarm mode, Docker cloud is starting to use the common Docker infrastructure tools. Docker cloud is free to try for 1 private repository and node and is chargeable after that. With Swarm mode, Docker cloud GUI cannot be used to manage applications. I think this would change soon as the feature comes out of beta mode. At this point, Docker cloud is supported only with AWS in Swarm mode. In older mode, other cloud providers like Azure, Google cloud is supported. Even though "docker-cloud" CLI option is available, it does not seem to be integrated yet with Swarm mode. 

**Pre-requisites:**  

 - Docker cloud account. Docker hub account can be used.
 - Select Swarm mode in Docker cloud.
 - AWS cloud account.

**Step 1:**  
**Registering cloud provider:**  
The first step is to create IAM role in AWS to allow Docker cloud to access AWS resources and then register AWS cloud provider in Docker cloud. This can be done using the procedure here(https://docs.docker.com/docker-cloud/cloud-swarm/link-aws-swarm/#attach-a-policy-for-legacy-aws-links). We need to get the Role ARN from AWS and add it in Docker cloud. Docker cloud can be accessed from https://cloud.docker.com. We can go to Cloud settings->Service provider and add the ARN.

**Step 2:**  
**Creating cluster:**  
We can create a cluster from Docker cloud interface. The only option available is to create cluster in AWS cloud using the Swarm mode. We need to mention inputs like manager, worker count, instance type, ssh key etc.  I chose manager count of 1 and worker count of 2. 

To manage this cluster from linux host, we can run the following container in our linux host and specify the cluster name that we created in Docker cloud. We would have to login to Docker cloud as part of command below.

    docker run --rm -ti -v /var/run/docker.sock:/var/run/docker.sock -e DOCKER_HOST dockercloud/client smakam/awscluster

Following output shows the 3 node cluster created in AWS:

    $ docker node ls
    ID                           HOSTNAME                      STATUS  AVAILABILITY  MANAGER STATUS
    47rs2b3vxc41xsme3ow83vt30 *  ip-172-31-39-71.ec2.internal  Ready   Active        Leader
    4mzfdvglxdgim0gqvzih0wpya    ip-172-31-46-9.ec2.internal   Ready   Active        
    t95poyxnrr1n1gshlhaypqdv4    ip-172-31-14-29.ec2.internal  Ready   Active

**Step 2-a:**  
**Registering already existing stack to Docker cloud:**  
We can register Swarm mode cluster created in other clouds or in our own machines into Docker cloud. 
To do this, we can run the registration container in Swarm master of the cluster that we want to bring in under Docker cloud using command below:

    docker run -ti --rm -v /var/run/docker.sock:/var/run/docker.sock dockercloud/registration

This will prompt for Docker cloud credentials. After the cluster is registered in Docker cloud, we can manage all clusters from same location.

I created a Swarm mode cluster in Azure and imported into Docker cloud using this approach.

**Step 3:**  
**Deploying the application:**  
We can deploy the votingstack using below command:

    docker stack deploy --compose-file docker-stack.yml vote

Following output shows the running services in the stack:

    $ docker stack ps vote
    ID            NAME               IMAGE                                         NODE                          DESIRED STATE  CURRENT STATE             ERROR  PORTS
    bou4ugdcjqsk  vote_redis.1       redis:alpine                                  ip-172-31-14-29.ec2.internal  Running        Running 22 seconds ago           
    fc5b7jsyu02h  vote_visualizer.1  dockersamples/visualizer:stable               ip-172-31-39-71.ec2.internal  Running        Running 9 seconds ago            
    g1jpldmkiiqi  vote_worker.1      dockersamples/examplevotingapp_worker:latest  ip-172-31-39-71.ec2.internal  Running        Preparing 28 seconds ago         
    nh3m6hyiw8kl  vote_result.1      dockersamples/examplevotingapp_result:before  ip-172-31-46-9.ec2.internal   Running        Running 18 seconds ago           
    ygb1jtzybm5w  vote_vote.1        dockersamples/examplevotingapp_vote:before    ip-172-31-46-9.ec2.internal   Running        Running 31 seconds ago           
    lwd2rw12534q  vote_db.1          postgres:9.4                                  ip-172-31-39-71.ec2.internal  Running        Running 25 seconds ago           
    re9ehsitmvdr  vote_redis.2       redis:alpine                                  ip-172-31-46-9.ec2.internal   Running        Running 20 seconds ago           
    h0xupbp3jq9z  vote_vote.2        dockersamples/examplevotingapp_vote:before    ip-172-31-14-29.ec2.internal  Running        Running 30 seconds ago   

For services that are exposed to outside world, Docker cloud will automatically create connect the services to AWS ELB. This is achieved by running system containers in the Container nodes which will register to ELB when new services are created. We can access the vote and result service using the ELB domain address. 

I saw a issue that deletion of Swarm mode cluster did not clean up the AWS resource properly and I had to manually do the deletion. When we try to remove the cluster from Docker cloud, it gives a warning that it does not remove the cluster. 1 way to manually clean up is to go to cloudformation section of AWS console and delete the stack manually. 

**Reference:**  
https://docs.docker.com/docker-cloud/
