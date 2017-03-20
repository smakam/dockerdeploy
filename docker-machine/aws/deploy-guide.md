**Summary:**
This steps deploy example-voting-app(https://github.com/docker/example-voting-app) to AWS cloud using docker-machine

**Pre-requisities:**

 - AWS cloud account 
 - Install docker-machine 0.10.0 
 - Store AWS credentials in .aws/credentials(access key and secret key)

**Step 1:**
**Create Master and Worker EC2 nodes:**

    docker-machine create --driver amazonec2 --amazonec2-ssh-keypath ~/aws1/smakam-east master
    docker-machine create --driver amazonec2 --amazonec2-ssh-keypath ~/aws1/smakam-east worker

**Step 2:**
Add your user to the docker group. This is needed to execute Docker commands without using sudo. By default, "ubuntu" is chosen as username. This step is needed in both master and worker node.
sudo gpasswd -a ubuntu docker
Log out and back in.

**Step 3:**
Create Swarm cluster:
In master:

    docker swarm init

In worker:

    docker swarm join ..

(The command to be executed in worker can be copied from master which is displayed as part of "docker swarm init" output)

**Step 4:**
Deploy application:

    eval $(docker-machine env master)
    docker stack deploy --compose-file docker-stack.yml vote


**Step 5:**
**Expose ports:**
By default, docker-machine exposes port 22 and port 2376. Port 2376 is necessary for Swarm mode communication. For this application, we need to open up ports 8080, 5000, 5001.