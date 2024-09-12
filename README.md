# Create-and-Manage-Services-on-Docker-Swarm
Managing services with Docker Swarm 

# Docker Swarm Project Documentation

## Overview
This project demonstrates how to create and manage a Docker Swarm cluster using three nodes: one manager node and two worker nodes. The tutorial will utilize the `tutum/hello-world` image as an example application. By following this guide, you will learn how to initialize a Docker Swarm, manage services, and apply best practices for a secure and efficient deployment.

## Table of Contents
- [Create-and-Manage-Services-on-Docker-Swarm](#create-and-manage-services-on-docker-swarm)
- [Docker Swarm Project Documentation](#docker-swarm-project-documentation)
  - [Overview](#overview)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
    - [Open Protocols and Ports](#open-protocols-and-ports)
  - [Setting Up the Swarm](#setting-up-the-swarm)
    - [Initialising the Cluster](#initialising-the-cluster)
    - [Adding Nodes](#adding-nodes)
  - [Deploying Application Services](#deploying-application-services)
  - [Managing the Swarm](#managing-the-swarm)
    - [Publish the Application Global](#publish-the-application-global)
    - [Inspecting Services](#inspecting-services)
    - [Scaling Services](#scaling-services)
    - [Applying Rolling Updates](#applying-rolling-updates)
    - [Draining Nodes](#draining-nodes)
    - [Deleting Services](#deleting-services)
    - [Deploying Microservces to Docker Swarm](#deploying-microservces-to-docker-swarm)
  - [Swarm Mode Routing Mesh](#swarm-mode-routing-mesh)
  - [Best Practices](#best-practices)
  - [Conclusion](#conclusion)

## Prerequisites
This project utilises:
- Three Linux hosts with Docker installed and capable of network communication.
- The IP address of the manager machine (e.g., `192.168.99.100`).
- Open ports between the hosts as specified below.

For this project, I will utilise three virtual machines on [Play with Docker](https://labs.play-with-docker.com/)

### Open Protocols and Ports
If not opened by default, manually allow:
- **Port 2377**: TCP for communication with and between manager nodes.
- **Port 7946**: TCP/UDP for overlay network node discovery.
- **Port 4789**: UDP for overlay network traffic.

For encrypted overlay networks, ensure IP protocol 50 (IPSec ESP) is allowed.

## Setting Up the Swarm
Before the creation of `swarm`, I will check if swarm is active and running.
```
docker info
```
>
> Output: 
>
```
 Swarm: inactive                     <======================== swarm is not active. 
 Runtimes: runc io.containerd.runc.v2 nvidia
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 8fc6bcff51318944179630522a095cc9dbf9f353
 runc version: v1.1.13-0-g58aa920
 init version: de40ad0
 Security Options:
  seccomp
   Profile: unconfined
```
Since swarm is inactive, I will have to create it. 
### Initialising the Cluster
To create a `swarm`, it has to be initialised with the IP address of the swarm manager. It can be retireved by running:
```
ifconfig 
```
1. **On the Manager Node**: I will run:
   ```bash
   docker swarm init --advertise-addr <IP Address>
   ```
> Output:
>
```
Swarm initialized: current node (5q***********************l) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-3***************************a7ut3jn1zgph4w886z47j-*********9wc0k5cbqgmfvjm **********:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
### Adding Nodes
1. **On Worker Nodes**:
   - I will run the join token from the manager node:
     ```bash
     docker swarm join --token SWMTKN-1-3***************************a7ut3jn1zgph4w886z47j-*********9wc0k5cbqgmfvjm **********:2377
     ```
   - This needs to be executed on each worker node to join the swarm. Let's confirm nodes available before joing the workers to the manager node. 
    ```bash
    docker node ls
    ```
    > Output:
    ```
    ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
    5qb5870v5nzgy53xurf62rqbl *   node3      Ready     Active         Leader           24.0.7
    ```
From the above result, we can observe that we have only one node, which assumes the `Leader` as **Manager Status**. 

(a) **Add Worker1**:
    ```bash
    docker swarm join --token SWMTKN-1-3***************************a7ut3jn1zgph4w886z47j-*********9wc0k5cbqgmfvjm **********:2377
    ```
    
    > Output:
    ```
    This node joined a swarm as a worker.
    ```

(b) **Add Worker2**:
    ```bash
   docker swarm join --token SWMTKN-1-3***************************a7ut3jn1zgph4w886z47j-*********9wc0k5cbqgmfvjm **********:2377
    ```

   > Output:
    ```
    This node joined a swarm as a worker.
    ```

Having initialised the swarm service and joined the worker nodes to form the swarm clusters, I will go back to the manager node and check the number of nodes in the cluster again. 
```bash
docker node ls 
```

> Output
>
```
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
q7e7kxh5vrhg4bv36olg3k0we     node1      Ready     Active                          24.0.7
336tiyodkhezyxfc1sgf2ltuk     node2      Ready     Active                          24.0.7
5qb5870v5nzgy53xurf62rqbl *   node3      Ready     Active         Leader           24.0.7
```

We can confirm that the two worker nodes (node1 and node2) have joined the manager or leader node (node3) on the cluster. 

## Deploying Application Services
Here, I will use the docker image, `tumtum/hello-world` as an example application. 
1. First, I will pull the image from `Dockerhub`.
```bash
docker pull tutum/hello-world
```

2. **Deploy the Hello World Service**:
   ```bash
   docker service create --name webapp --published published=8080,target=80 tutum/hello-world
   ```
   ![alt text](../images/image-1.png)

2. **List Service**:
```bash
docker service ls
```
> Output: One service was created.
>
```bash
ID             NAME      MODE         REPLICAS   IMAGE                      PORTS
ttyj5j938bxp   webapp    replicated   1/1        tutum/hello-world:latest   *:8080->80/tcp
```
3. Get the worker on which the service is running one.
```
docker service ps webapp
```

> Output:
>
```
ID             NAME       IMAGE                      NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
5dtaqjg2qmwv   webapp.1   tutum/hello-world:latest   node3     Running         Running 8 minutes ago
```

The web app was published on the node3, the `Leader` or `manager` node. This means that the application is published locally. 

4. **Access the Web Application**:<p>
![alt text](../images/image-2.png)

## Managing the Swarm
### Publish the Application Global
The web app as at now has only been published locally on the manager node. None of the worker nodes have been assigned to publish the app. To make the service available for the other service, I will first remove the local service and then make it global. 

```
docker service rm webapp                         # Removes the local service 

docker service create --name webapp --publish published=8080,target=80 --mode global tutum/hello-world     # create a global service
```
> Output:
>
```bash
ys82p98ko50yqwyw824taw8ft
overall progress: 3 out of 3 tasks 
q7e7kxh5vrhg: running   
5qb5870v5nzg: running   
336tiyodkhez: running   
verify: Service converged 
```
The service has been made known to all the nodes, hence, the `3 out of 3 task running`. This means that we would have 3 replicas created. Let's get the service list again. 
```
docker service ls
```

> Output
```bash
ID             NAME      MODE      REPLICAS   IMAGE                      PORTS
ys82p98ko50y   webapp    global    3/3        tutum/hello-world:latest   *:8080->80/tcp
```

Once gain, I will query the nodes on which the services are running. 
```bash
docker service ps webapp
```

> Output:
>
```bash
ID             NAME                               IMAGE                      NODE      DESIRED STATE   CURRENT STATE            ERROR     PORTS
hujsyi299tal   webapp.5qb5870v5nzgy53xurf62rqbl   tutum/hello-world:latest   node3     Running         Running 12 minutes ago             
snccmo00tigb   webapp.336tiyodkhezyxfc1sgf2ltuk   tutum/hello-world:latest   node2     Running         Running 12 minutes ago             
xznoohrc30zt   webapp.q7e7kxh5vrhg4bv36olg3k0we   tutum/hello-world:latest   node1     Running         Running 12 minutes ago   
```

The results indicates that the services were distributed accross the cluster. We can confirm from each worker node.

(a) On the `Node1 (worker1)`
```
docker ps 
```
> Output:
>
```
CONTAINER ID   IMAGE                      COMMAND                  CREATED          STATUS          PORTS     NAMES
7c5389f71993   tutum/hello-world:latest   "/bin/sh -c 'php-fpm…"   17 minutes ago   Up 17 minutes   80/tcp    webapp.q7e7kxh5vrhg4bv36olg3k0we.xznoohrc30zt9pcf9mxs0ao4t
```

(b) On the `Node2 (worker2)`
```
docker ps 
```
> Output:
>
```
CONTAINER ID   IMAGE                      COMMAND                  CREATED          STATUS          PORTS     NAMES
a358e55cdd90   tutum/hello-world:latest   "/bin/sh -c 'php-fpm…"   19 minutes ago   Up 19 minutes   80/tcp    webapp.336tiyodkhezyxfc1sgf2ltuk.snccmo00tigbz4i8pbkm6a5u0
```

> **Note:** Compare names of containers to confirm containers were created on each worker node as scheduled by the manager node (node3). 

### Inspecting Services
1. **Inspect the Service**: On the manager node, let's inspect the services.
   ```bash
   docker service inspect --pretty webapp
   ```

   ![alt text](../images/image-3.png)


### Scaling Services
1. **Scale the Service**: Let's scale the webapp to 15
   ```bash
   # First change to replica mode to be able to scale
   docker service create --replicas 6 --name webapp --publish published=8080,target=80 tutum/hello-world
   ```
   > Output:
   
  ```
  x6gfv36s1xq4k51quw9m4jem0
  overall progress: 6 out of 6 tasks 
  1/6: running   
  2/6: running   
  3/6: running   
  4/6: running   
  5/6: running   
  6/6: running   
  verify: Service converged
  ```

2. Check service distribution 
  
```bash
  docker service ps webapp 
```
> Output:
>
```bash
ID             NAME       IMAGE                      NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
6cq7h6hpig10   webapp.1   tutum/hello-world:latest   node2     Running         Running 5 minutes ago             
jqi7354rwk9m   webapp.2   tutum/hello-world:latest   node1     Running         Running 5 minutes ago             
leqhz0oktzrh   webapp.3   tutum/hello-world:latest   node3     Running         Running 5 minutes ago             
29noj7ifd56c   webapp.4   tutum/hello-world:latest   node2     Running         Running 5 minutes ago             
kss2da2ssqq6   webapp.5   tutum/hello-world:latest   node1     Running         Running 5 minutes ago             
dhj4dc8skwlv   webapp.6   tutum/hello-world:latest   node3     Running         Running 5 minutes ago 
```
Each Worker is running two tasks. A manager can also be referred to as a worker. 

3. Scale the service in the swarm 
```bash
docker service scale webapp=15
```
>
> Output
>
```
webapp scaled to 15
overall progress: 15 out of 15 tasks 
1/15: running   
2/15: running   
3/15: running   
4/15: running   
5/15: running   
6/15: running   
7/15: running   
8/15: running   
9/15: running   
10/15: running   
11/15: running   
12/15: running   
13/15: running   
14/15: running   
15/15: running   
verify: Service converged 
```

4. Check the service task distribution:
```bash
docker service ps webapp
```

> Output
>
```bash
ID             NAME        IMAGE                      NODE      DESIRED STATE   CURRENT STATE            ERROR     PORTS
6cq7h6hpig10   webapp.1    tutum/hello-world:latest   node2     Running         Running 11 minutes ago             
jqi7354rwk9m   webapp.2    tutum/hello-world:latest   node1     Running         Running 11 minutes ago             
leqhz0oktzrh   webapp.3    tutum/hello-world:latest   node3     Running         Running 11 minutes ago             
29noj7ifd56c   webapp.4    tutum/hello-world:latest   node2     Running         Running 11 minutes ago             
kss2da2ssqq6   webapp.5    tutum/hello-world:latest   node1     Running         Running 11 minutes ago             
dhj4dc8skwlv   webapp.6    tutum/hello-world:latest   node3     Running         Running 11 minutes ago             
r3sezutb167t   webapp.7    tutum/hello-world:latest   node2     Running         Running 2 minutes ago              
ubi377t53ij7   webapp.8    tutum/hello-world:latest   node3     Running         Running 2 minutes ago              
3p0zxhlez3yz   webapp.9    tutum/hello-world:latest   node2     Running         Running 2 minutes ago              
ihl2x0kv1aqq   webapp.10   tutum/hello-world:latest   node3     Running         Running 2 minutes ago              
p4j25id6q6by   webapp.11   tutum/hello-world:latest   node1     Running         Running 2 minutes ago              
ok16fjx4n1j7   webapp.12   tutum/hello-world:latest   node2     Running         Running 2 minutes ago              
13xlzoa9s5i3   webapp.13   tutum/hello-world:latest   node1     Running         Running 2 minutes ago              
tj7ctl91gse5   webapp.14   tutum/hello-world:latest   node3     Running         Running 2 minutes ago              
geru9uuxuos5   webapp.15   tutum/hello-world:latest   node1     Running         Running 2 minutes ago
```
Once again, each node or woker (including the manager) is running 5 tasks. 


### Applying Rolling Updates
1. **Update the Service**:
```bash
docker service create \
  --replicas 3 \
  --name redis \
  --update-delay 10s \
  tutum/hello-world:latest
```
The `--update-delay` flag configures the time delay between updates to a service task or sets of tasks.
> Results:
>
```bash
huvy6wgdxf1ogxcglriyjro4a
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Waiting 5 seconds to verify that tasks are stable... 
service update paused: update paused due to failure or early termination of task sdh2nzyunhvoe1dsc0xdyc5rj
```

Checking why the update results: Updated was successful.
```bash
docker service inspect --pretty webapp
```
>
> Results:
>
```yml
ID:             it5livppxqm1w4o6mxxx
Name:           webapp
Service Mode:   Replicated
 Replicas:      1
Placement:
UpdateConfig:
```

### Draining Nodes
At times, like during `scheduled maintenance`, there might be the need to set a node to `Drain` availability. This status prevents the node from accepting new tasks from the swarm manager. Additionally, it instructs the manager to stop any tasks currently running on that node and to launch replica tasks on a node with `Active` availability.
1. **Confirm if all service nodes are active**:
```bash
docker node ls
```

> Results:
>
```bash
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
qbgxn0pzximzyz918ncss5zio     node1      Ready     Active                          24.0.7
u4i10vduj1q2gydpfmiwjq5ek     node2      Ready     Active                          24.0.7
sg8zyjdc4o93h72bawlht88bu *   node3      Ready     Active         Leader           24.0.7
```
2. **Drain a Node**:
   ```bash
   docker node update --availability drain node2
   ```
**Confirm Node Drain**:
```bash

```

>
> Results: node is drained.
```
ID:                     u4i10vduj1q2gydpfmiwjq5ek
Hostname:               node2
Joined at:              2024-09-12 09:31:10.699654124 +0000 utc
Status:
 State:                 Ready
 Availability:          Drain            <<<=====================================
```
**Let's see how the swarm manager updated the task assignments for the redis service:**
```bash
docker service ps webapp 
```

Swarm has redistributed all tasks to node3. <p>
![alt text](../images/image-7.png)

**Reactivate Node2**:

```bash
docker node update --availability active node2
```

**Confirm reactivation**:
```bash
docker node inspect --pretty node2
```

> Output:
>
```yml
ID:                     u4i10vduj1q2gydpfmiwjq5ek
Hostname:               node2
Joined at:              2024-09-12 09:31:10.699654124 +0000 utc
Status:
 State:                 Ready
 Availability:          Active              <<<========================================
```

### Deleting Services 
1. **Remove the Service**: In this session I will remove the hello-world app and deploy a simple microservice, Docker's Example Voting App, on the swarm. 
```bash
docker service rm webapp
```

### Deploying Microservces to Docker Swarm

2. **Deploy the Microservice**: <p>
Clone the app and change directory to the voting app

```bash
git clone https://github.com/docker/example-voting-app && cd example-voting-app
```

3. Deploy the Service Application
The voting app is a group of services to be deployed. This can be referred to as a `stack`. A stack is a collection of services deployed together. The `docker-stack.yml` file in the current directory will be used to deploy the voting application as a stack on the swarm. 

```bash
docker stack deploy --compose-file=docker-stack.yml voting_stack
```
> Result
>
```sh
Creating network voting_stack_frontend
Creating network voting_stack_backend
Creating service voting_stack_worker
Creating service voting_stack_redisCreating service voting_stack_db
Creating service voting_stack_vote
Creating service voting_stack_result
```
4. Check No. Stacks Created
```bash
docker stack ls
```
>
> Result:
>
```
NAME      SERVICES
voteapp   6
```

5. **List Services**:
```bash
docker service ls 
```
> Result:
```bash
ID             NAME                 MODE         REPLICAS   IMAGE                                          PORTS
huvy6wgdxf1o   voteapp_db           replicated   1/1        postgres:9.4                                   
v6xgkm28d57y   voteapp_redis        replicated   1/1        redis:alpine                                   
i3ivb2ge3n7x   voteapp_result       replicated   1/1        dockersamples/examplevotingapp_result:latest   *:5001->80/tcp
414rk8btrphs   voteapp_visualizer   replicated   1/1        dockersamples/visualizer:latest                *:8080->8080/tcp
lq0zl131gumc   voteapp_vote         replicated   2/2        dockersamples/examplevotingapp_vote:latest     *:80->80/tcp
4x4u2rkaxubd   voteapp_worker       replicated   1/1        dockersamples/examplevotingapp_worker:latest  
```

6. **List stack services**:
```bash
docker stack services voteapp
```
> Results:
```bash
ID             NAME                 MODE         REPLICAS   IMAGE                                          PORTS
huvy6wgdxf1o   voteapp_db           replicated   1/1        postgres:9.4                                   
v6xgkm28d57y   voteapp_redis        replicated   1/1        redis:alpine                                   
i3ivb2ge3n7x   voteapp_result       replicated   1/1        dockersamples/examplevotingapp_result:latest   *:5001->80/tcp
414rk8btrphs   voteapp_visualizer   replicated   1/1        dockersamples/visualizer:latest                *:8080->8080/tcp
lq0zl131gumc   voteapp_vote         replicated   2/2        dockersamples/examplevotingapp_vote:latest     *:80->80/tcp
4x4u2rkaxubd   voteapp_worker       replicated   1/1        dockersamples/examplevotingapp_worker:latest 
```

7.  **Access Services on the Browser**:
(a) Access the Frontend via http://localhost:80.<p>
![alt text](../images/image-4.png)

(b) Access the Results (backend) via http://localhost:5001.<p>
![alt text](../images/image-5.png)

(c) Visualise the services running. <p>
![alt text](../images/image-6.png)

## Swarm Mode Routing Mesh
Docker Swarm routing mesh allows you to access services from any node in the swarm. This simplifies service discovery and load balancing.

## Best Practices
- Always use fixed IP addresses for the manager node.
- Regularly monitor and manage the health of the services.
- Implement encrypted overlay networks for enhanced security.
- Use Docker secrets to manage sensitive information.

## Conclusion
This project provides a comprehensive guide to creating and managing a `Docker Swarm cluster`. I have outlined the steps to efficiently deploy and manage applications in a distributed environment whilst adhering to the best practices will ensure a secure and reliable setup.

