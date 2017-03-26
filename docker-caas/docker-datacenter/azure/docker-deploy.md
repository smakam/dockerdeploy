**Summary:**  
The steps here deploys a voting application(https://github.com/docker/example-voting-app) in Docker datacenter that runs on Azure cloud. Docker Datacenter is Docker’s enterprise grade CaaS(Container as a service) solution where they have integrated their open source software with some proprietary software and support to make it into a commercial product. Docker datacenter can run in both public and private cloud. Docker datacenter runs Docker EE software. Docker datacenter is a paid software and can be tried out for free for 30 days.

**Docker datacenter:**  
Docker datacenter has the following components:  
**UCP:**  
UCP provides controller to manage the platform and it integrates well with Light weight directory access protocol(LDAP) and Role based access control(RBAC). This allows enterprises to integrate Docker Datacenter with their current user management solutions. Swarm mode is used for orchestration. UCP provides a nice GUI and same Docker APIs can be used to control UCP. Multiple UCP controllers can operate in HA mode. UCP also provides monitoring and logging support for Containers.  
**DTR:**  
DTR provides a secure Docker image repository and it integrates well with UCP.

**Pre-requisites:**  

 - Azure cloud account
 - Docker store account(same as Docker hub)
 - Docker datacenter license key

**Step 1:**  
**Create Service principal:**  
"Service principal" is a concept used in Azure to that allows invoking Azure API with needed permissions. It is like IAM concept in AWS. The first step is to create Service principal and then use the credentials given by the Service principal to invoke Azure template. I used the following command to create the Service principal:

    docker run -ti docker4x/create-sp-azure myddcee

**Step 2:**  
**Install Docker datacenter:**  
The installation can be done from here(https://store.docker.com/editions/enterprise/docker-ee-azure?tab=description).
Following are the inputs needed from user:

 - Appid and Appsecret (Obtained from previous step)
 - Master - minimum count - 1
 - Worker - Need to specify count and type
 - ssh key
 - username and password
 - license

once deployment is successful, we can see ucploginurl and dtrloginurl in deployment section.

**Step 3:**  
**Deploy application:**  
Using UCP URL and username/password specified during installation, we can login to UCP GUI. 
To deploy the votingapp, we can select stacks and applications->Deploy option. We can input the compose stack file and start the deployment. The services that are exposed will provide the URL under the publish section using which we can access the service. 

**Reference:**  
https://blog.docker.com/2016/06/docker-datacenter-aws-azure-cloud/
