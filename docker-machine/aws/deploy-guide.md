**Summary:**  
This guide describes steps to deploy example-voting-app(https://github.com/docker/example-voting-app) to AWS cloud using docker-machine. We will use Docker Swarm mode for the application deployment.

**Pre-requisities:**  
 - AWS cloud account 
 - Install docker-machine 0.10.0 
 - Store AWS credentials in .aws/credentials(access key and secret key)

**Step 1:**  
**Create Master and Worker EC2 nodes:**  

    docker-machine create --driver amazonec2 --amazonec2-ssh-keypath <keyfile> master
    docker-machine create --driver amazonec2 --amazonec2-ssh-keypath <keyfile> worker

**Step 2:**  
Add your user to the docker group. This is needed to execute Docker commands without using sudo. By default, "ubuntu" is chosen as username. By default, "ubuntu" is chosen as username. This step is needed in both master and worker node.

    sudo gpasswd -a ubuntu docker

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
    docker stack deploy --compose-file docker-stack.yml vote


**Step 5:**  
**Expose ports:**  
By default, docker-machine exposes port 22 and 2376 and puts the EC2 nodes in new security group docker-machine. Port 2376 is used for older Swarm. For new Swarm mode, we need to expose tcp ports 2377, 7946 and udp ports 4789, 7946. For this application, we need to open up ports 8080, 5000, 5001. This can be achieved by modifying the appropriate security group.

Since Docker Swarm uses a routing mesh, the services can be accessed using any of the nodes using the public IP address and port numbers.

**Outputs:**  
Following command shows the 2 node Swarm cluster:

    $ docker node ls
    ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
    c831p10u709xgwjn7hdm2fxgv    worker    Ready   Active        
    hya2kk7w1id12cnfs4wzdn06a *  master    Ready   Active        Leader

The stack and service status can be checked using following commands:

    $ docker service ls
    ID            NAME             MODE        REPLICAS  IMAGE
    764j0d2099rb  vote_visualizer  replicated  1/1       dockersamples/visualizer:stable
    8r8kam0xi5y6  vote_worker      replicated  1/1       dockersamples/examplevotingapp_worker:latest
    tj2oggnnspwb  vote_db          replicated  1/1       postgres:9.4
    tp9ghzq3zs05  vote_redis       replicated  2/2       redis:alpine
    uebeyyf3if0x  vote_vote        replicated  2/2       dockersamples/examplevotingapp_vote:before
    xzcimrcoktk0  vote_result      replicated  1/1       dockersamples/examplevotingapp_result:before

    $ docker stack ps vote
    ID            NAME               IMAGE                                         NODE    DESIRED STATE  CURRENT STATE           ERROR  PORTS
    x1madunj2sks  vote_vote.1        dockersamples/examplevotingapp_vote:before    master  Running        Running 11 minutes ago         
    y0wlnpm1e2wq  vote_db.1          postgres:9.4                                  master  Running        Running 10 minutes ago         
    yzeus02p689g  vote_redis.1       redis:alpine                                  master  Running        Running 11 minutes ago         
    qqyn26jymbse  vote_visualizer.1  dockersamples/visualizer:stable               master  Running        Running 10 minutes ago         
    mdmqsi6ha5qw  vote_worker.1      dockersamples/examplevotingapp_worker:latest  master  Running        Running 10 minutes ago         
    v5ntjw928122  vote_result.1      dockersamples/examplevotingapp_result:before  master  Running        Running 10 minutes ago         
    hnj31jwvv1qj  vote_vote.2        dockersamples/examplevotingapp_vote:before    worker  Running        Running 12 minutes ago         
    rips4tte3qn7  vote_redis.2       redis:alpine                                  worker  Running        Running 12 minutes ago   


**Reference:**  
https://docs.docker.com/machine/drivers/aws/
