#AIT - Labo 04 - Virtualization
## ALBASINI Romain, SELIMI Dardan

##Introduction
This lab builds on a previous lab on load balancing. In this lab, we will go through one possible approach to manage a scalable infrastructure where we can add and remove nodes without having to rebuild the HAProxy image.
The initial configuration is the following :  
![Architecture](../assets/img/initial-architecture.png)

Both servers and the proxy are Docker containers. Requests will be simulated alternatively with different browsers accessing the 192.168.42.42 page. When accessing the URL, the proxy will decide which server is in charge for this request. 

Like we just mentioned, the goal is to implement a scalable infrastructure, looking like this :  
![Architecture](../assets/img/final-architecture.png)

In order to do so, we will perform several tasks which details are explained in this report.


##Table of content
0. [Identify issues and install the tools](#task-0)
1. [Add a process supervisor to run several processes](#task-1)
2. [Add a tool to manage membership in the web server cluster](#task-2)
3. [React to membership changes](#task-3)
4. [Use a template engine to easily generate configuration files](#task-4)
5. [Generate a new load balancer configuration when membership changes](#task-5)
6. [Make the load balancer automatically reload the new configuration](#task-6)
7. [Difficulties](#difficulties)
8. [Conclusion](#conclusion)

## <a name="task-0"></a>Task 0: Identify issues and install the tools	 

**Issues :**

1. **[M1] Do you think we can use the current solution for a production environment? What are the main problems when deploying it in a production environment?**  
 No, the current solution is not scalable. If the trafic increases, we can't easily add or delete nodes. To do so, we must edit the configuration manually, and then restart the containers, implying the application to be offline during this time (as Docker is used)  
  
2. **[M2] Describe what you need to do to add new webapp container to the infrastructure. Give the exact steps of what you have to do without modifiying the way the things are done. Hint: You probably have to modify some configuration and script files in a Docker image.**  
 1) We edit `haproxy.cfg` to add the node  
 `server s3 <s3>:3000 check`  <br />
 2) We edit the `run.sh` script.  
 `sed -i 's/<s3>/$S3_PORT_3000_TCP_ADDR/g' /usr/local/etc/haproxy/haproxy.cfg`  
 3) We rebuild `ha`  
 `docker build -t softengheigvd/ha .`  
 4) We run `ha`  
 `docker run -d -p 80:80 -p 1936:1936 -p 9999:9999 --link s1 --link s2 --link s3 --name ha softengheigvd/ha`  
 5) We run `s3`  
 `docker run -d --name s3 softengheigvd/webapp`  
<br />

3. **[M3] Based on your previous answers, you have detected some issues in the current solution. Now propose a better approach at a high level.**  
 As explained, this isn't dynamic in any way. Every modification must be implemented manually. We would like `haproxy.cfg` to by edited automatically when a node is added/deleted. This approach would be much easier and scalable, and wouldn't require human intervention.  
<br />

4. **[M4] You probably noticed that the list of web application nodes is hardcoded in the load balancer configuration. How can we manage the web app nodes in a more dynamic fashion?**  
 We need to find a took to update automatically the list of nodes. We will see during the lab that using the `serf`agent will let us add those nodes dynamically. This way, a manual update impyling hardcoding the config of the load balancer won't be required anymore. Every addition/removal of a node will be updated automaticaly.  
<br />

5. **[M5] In the physical or virtual machines of a typical infrastructure we tend to have not only one main process (like the web server or the load balancer) running, but a few additional processes on the side to perform management tasks. For example to monitor the distributed system as a whole it is common to collect in one centralized place all the logs produced by the different machines. Therefore we need a process running on each machine that will forward the logs to the central place. (We could also imagine a central tool that reaches out to each machine to gather the logs. That's a push vs. pull problem.) It is quite common to see a push mechanism used for this kind of task. Do you think our current solution is able to run additional management processes beside the main web server / load balancer process in a container? If no, what is missing / required to reach the goal? If yes, how to proceed to run for example a log forwarding process?**   
 Docker was designed to support only one process per Docker image. The image can't output its logs elsewhere in the initial configuration. We will see during the lab that using `s6` will let us having more than one process running simultaneously in a docker image.  
<br />   

6. **[M6] In our current solution, although the load balancer configuration is changing dynamically, it doesn't follow dynamically the configuration of our distributed system when web servers are added or removed. If we take a closer look at the run.sh script, we see two calls to sed which will replace two lines in the haproxy.cfg configuration file just before we start haproxy. You clearly see that the configuration file has two lines and the script will replace these two lines. What happens if we add more web server nodes? Do you think it is really dynamic? Can you propose a solution to solve this?**  
 As the `sed` commands are hardcoded, every addition of a node require the `sed` command to be manually implemented. We can't describe the current implementation as dynamic.
<br />

**Deliverables**:

1. **Take a screenshot of the stats page of HAProxy at <http://192.168.42.42:1936>. You should see your backend nodes.**   
 As expected, we can see both of our backend nodes running :  
 ![Architecture](../logs/task0/task0_1.PNG)

2. **Give the URL of your repository URL in the lab report.**  
 https://github.com/dardanSelimi/Teaching-HEIGVD-AIT-2016-Labo-Docker/tree/master/report

## <a name="task-1"></a>Task 1: Add a process supervisor to run several processes

**Deliverables**:

1. **Take a screenshot of the stats page of HAProxy at <http://192.168.42.42:1936>. You should see your backend nodes. It should be really similar to the screenshot of the previous task.**  
 Indeed, the HAProxy's stats page is similar to the one seen in Task 0 :  
 ![Architecture](../logs/task1/1.1.PNG)

2. **Describe your difficulties for this task and your understanding of what is happening during this task. Explain in your own words why are we installing a process supervisor. Do not hesitate to do more research and to find more articles on that topic to illustrate the problem.**  
 As explained in the question M5, we can't have more that one running process per container. Installing a process supervisor will allow us to bypass this limitation. In our case, `ha` and the `serf` agent will run on the same docker image. To do so, we will use `s6`. It is a small init system that will coordinate the boot process and will allow to run multiple processes inside one container.  

   
## <a name="task-2"></a>Task 2: Add a tool to manage membership in the web server cluster

**Deliverables**:

1. **Provide the docker log output for each of the containers: `ha`, `s1` and `s2`.**  
   * ha log : [logs/task2/ha.log](../logs/task2/ha.log)
   * s1 log : [logs/task2/s1.log](../logs/task2/s1.log)
   * s2 log : [logs/task2/s2.log](../logs/task2/s2.log)

2. **Give the answer to the question about the existing problem with the current solution.**  
 As we explored during the manipulations, the problem is that we have to start `ha` first for the nodes to be linked to it. 
Problème est qu'on doit starter ha d'abord pour que les nodes puissent y être reliés. 
Une solution peut être d'utiliser l'option `-retry-join` et de donner l'adresse de ha pour que l'agent puisse joindre ha. Voir le [lien](https://www.consul.io/docs/agent/options.html) pour plus de détails sur ta soeur

3. **Give an explanation on how `Serf` is working. Read the official website to get more details about the `GOSSIP` protocol used in `Serf`. Try to find other solutions that can be used to solve similar situations where we need some auto-discovery mechanism.**  
 To goal of `Serf` is to let each node to know every other node existing at any given time. Each container has a `serf agent` running, and the different agents will later communicate to exchange information.  
 The protocol used is [GOSSIP](https://www.serf.io/docs/internals/gossip.html). It is based on [SWIM](https://www.cs.cornell.edu/~asdas/research/dsn02-swim.pdf) with improvements on propagation speed and convergence rate. The first agent creates a new cluster, and additional nodes will join it. When joining, the new node will do a full state sync with the existing member over TCP, and begins "gossiping". This communication is run over UDP, and its purpose is to tell the other nodes its existence. When a node fails to send its periodic ACK messages, the other nodes will be asked to try to reach the defaillant node. If it is still not responding, it will be marked as suspicious. If the suspicious node couldn't dispute the suspicion during a certain amount of time, it is finally considered dead.
    

## <a name="task-3"></a>Task 3: React to membership changes

**Deliverables**:

1. **Provide the docker log output for each of the containers:  `ha`, `s1` and `s2`.**  
 * We first start `ha` (log : [logs/task3/ha1.log](../logs/task3/ha1.log))
 * Then, we start one of the nodes (s1 in our case) (log : [logs/task3/s11.log](../logs/task3/s11.log))
 * Giving us to following log for `ha`: [logs/task3/ha12.log](../logs/task3/ha12.log)
 
 *Remark* : We only capture the log from `s1` as the instructions were to start only one of the nodes. We can see in `ha`'s log that `s1` started successfully as it received the `EventMemberJoin` event.  

2. **Provide the logs from the `ha` container gathered directly from the `/var/log/serf.log` file present in the container.**  
 * serf log : [logs/task3/serf.log](../logs/task3/serf.log)

## <a name="task-4"></a>Task 4: Use a template engine to easily generate configuration files

**Deliverables**:

1. **You probably noticed when we added `xz-utils`, we have to rebuild the whole image which took some time. What can we do to mitigate that? Tell us about the pros and cons to merge as much as possible of the command.**  

    Editing the Dockerfile implies a complete rebuild of the container to take changes into account. A docker image is composed of read-only layers that represent the filesystem differences. When editing the `Dockerfile`, a new small layer is added for every `RUN` instruction written. Adding a lot of layers will take time and makes the deployment heavier. It is a good practice to try to merge the `RUN` instructions to mitigate that. It is preferable to write  
    `RUN command 1 && command 2 && command 3`  
    than  
	  ```
	  RUN command 1
	  RUN command 2
	  RUN command 3
	```  
	Docker itself recommands to merge the commands as much as possible ([link](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#run))  
	When building a new image, Docker looks for changes by comparing it to its cached version. Splitting `apt-get` and `apt-update` in two `RUN` instructions will result in the second instruction not being executed, as according to Docker, no changes were detected (see link above and [this](http://thenewstack.io/understanding-the-docker-cache-for-faster-builds/) for further details).  

	<br />
	The problem with this system is the disk usage due to the cache storage. Big docker images will then need a consequent amount of cache storage. 
	


    **There are also some articles about techniques to reduce the image size. Try to find them. They are talking about `squashing` or `flattening` images.**

	 Here are some interesting links : 
	* [squashing](http://jasonwilder.com/blog/2014/08/19/squashing-docker-images/)  
	* [flattening](http://tuhrig.de/flatten-a-docker-container-or-image/)   
<br />

2. **Propose a different approach to architecture our images to be able to reuse as much as possible what we have done. Your proposition should also try to avoid as much as possible repetitions between your images.**  

	As explained above, our `Dockerfile`  `RUN` instructions should be as merged as possible to mitigate the number of layers used.  

3. **Provide the `/tmp/haproxy.cfg` file generated in the `ha` container after each step.  Place the output into the `logs` folder like you already did for the Docker logs in the previous tasks. Three files are expected. In addition, provide a log file containing the output of the `docker ps` console and another file (per container) with `docker inspect <container>`. Four files are expected.**  
   * haproxy.cfg after starting ha : [logs/task4/haproxy_1.cfg](../logs/task4/haproxy_1.cfg)
   * haproxy.cfg after starting s1 : [logs/task4/haproxy_2.cfg](../logs/task4/haproxy_2.cfg)
   * haproxy.cfg after starting s2 : [logs/task4/haproxy_3.cfg](../logs/task4/haproxy_3.cfg)
   * Output of `docker ps` : [logs/task4/docker_ps.log](../logs/task4/docker_ps.log)
   * Output of `docker inspect ha` : [logs/task4/inspect_ha.log](../logs/task4/inspect_ha.log)
   * Output of `docker inspect s1` : [logs/task4/inspect_s1.log](../logs/task4/inspect_s1.log)
   * Output of `docker inspect ha` : [logs/task4/inspect_s2.log](../logs/task4/inspect_s2.log)
    
4. **Based on the three output files you have collected, what can you say about the way we generate it? What is the problem if any?**  

    The outputs have been generated manually. We could have a system to automatize that.  
   
## <a name="task-5"></a>Task 5: Generate a new load balancer configuration when membership changes

**Deliverables**:

1. **Provide the file `/usr/local/etc/haproxy/haproxy.cfg` generated in the `ha` container after each step. Three files are expected. In addition, provide a log file containing the output of the `docker ps` console and another file (per container) with `docker inspect <container>`. Four files are expected.**  
   * haproxy.cfg after starting ha : [logs/task5/haproxy_1.cfg](../logs/task5/haproxy_1.cfg)
   * haproxy.cfg after starting s1 : [logs/task5/haproxy_2.cfg](../logs/task5/haproxy_2.cfg)
   * haproxy.cfg after starting s2 : [logs/task5/haproxy_3.cfg](../logs/task5/haproxy_3.cfg)
   * Output of `docker ps`: [logs/task5/docker_ps.log](../logs/task5/docker_ps.log)
   * Output of `docker inspect ha`: [logs/task5/inspect_ha.log](../logs/task5/inspect_ha.log)
   * Output of `docker inspect s1`: [logs/task5/inspect_s1.log](../logs/task5/inspect_s1.log)
   * Output of `docker inspect ha`: [logs/task5/inspect_s2.log](../logs/task5/inspect_s2.log)
  
2. **Provide the list of files from the `/nodes` folder inside the `ha` container. One file expected with the command output.**  
   * List of files from `/nodes`: [logs/task5/ls_nodes.log](../logs/task5/ls_nodes.log)

3. **Provide the configuration file after you stopped one container and the list of nodes present in the `/nodes` folder. One file expected with the command output. Two files are expected. In addition, provide a log file containing the output of the `docker ps` console. One file expected.**  
   * haproxy.cfg after stopping s1 : [logs/task5/haproxy_4.cfg](../logs/task5/haproxy_2.cfg)
   * List of files from `/nodes`: [logs/task5/ls_nodes2.log](../logs/task5/ls_nodes2.log)
   * Output of `docker ps`: [logs/task5/docker_ps2.log](../logs/task5/docker_ps2.log)
   
## <a name="task-6"></a>Task 6: Make the load balancer automatically reload the new configuration

**Deliverables**:

1. **Take a screenshots of the HAProxy stat page showing more than 2 web applications running. Additional screenshots are welcome to see a sequence of experimentations like shutting down a node and starting more nodes. Also provide the output of `docker ps` in a log file. At least one file is expected. You can provide one output per step of your experimentation according to your screenshots.**  
 * We first ran `ha` and `s1`. As expected, HAProxy's stat page shows only one application running : 
 ![Architecture](../logs/task6/6.1_port.PNG)
 `docker ps` output : [logs/task6/6.1_ps.log](../logs/task6/6.1_ps.log)

 * We then ran `s2`. As expected, HAProxy's stat page now shows both applications running : 
 ![Architecture](../logs/task6/6.2_port.PNG)
 `docker ps` output : [logs/task6/6.2_ps.log](../logs/task6/6.2_ps.log)

 * We now ran a third node `s3`. HAProxy's stat page now shows the 3 applications running : 
 ![Architecture](../logs/task6/6.3_port.PNG)
 `docker ps` output : [logs/task6/6.3_ps.log](../logs/task6/6.3_ps.log)

 * Finally, we stopped `s1`. HAProxy's stat page then shows s1 as stopped : 
 ![Architecture](../logs/task6/6.4_port.PNG)
 and after a few seconds deletes it : 
 ![Architecture](../logs/task6/6.4_port2.PNG)
 `docker ps` output : [logs/task6/6.4ps.log](../logs/task6/6.4_ps.log)
   
2. **Give your own feelings about the final solution. Propose improvements or ways to do the things differently. If any, provide references to your readings for the improvements.**

	When we add/delete a node, the system freezes during a few seconds. There is a timelapse in which our application can't be reached. The system's availability might not be sufficient for some highly-available applications (high availability systems).  
	Yelp has been able to reach a system close to zero downtime in their system. They use fast reload that uses `SO_REUSEPORT` to bind to the same ports that the old HAProxy is listening to and sends a signal to the old HAProxy instance to shut down.  
	The maintainer of `HAProxy` suggested to drop SYN packets for the duration of the reload.  
	
	**Reference links**
	* [Yelp - Zero downtime](https://engineeringblog.yelp.com/2015/04/true-zero-downtime-haproxy-reloads.html)  
	* [marc.info - Drop SYN packets](http://marc.info/?l=haproxy&m=133262017329084&w=2) 

  
   
## <a name="difficulties"></a>Difficulties

We didn't encounter specific difficulties. The lab was very well-explained, and the given explainations and links given were very useful.  
We already had some experience working with Docker, so we were quite familiar with most of the commands. Understanding the deep functionning of the different scripts was perhaps the most tricky part. But again, the good documentation given helped us a lot. 

## <a name="conclusion"></a>Conclusion
This lab was very interesing, and we learned a lot about Docker and HAProxy. Reflecting on dynamic implementation of the proxy was fun to do.  
We realize the importance of Docker in a professional environment, and being familiar with it is only an upside for our future career !
