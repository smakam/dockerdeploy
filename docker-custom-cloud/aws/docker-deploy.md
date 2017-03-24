
**Summary:**  
The steps here can be used to create Swarm mode cluster in AWS cloud using "Docker for AWS" and it also deploys example-voting-app(https://github.com/docker/example-voting-app) to AWS cloud. Docker CE version runs in the AWS cloud using these steps. 
"Docker for AWS" is Docker's attempt to make Docker containers AWS cloud friendly. AWS cloudformation template is used to create the necessary resources. 

Following are some integrations that Docker has added as part of this offering:

 - Master and worker nodes are created and Swarm mode cluster gets automatically created.
 - Security groups are automatically created to expose the necessary ports needed for Swarm mode and ssh. Only master node is exposed to outside world. 
 - Load balancer gets automatically created when new Docker services are created. 
 - Logs are automatically directed to AWS cloudwatch log service.
 - AWS infrastructure internals like VPC, Subnet, IAM profiles gets automatically created. 

**Pre-requisities:**
 - AWS cloud account 
 - Install docker-machine 0.10.0 
 - Google project ID

**Step 1:**  
**Create IAM role if needed**

**Step 2:**  
**Create cluster**  
We need to select the Docker CE version here. This can either be "stable" or "edge" versions. The links corresponding to the version will start AWS with appropriate cloudformation template. 
Following are some major inputs to be specified as part of Cloudformation template:

 - Number of master nodes. (I used 1)
 - Instance type of master nodes.(I used t2.micro)
 - ssh key to ssh to master node.(I used existing key)
 - Number of worker nodes (I used 2)
 - Instance type of worker nodes. (I used t2.micro)

Following output shows the 3 node cluster created for me:

    $ docker node ls
    ID                           HOSTNAME                       STATUS  AVAILABILITY  MANAGER STATUS
    1qe6ky14clqn4fea6hqpuiqg6    ip-172-31-0-98.ec2.internal    Ready   Active        
    ha1tt39qvba6v5x17m6gftyvj *  ip-172-31-31-244.ec2.internal  Ready   Active        Leader
    u7w5rwr289careporolfy6lkp    ip-172-31-24-119.ec2.internal  Ready   Active   
   
**Step 3:**  
**Deploy application:**  

    eval $(docker-machine env master)
    docker stack deploy --compose-file docker-stack.yml vote

Since this application has 3 services that exposes ports 8080, 5000 and 5001, an ELB gets automatically created in AWS which is mapped to the Docker services. 


**Some Internals:**  

  

**Reference:**  
https://docs.docker.com/docker-for-aws/
