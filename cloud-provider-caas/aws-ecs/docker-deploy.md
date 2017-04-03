**Summary:**  
The steps here deploys a voting application(https://github.com/docker/example-voting-app) in AWS ECS. I have used the docker-compose application definition format rather than ECS task and service definition format for this example. 

**ECS:**  
AWS provides EC2 Container service(ECS) for folks who want to deploy managed Docker containers in AWS infrastructure. With ECS, Amazon provides its own scheduler to manage Docker containers. ECS installs agents in each node that allows scheduler to interact with agents. ECS integrates very well with other AWS services including elb/alb load balancer, cloudwatch logging service, cloudformation templates etc. AWS recently introduced Application load balancer(ALB) that does L7 load balancing and this integrates well with ECS. Using ALB, we can load balance services directly across Containers. With ECS, users get charged for the EC2 instances and not for the Containers.
ECS container services can be described as a json file. There are 2 parts: First is the task definition format that describes the containers composing the application and how they are linked. Second is the service definition format that describes how many instances of task needs to run and how they need to be exposed to outside including load balancer. In docker-compose version 3, both the task and service definition are combined into 1 file. ECS support docker-compose version 2 with many limitations. 

**Step 1:**  
**Create IAM role:**  
Following link(http://docs.aws.amazon.com/AmazonECS/latest/developerguide/instance_IAM_role.html) specifies how to create role that allows ECS container agent to call ECS api.  Using this, I created role "AmazonECSContainerInstanceRole". We need to use this role when creating the cluster.

**Step 2:**  
**Creating cluster:**  
The cluster can be created either using GUI or using CLI. Following are the important parameters needed:

 - Number of agent nodes.(I have used 2 for this example)
 - ssh key 
 - Security group
 - Instance type ( I used t2.micro)
 - ECS instance IAM role (I used AmazonECSContainerInstanceRole created in previous step)

The cluster can also be created using ECS cli. There are 2 options for CLI. First is to use "aws ecs" and the other is "ecs-cli".

Following command shows the 2 nodes that are part of "ecscluster":

    $ aws ecs list-container-instances --cluster ecscluster
    {
        "containerInstanceArns": [
            "arn:aws:ecs:us-east-1:173760706945:container-instance/04caf28a-4a1e-426e-a82c-26c6da6dd324", 
            "arn:aws:ecs:us-east-1:173760706945:container-instance/28a33b33-1883-4be0-8981-56664166c17a"
        ]
    }

**Step 3:**  
**Deploying application:**  

ecs-cli supports docker-compose formats with some limitations(http://stackoverflow.com/questions/37466422/ec2-container-service-networking). Compose v2 format with named networks and volumes is not supported. Also, compose v3 format is not possible because v3 format is specific to swarm mode. ECS supports only Docker host and bridge mode and not overlay. To have 2 containers to talk to each other, the only way is using links. That means related containers have to stay in same host. 

We need to set the ecs-cli context to the cluster that we created above. This can be done using the following command:

    ecs-cli configure --cluster <clustername>

Following command deploys the tasks of votingapp using docker-compose:

    ecs-cli compose -f docker-voting-aws up

I have modified the votingapp compose file to use only links and also removed named networks and volumes. Another change that I made is to use Java version of worker instead of dotnet because of the issue mentioned here. (https://github.com/dotnet/corefx/issues/8086). The compose file can be found here(https://github.com/smakam/dockerdeploy/blob/master/aws-ecs/docker-voting-aws.yml)

Following output shows the running containers in 1 of the agent nodes:

    $ docker ps
    CONTAINER ID        IMAGE                                         COMMAND                  CREATED             STATUS              PORTS                     NAMES
    c61bb4383e36        tmadams333/example-voting-app-result:latest   "node server.js"         6 minutes ago       Up 6 minutes        0.0.0.0:5001->80/tcp      ecs-ecscompose-vote_service-18-result-b2dfc6b4ffc3be879401
    b21295fe5684        docker/example-voting-app-vote:latest         "gunicorn app:app -b "   6 minutes ago       Up 6 minutes        0.0.0.0:5000->80/tcp      ecs-ecscompose-vote_service-18-vote-8acdc985fa94da8cdf01
    a94c4ccf938f        smakam/workerj:latest                         "java -jar target/wor"   6 minutes ago       Up 6 minutes                                  ecs-ecscompose-vote_service-18-worker-8e8bb5a5a3f0cea0ff01
    76f457389ab1        postgres:9.4                                  "docker-entrypoint.sh"   7 minutes ago       Up 7 minutes        0.0.0.0:32768->5432/tcp   ecs-ecscompose-vote_service-18-db-eaf1c1f0c7def0948501
    dece9f34c824        redis:alpine                                  "docker-entrypoint.sh"   7 minutes ago       Up 7 minutes        0.0.0.0:32769->6379/tcp   ecs-ecscompose-vote_service-18-redis-c68497f4b69cd9be0b00
    0e35f590bfbd        amazon/amazon-ecs-agent:latest                "/agent"                 29 minutes ago      Up 29 minutes                                 ecs-agent

At this point, the "vote" and "result" container can be exposed from the agent node by exposing ports 5000 and 5001 using security group. 

We can create 	ECS service on top of the tasks and expose them using AWS load balancer ELB or ALB. To achieve this, we need to first create the load balancer in AWS and then create service using ECS GUI. When service is created, we can specify the load balancer name. 

Overall, I found task and service deployment model very restrictive in ECS. Following are some limitations I see:

 - Task file combines related containers together. The service file describes the scaling of tasks. This causing scaling tasks which is not really needed. 
 - All containers in a single task always gets scheduled on same node. This does not give very efficient scheduling.
 - Docker compose files work in a very limited fashion with ECS. There are many options in compose that are not supported by ECS.

To destroy the application created using compose, we can use the following ecs command:

    ecs-cli compose -f docker-voting-aws down

**Reference:**  
http://www.allthingsdistributed.com/2015/07/under-the-hood-of-the-amazon-ec2-container-service.html
http://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html
