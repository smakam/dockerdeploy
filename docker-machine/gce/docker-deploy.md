**Summary:**  
This steps deploy example-voting-app(https://github.com/docker/example-voting-app) to Google cloud using docker-machine

**Pre-requisities:**
 - Google cloud account 
 - Install docker-machine 0.10.0 
 - Google project ID

**Step 1:**  
**Create Master and Worker GCE nodes:**  

    docker-machine create --driver google --google-project <project id> master
    docker-machine create --driver google --google-project <project id> worker

By default, Google cloud uses n1 standard instance type. 

**Step 2:**  
Add your user to the docker group. This is needed to execute Docker commands without using sudo. By default, "docker-user" is chosen as username. This step is needed in both master and worker node.

    sudo gpasswd -a docker-user docker

Log out and back in.

**Step 3:**  
**Create Swarm cluster:**  
**In master:**  

    docker swarm init

**In worker:**  

    docker swarm join ..

(The command to be executed in worker can be copied from master which is displayed as part of "docker swarm init" output)

**Step 4:**  
**Deploy application:**  

    eval $(docker-machine env master)
    git clone https://github.com/docker/example-voting-app.git
    cd example-voting-app
    docker stack deploy --compose-file docker-stack.yml vote


**Step 5:**  
**Expose ports:**  
It is not needed to expose any ports for Swarm mode node communication as both the nodes are placed in same virtual network and the port based communication is not blocked. For this application, we need to open up ports 8080, 5000, 5001 to access the services from outside world. This can be achieved by modifying the appropriate security group.

Since Docker Swarm uses a routing mesh, the services can be accessed using any of the nodes using the public IP address and port numbers.
  

**Reference:**  
https://docs.docker.com/machine/drivers/gce/
