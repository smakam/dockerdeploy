**Summary:**  
The steps here deploys a voting application(https://github.com/docker/example-voting-app) in google container engine. Here, I have used Kompose tool(https://github.com/kubernetes-incubator/kompose) to convert docker-compose file to Kubernetes deployment files and then deployed using kubectl. 

**Google container engine:**  
Google container engine uses Kubernetes as orchestration engine with Docker as runtime. Google container engine nicely integrates Kubernetes with Google compute engine components. Compared to base Kubernetes, Google container engine adds value by managing installation of kubernetes and its components, doing cluster management and load balancing.

**Pre-requisites:**  

 - Google cloud account
 - gcloud installation
 - kubectl installation

**Step 1:**  
**Create cluster:**  
Use the following command to create a 2 node container cluster:  

    gcloud container clusters create example-cluster --machine-type g1-small --num-nodes 2

**Note:**  
kubectl uses the credentials  in ~/.kube/config to access the cluster

Following command shows the 2 node cluster created:

    $ kubectl get node
    NAME                                             STATUS    AGE
    gke-example-cluster-default-pool-8a5eb745-75ww   Ready     2h
    gke-example-cluster-default-pool-8a5eb745-gn1g   Ready     2h

To ssh into the node, we can use:

    gcloud compute ssh <nodename>


**Step 2:**  
**Deploy application:**  
Since Kompose tool does not support Compose v3 format and the fact that it does not support named networks and volumes, I have created a custom voting compose file based on examples I found in kompose website. I used the compose file from here(https://github.com/smakam/dockerdeploy/blob/master/cloud-provider-caas/google-container-engine/docker-voting.yml). I have specified labels "kompose.service.type=loadbalancer" on vote and result service since these services needs to be exposed to outside world using load balancer.

**Convert compose to Kubernetes definition format:**  
Use the following command to do the conversion:

    kompose convert -f docker-voting

I have also put the converted Kubernetes definition files here(https://github.com/smakam/dockerdeploy/tree/master/cloud-provider-caas/google-container-engine/votingapp)

**Deploy the votingapp:**  
Use the following command to deploy the votingapp:

    kubectl create -f vote_service

To access the Kubernetes dashboard, we can proxy to local webserver using:
kubectl proxy
Then navigate to this link in your browser: localhost:8001/ui

Following command shows the running services of votingapp:

    $ kubectl get service
    NAME         CLUSTER-IP     EXTERNAL-IP      PORT(S)     AGE
    db           10.3.244.211   <none>           5432/TCP    2m
    kubernetes   10.3.240.1     <none>           443/TCP     2h
    redis        10.3.245.27    <none>           6379/TCP    2m
    result       10.3.240.173   104.197.238.77   5001/TCP    2m
    vote         10.3.252.59    104.197.6.27     5000/TCP    2m
    worker       None           <none>           55555/TCP   2m

As we can see above, "result" and "vote" services have external IP and they are exposed using google compute load balancer. 
If we create the votingapp in kubernetes definition format from scratch, we can use persistent volumes and also deploy multiple replicas for each service. The goal here was to illustrate how we can take docker compose files and deploy using Kubernetes and not go into details of Kubernetes definition formats.

**Delete service and cluster:**  
Following command deletes votingapp service:

    kubectl delete -f votingapp

Following command deletes the 2 node container engine cluster:

    gcloud container clusters delete example-cluster

**Reference:**  
https://cloud.google.com/container-engine/
