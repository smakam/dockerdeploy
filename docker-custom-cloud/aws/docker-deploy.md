
**Summary:**  
The steps here can be used to create Swarm mode cluster in AWS cloud using "Docker for AWS" and it also deploys example-voting-app(https://github.com/docker/example-voting-app) to AWS cloud. Docker CE version runs in the AWS cloud using these steps. 
"Docker for AWS" is Docker's attempt to make Docker containers AWS cloud friendly. AWS cloudformation template is used to create the necessary resources. 

Following are some integrations that Docker has added as part of this offering:

 - Master and worker nodes along with Swarm mode cluster gets automatically created.
 - Security groups are automatically created to expose the necessary ports needed for Swarm mode and ssh. Only master node is exposed to outside world. 
 - ELB(Load balancer) gets automatically created when new Docker services are created. 
 - Logs are automatically directed to AWS cloudwatch log service.
 - AWS infrastructure internals like VPC, Subnet, IAM profiles gets automatically created. 

**Video:**  
[![Docker custom cloud](https://github.com/smakam/dockerdeploy/blob/master/images/dockercustomcloud.jpg)](https://www.youtube.com/watch?v=l2Yy8poceuY")

**Pre-requisities:**
 - AWS cloud account 
 - AWS ssh key needs to be created

**Step 1:**  
**Setup AWS Infra**  
If you are logging in as IAM user, appropriate permissions have to be defined so that Cloudformation can create the necessary resources. This is achieved by creating IAM role as defined here(https://docs.docker.com/docker-for-aws/iam-permissions/). Since I have not logged in as IAM user, I did not have to do this. 
The default Cloudformation template creates new AWS VPC. If you want to use existing VPC, the VPC should be setup like this(https://docs.docker.com/docker-for-aws/faqs/#can-i-use-my-existing-vpc).

**Step 2:**  
**Create cluster**  
There is a link here(https://docs.docker.com/docker-for-aws/#docker-community-edition-ce-for-aws) where we can select the Docker version to be installed. This can either be "stable" or "edge" versions. The edge version has features that has not been fully tested. For this example, I have selected the Docker CE edge version.The links corresponding to the version will start AWS with appropriate cloudformation template. 
Following are some major inputs to be specified as part of Cloudformation template:

 - Number of master nodes. (I used 1)
 - Instance type of master nodes.(I used t2.micro)
 - ssh key to ssh to master node.(I used existing key)
 - Number of worker nodes (I used 2)
 - Instance type of worker nodes. (I used t2.micro)

Following output shows the 3 node cluster created for me. This can be viewed from master node. 

    $ docker node ls
    ID                           HOSTNAME                       STATUS  AVAILABILITY  MANAGER STATUS
    1qe6ky14clqn4fea6hqpuiqg6    ip-172-31-0-98.ec2.internal    Ready   Active        
    ha1tt39qvba6v5x17m6gftyvj *  ip-172-31-31-244.ec2.internal  Ready   Active        Leader
    u7w5rwr289careporolfy6lkp    ip-172-31-24-119.ec2.internal  Ready   Active   

From outside world, ssh is allowed only to the master node. To ssh to master node, use "docker" as username along with the ssh key used when setting up the cluster. From master node, we can ssh to worker nodes using the private network. 
   
**Step 3:**  
**Deploy application:**  
Following command deploys the application:  

    docker stack deploy --compose-file docker-stack.yml vote

Since this application has 3 services that exposes ports 8080, 5000 and 5001, an ELB gets automatically created in AWS which is mapped to the Docker services. 


**Some Internals:**  

 - Docker uses custom linux distribution which has the latest kernel for master and worker nodes.
 - Docker creates system containers in master and worker nodes that takes care of background tasks like tying in with AWS logs, ELB etc.
 - Upgrade is done using rolling upgrade of nodes - minimum 3 managers are needed
 - cloudstor is used as volume plugin for persistent storage - available only in edge channel. I could not initially get this to work. The issue is caused by incorrect alias and I got it fixed by following the steps here(https://forums.docker.com/t/services-stuck-in-pending-when-using-cloudstor-azure-volumes/29938/10). The correction steps are same for AWS and Azure. 
  

**Reference:**  
https://docs.docker.com/docker-for-aws/
