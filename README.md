# Admin Training VM Setup - Step 1

Setup a base VM with:
- VNC desktop
- Docker network
- DNS server
- 6 Redis Labs nodes
- Redis Insight
- and DNS Utils.

Here's a description of what you get.
https://drive.google.com/open?id=17tM53iHHTu-DQNPD48dYQEAias0qmqz30hpP2embUV4

A gcloud command will be provided so employees can spin up their own VM rather than set them up from scratch.

VM setup is broken up into 3 stages below:
1. Start Docker network and a vanilla VNC desktop
2. Setup and save a DNS server
3. Setup and save the VNC desktop.

During setup, you'll save your work along the way (VM and Docker images) so you can skips steps later.

Nodes look like they run on VMs, but they really run in containers.

A user accesses a VM with a VNC desktop running in a browser on port 80. All users need is the VM public IP and VNC password.

From there, users can:
- Configure and investigate DNS
- Start and stop nodes
- Join nodes in clusters
- Add databases
- Access databases from command line or Redis Insight
- Investigate failover of nodes, databases, and cluster.

Set up:

1. Create a VPC in GCP with subnet 172.18.0.0/16 in the region where you want to run VMs.

Requirement | Specification
------------|--------------
Name | admin-training-vpc
Subnet Creation Mode | Custom
Subnet Name | admin-training-subnet
Subnet IP Address Range | 172.18.0.0/16

2. Create a firewall rule that allows ingress on all ports from all sources (0.0.0.0/0) to all targets.
 
3. Create the base VM in the region and VPC where you want to run instances.
  
Requirement  | Specification  
------------ | -------------
Name | admin-training-image-step-1
CPU | 4
Memory | 15 GB
OS | Ubuntu 18.04 LTS
Disk | 30 GB
Networking | reat-vpc
  
4. SSH to the base VM from GCP console to finish setup.

5. Install 'vi', add user 'trainee', add the user to the 'docker' group, and give the student permissions to run Docker commands without entering 'sudo'.

```bash 
# install and set default editor to vim
# then add 'trainee' user to 'docker' group so it can start and stop containers that run Redis software
sudo su
apt -y update
apt -y install vim

update-alternatives --config editor


```

Enter '3' to set the editor type.

```bash
adduser --disabled-password --gecos "" trainee
groupadd docker
usermod -aG docker trainee


```

Add the following line to /etc/sudoers using "sudo visudo" so students can start and stop containers without entering 'sudo'.

```bash
trainee ALL=(ALL) NOPASSWD:ALL
```

6. Install Docker.

```bash
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common 


```

and enter 'Y'.

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

apt-key fingerprint 0EBFCD88
   
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

apt-get update
apt-get -y install docker-ce


```

7. Switch to 'trainee' user to create the Docker network, add user scripts, and build the VNC Docker image.

```bash
sudo su - trainee


```

8. Edit trainee's .bashrc file and uncomment the line '#force_color_prompt'. This sets the prompt to 'green'. Shell prompt color for the VNC user is 'yellow'. Shell prompt color for 'root' users on Redis Enterprise nodes is 'white'. This way users can tell them apart.

9. Generate keys so students can 'silently' SSH from VNC container to base VM and RL nodes as if they were on their own machines. 

```bash
#
# reset
# rm -rf .ssh vnc_docker
#
mkdir .ssh
ssh-keygen -q -t rsa -N '' -f .ssh/id_rsa 2>/dev/null <<< y >/dev/null
cp -r .ssh/id_rsa.pub .ssh/authorized_keys 

mkdir vnc_docker
cp -r .ssh/ vnc_docker/ssh 


```

10. Create bashrc with aliases so users can:
- Start, stop, and SSH to nodes as if they were on VMs
- Run Redis in a separate container for lab 2
- Run DNS Utils.

```bash
cat << EOF > vnc_docker/bashrc
source \$STARTUPDIR/generate_container_user

export PS1='\e[1;33m\u@\h\e[m:\e[1;34m\w\e[m\$ '

alias run_dnsutils="ssh -t trainee@\$INT_IP ./scripts/run_dnsutils.sh "

alias run_north_cluster="ssh -t trainee@\$INT_IP ./scripts/run_north_cluster.sh "
alias run_north_nodes="ssh -t trainee@\$INT_IP ./scripts/run_north_nodes.sh "

alias run_redis_start="ssh -t trainee@\$INT_IP docker run -it --name redis -h redis -w / redis bash"
alias run_redis_stop="ssh -t trainee@\$INT_IP docker container rm \$\(docker container ls -q -f '\''status=exited'\''\)"

alias run_south_cluster="ssh -t trainee@\$INT_IP ./scripts/run_south_cluster.sh "
alias run_south_nodes="ssh -t trainee@\$INT_IP ./scripts/run_south_nodes.sh "

alias ssh_base-vm="ssh -t trainee@\$INT_IP"        # only used by admins when building VMs

alias start_n1="ssh -t trainee@\$INT_IP docker start n1 "
alias start_n2="ssh -t trainee@\$INT_IP docker start n2 "
alias start_n3="ssh -t trainee@\$INT_IP docker start n3 "
alias start_s1="ssh -t trainee@\$INT_IP docker start s1 "
alias start_s2="ssh -t trainee@\$INT_IP docker start s2 "
alias start_s3="ssh -t trainee@\$INT_IP docker start s3 "

alias stop_n1="ssh -t trainee@\$INT_IP docker stop n1 "
alias stop_n2="ssh -t trainee@\$INT_IP docker stop n2 "
alias stop_n3="ssh -t trainee@\$INT_IP docker stop n3 "
alias stop_s1="ssh -t trainee@\$INT_IP docker stop s1 "
alias stop_s2="ssh -t trainee@\$INT_IP docker stop s2 "
alias stop_s3="ssh -t trainee@\$INT_IP docker stop s3 "
EOF

mkdir scripts

cat << EOF > scripts/run_north_nodes.sh
docker kill n1; docker rm n1;
docker kill n2; docker rm n2;
docker kill n3; docker rm n3;
docker run --name n1 -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --cap-add=ALL --net rlabs --hostname n1.rlabs.org --ip 172.18.0.21 redislabs/redis
docker run --name n2 -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --cap-add=ALL --net rlabs --hostname n2.rlabs.org --ip 172.18.0.22 redislabs/redis
docker run --name n3 -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --cap-add=ALL --net rlabs --hostname n3.rlabs.org --ip 172.18.0.23 redislabs/redis
docker exec --user root n1 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root n2 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root n3 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
sleep 60
EOF

cat << EOF > scripts/run_south_nodes.sh
docker kill s1; docker rm s1;
docker kill s2; docker rm s2;
docker kill s3; docker rm s3;
docker run --name s1 -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --cap-add=ALL --net rlabs --hostname s1.rlabs.org --ip 172.18.0.31  redislabs/redis
docker run --name s2 -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --cap-add=ALL --net rlabs --hostname s2.rlabs.org --ip 172.18.0.32 redislabs/redis
docker run --name s3 -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --cap-add=ALL --net rlabs --hostname s3.rlabs.org --ip 172.18.0.33 redislabs/redis
docker exec --user root s1 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root s2 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root s3 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
sleep 60
EOF

cat << EOF > scripts/run_north_cluster.sh
docker exec -it n1 bash -c "/opt/redislabs/bin/rladmin cluster create persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.21 \
        name north.rlabs.org username admin@rlabs.org password admin";

docker exec -it n2 bash -c "/opt/redislabs/bin/rladmin cluster join persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.22 \
        username admin@rlabs.org password admin nodes 172.18.0.21";

docker exec -it n3 bash -c "/opt/redislabs/bin/rladmin cluster join persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.23 \
        username admin@rlabs.org password admin nodes 172.18.0.21";
EOF

cat << EOF > scripts/run_south_cluster.sh
docker exec -it s1 bash -c "/opt/redislabs/bin/rladmin cluster create persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.31 \
        name south.rlabs.org username admin@rlabs.org password admin";
docker exec -it s2 bash -c "/opt/redislabs/bin/rladmin cluster join persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.32 \
        username admin@rlabs.org password admin nodes 172.18.0.31";
docker exec -it s3 bash -c "/opt/redislabs/bin/rladmin cluster join persistent_path \
        /var/opt/redislabs/persist ephemeral_path /var/opt/redislabs/tmp addr 172.18.0.33 \
        username admin@rlabs.org password admin nodes 172.18.0.31";
EOF

cat << EOF > scripts/run_dnsutils.sh
docker kill dnsutils; docker rm dnsutils
docker run --name dnsutils -it -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --net rlabs --hostname dnsutils.rlabs.org --ip 172.18.0.6 tutum/dnsutils
EOF

# make the scripts executable
chmod 755 scripts/run_north_nodes.sh
chmod 755 scripts/run_south_nodes.sh
chmod 755 scripts/run_north_cluster.sh
chmod 755 scripts/run_south_cluster.sh
chmod 755 scripts/run_dnsutils.sh


```

11. Create the Docker network.

```bash
mkdir resolve
echo 'nameserver 172.18.0.20' > resolve/resolv.conf

docker network create --subnet=172.18.0.0/16 rlabs


```

12. Build the VNC Docker image. The container starts later in a VM startup script.

```bash
cat << EOF > vnc_docker/Dockerfile
## Custom Dockerfile
FROM consol/ubuntu-xfce-vnc

# Switch to root user to install additional software
USER 0

## Install SSH, DNS Utils, and the VNC user's bashrc
RUN apt update; apt install -y ssh dnsutils;
RUN mkdir /headless/.ssh
COPY ./ssh /headless/.ssh
RUN chown -R 1000 /headless/.ssh/
COPY bashrc /headless/.bashrc
RUN chown -R 1000 /headless/.bashrc

## switch back to default user
USER 1000
EOF

cd vnc_docker
docker build -t vanilla-vnc .


```

You have the following:
- A Docker network
- A vanilla VNC Docker image.

Save your work.

13. Create a snapshot from the VM called 'admin-training-step-1'.

14. Create an image from the snapshot called 'admin-training-step-1'.


# Admin Training VM Setup - Step 2

You have the following:
- A Docker network
- A vanilla VNC Docker image.

Configure a DNS server to resolve host and cluster names on the Docker network.

1. Create a new VM called 'admin-training-step-2' from image 'admin-training-step-1'.

Add startup script below to run VNC so you can configure DNS using a GUI.

The script also passes the internal VM IP to VNC so users can SSH to the base VM where they start and stop node containers. Sometimes VMs report IPs using different formats, so a second script is provided. If the first one doesn't work, you'll find out in a later step and have to try again using the second script.

Here's the original script that usually works.

```bash
docker run --name vanilla-vnc  -d -e INT_IP=`/sbin/ifconfig | grep -A 1 ens4 | grep inet | awk -F ' ' '{ print $2 }'`  -e VNC_PW=trainee! --net rlabs --hostname vnc-terminal.rlabs.org --ip 172.18.0.2 -p 80:6901  vanilla-vnc
```

Here's the second script (this one has a 2 'awk' filters).

```bash
docker run --name vanilla-vnc -d -e INT_IP=`/sbin/ifconfig | grep -A 1 ens4 | grep inet | awk -F : '{ print $2 }' | awk -F ' ' '{ print $1}'` -e VNC_PW=trainee! --net rlabs --hostname vnc-terminal.rlabs.org --ip 172.18.0.2  -p 80:6901  vanilla-vnc
```

Configure DNS.

2. Get the VM public IP from GCP console.

3. Point your laptop browser to the public IP on port 80.

4. Sign in to VNC desktop with password 'trainee!'.

5. Open a terminal shell window.

6. Make sure the internal IP is set correctly. If not, start another VM with the other 'awk' filter.

```bash
echo $INT_IP


```

7. See the list of alias commands users can run from VNC.

```bash
alias


``` 

8. Run the following command to SSH to the base VM. From there, you can run the vanilla DNS server as a container and attach it to the Docker network.

```bash
ssh_base-vm


```

9. View the list of Docker containers running on the VM and make sure VNC's the only one.

```bash
docker ps


```

10. Run the following command to start a vanilla Bind server in a container and attach it to the private Docker network.

```bash
docker run --name vanilla-dns -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --net rlabs --hostname ns.rlabs.org --ip 172.18.0.20 -p 10000:10000/tcp  sameersbn/bind


```

Someday, you may be able to run this command to start CoreDNS using only two text files: Corefile and rlabs.db in /home/trainee/coredns/:

```bash
docker run --name vanilla-dns -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf -v /home/trainee/coredns/:/root/ --restart=always --net rlabs -hostname ns.rlabs.org --ip 172.18.0.20  coredns/coredns -conf /root/Corefile


```

11. Open Chrome browser on the VNC desktop.

12. Point it to https://172.18.0.20:10000 (this is Bind's GUI called WebMin).

13. Sign in as 'root' with 'password'.

14. Configure the server using these steps.

https://drive.google.com/open?id=1QBFCyuU9uUikC5jEy1QwiEe4ln2S7MzekM0qke1Y9Jg 

15. Return to the VNCs shell terminal.

16. Enter the following commands to exit the base VM SSH session run and DNS Utils to test DNS setup.

```bash
exit
./scripts/run_dnsutils.sh

```

17. Run 'nslookup' to make sure hostnames resolve as follows:

```bash
nslookup n1.rlabs.org
nslookup s1.rlabs.org


```

```bash
n1 = 172.18.0.21
s1 = 172.18.0.31
```

18. If not, run the following commands to destroy the Bind DNS container and try again until it works.

NOTE: Skip these commands if DNS is working.

```bash
exit
ssh_base-vm
docker stop vanilla-dns
docker rm vanilla-dns


```

Now you're ready to commit DNS changes to a Docker image in GCR.

19. Create a service account key with write permissions to the gcr.io/redislabs-university repository in GCR.

20. SSH to the VM from GCP console so you're using your GCP account to perform the next steps.

21. Commit changes to the vanilla DNS container to a local Docker image first and tag it for upload to GCR.

```bash
sudo docker commit vanilla-dns admin-training-dns
sudo docker tag admin-training-dns gcr.io/redislabs-university/admin-training-dns


```

The subsequent 'docker push' command requires Docker to first authenticate to GCR with 'write' access to the 'redislabs-university' repo. We'll use GCS to transfer the key file to the VM.

22. From you local laptop, upload the key file to the 'admin-training-bucket' in GCS with no extra characters.

```bash
gsutil cp . gs://admin-training-bucket/ru-gcr-write-key.json


```

23. Download the key from GCS to the base VM with the following commands.

```bash
cd /tmp
gsutil cp gs://admin-training-bucket/ru-gcr-write-key.json .


```

24. Authenticate Docker to GCR with the service account key.

```bash
cat /tmp/ru-gcr-write-key.json | sudo docker login -u _json_key --password-stdin https://gcr.io


```

25. Make sure you get the following.

```bash
Login Succeeded
```

26. Push the configured Bind DNS server image to GCR.

```bash
sudo docker push gcr.io/redislabs-university/admin-training-dns 


```

Replace the running DNS container and local Docker images with the GCR image and test it. The final VM image you create will download and run the configured DNS server from its GCR image.

27. Remove the DNS container and local images.

```bash
sudo docker stop vanilla-dns
sudo docker rm vanilla-dns
sudo docker rmi sameersbn/bind
sudo docker rmi admin-training-dns


```

28. Run the DNS server from the GCR image.

```bash
docker run --name configured-dns -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --net rlabs --hostname ns.rlabs.org --ip 172.18.0.20 -p 10000:10000/tcp  gcr.io/redislabs-university/admin-training-dns


```

29. Test DNS still works.

You have the following:
- A Docker network
- A vanilla VNC Docker image
- A configured DNS container and GCR image.

Save your work.

30. Create a snapshot of the VM called 'admin-training-step-2'.

31. Create an image from the snapshot called 'admin-training-step-2'.


# Admin Training VM Setup - Step 3

You have the following:
- A Docker network
- A vanilla VNC Docker image
- A configured DNS container and GCR image.

Now you'll configure the vanilla VNC container to provide a consistent layout for users that includes:
- A background image and 2 workspaces
- A Chrome launcher with tabs and bookmarks to all node admin consoles, Redis Insight, and the DNS UI
- 2 terminal launchers to the VNC terminal and base VM
- 2 terminal launchers with 3 tabs each, opened and SSHed to 'north' and 'south' Redis Enterprise nodes.

1. Create a new VM called 'admin-training-step-3' from image 'admin-training-step-2'.

Add the startup script to run VNC again.

```bash
docker run --name vanilla-vnc  -d -e INT_IP=`/sbin/ifconfig | grep -A 1 ens4 | grep inet | awk -F ' ' '{ print $2 }'`  -e VNC_PW=trainee! --net rlabs --hostname vnc-terminal.rlabs.org --ip 172.18.0.2 -p 80:6901  vanilla-vnc
```

Configure DNS.

2. Get the VM public IP from GCP console.

3. Point your laptop browser to the public IP on port 80.

4. Sign in to VNC desktop with password 'trainee!'.

5. Open a terminal shell window.

6. Make sure the internal IP is set correctly. If not, start another VM with the other 'awk' filter.

```bash
echo $INT_IP


```

7. Run the following to make sure your DNS resolves hostnames.

```bash
run_dnsutils
nslookup n1.rlabs.org


```

Before configuring VNC, add Redis Insight and make sure:
- Nodes start and join clusters
- DNS resolves cluster names
- Databases get accessed by cluster names from command line and Insight.


8. Add Redis Insight as a container so students can view database contents in a GUI.

```bash
exit
ssh_base-vm
docker run --name insight -d -v redisinsight:/db -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --net rlabs --hostname insight.rlabs.org --ip 172.18.0.4  redislabs/redisinsight


```

9. Start all Redis Enterprise nodes in containers (3 north, 3 south).

```bash
run_north_nodes
run_south_nodes


```

10. Build Redis Enterprise clusters from nodes.

```bash
run_north_cluster
run_south_cluster


```

11. Run the following to make sure DNS is resolving cluster names.

NOTE: Cluster names only resolve in DNS after you join nodes to clusters (pDNS needs to run).

```bash
run_dnsutils
dig @ns.rlabs.org north.rlabs.org
dig @ns.rlabs.org south.rlabs.org


```

12. Open Chrome browser in VNC and point it to admin consoles on port 8443 as follows:

```bash
https://n1:8443
https://n2:8443
https://n3:8443
https://s1:8443
https://s2:8443
https://s3:8443
```

13. Create a database from node 'n1' listening on port '12000'.

14. Run the following in a terminal to SSH to node 'n2'.

```bash
ssh_n2


```

15. Connect redis-cli to the database from node n2.

```bash
redis-cli -p 12000 -h north.rlabs.org


```

16. Set a key in the database

Test the database connection from Redis Insight.

17. Open another tab in Chrome and point it the Redis Insight at:

```bash
http://insight:8001
```

18. Click Add Database to add the database to Redis Insight and enter the following:
Name | anything
Host | north.rlabs.org
Port | 12000
Password | blank

19. Make sure it connects.

20. Click Browse to see the key you set.

Now you know DNS, Redis Insight, and your node and cluster scripts work.

21. Open another tab in Chrome and point it to the DNS UI and sign in with 'root' and 'password'.

```bash
https://172.18.0.20:10000

```

22. Save the 8 tabs as bookmarks and save the pages to open on startup to the current pages and then fix the setting to always open like that.

23. Follow steps here to set up the VNC desktop with a Redis background image, 2 workspaces, and launchers for opening Chrome and shell terminals in specific locations and opening tabs to node command line shells.

https://docs.google.com/document/d/1X8K2jZTwBLr_jG9a01-u_dxRrTwb2URNCy0zXoQZUG4/edit#heading=h.ihyyp7do45r3

24. Stop the nodes before saving your work.

```bash
stop_n1
stop_n2
stop_n3
stop_s1
stop_s2
stop_s3


```

Replace the running VNC container and local Docker images with the GCR image and test it. The final VM image you create will download and run the configured VNC server from its GCR image at run time.

25. SSH to the VM from GCP console as your Redis Labs employee GCP account. You will:
- Remove the Redis Labs node containers 
- Commit and push a new Docker image of the VNC container to GCR
- Stop and remove the running VNC container and image
- Run a new VNC container (configured-vnc) from the GCR image and test it
- Stop and remove the new VNC container and image from the VM (no VNC running)
- Take a snapshot and image of the VM.

26. Remove Redis Labs node containers. If the containers are saved in the VM image, they will auto-start and auto-configure into clusters. You want students to start them so they build clusters from scratch.  Leave the local Docker image to save load time in class.

```bash
sudo docker rm n1
sudo docker rm n2
sudo docker rm n3
sudo docker rm s1
sudo docker rm s2
sudo docker rm s3


```

27. Commit changes to the vanilla VNC container (vnc) to a local Docker image first and tag it for upload to GCR.

```bash
sudo docker commit vnc admin-training-vnc
sudo docker tag admin-training-vnc gcr.io/redislabs-university/admin-training-vnc


```


28. Download the key from GCS to the base VM with the following commands.

```bash
cd /tmp
gsutil cp gs://admin-training-bucket/ru-gcr-write-key.json .


```

29. Authenticate Docker to GCR with the service account key.

```bash
cat /tmp/ru-gcr-write-key.json | sudo docker login -u _json_key --password-stdin https://gcr.io


```

30. Make sure you get the following.

```bash
Login Succeeded
```

31. Push the configured VNC server image to GCR.

```bash
sudo docker push gcr.io/redislabs-university/admin-training-vnc


```

Replace the running VNC container and local Docker image with the GCR image and test it.

32. Remove the VNC container and local images.

```bash
sudo docker stop vnc
sudo docker rm vnc
sudo docker rmi vanilla-vnc
sudo docker rmi admin-training-vnc


```

33. Run the VNC server from the GCR image.

IMPORTANT NOTE: You only have to run this command with 'sudo' this one time because this user needs sudo permissions. You could also have run these above Docker commands without 'sudo' if you switch user to 'trainee' after SSH from GCP console.

```bash
sudo docker run --name configured-vnc  -d -e INT_IP=`/sbin/ifconfig | grep -A 1 ens4 | grep inet | awk -F ' ' '{ print $2 }'`  -e VNC_PW=trainee! --net rlabs --hostname vnc-terminal.rlabs.org --ip 172.18.0.2 -p 80:6901  gcr.io/redislabs-university/admin-training-vnc


```


34. Test VNC still works.
- Open Chrome and node terminals in the 3 workspaces
- Run node containers and create clsuters
- Create a database and test DNS from node terminals and Redis Insight.

You have the following:
- A Docker network
- A configured VNC container and GCR image
- A configured DNS container and GCR image.

35. Stop and remove node containers one more time from your GCP SSH terminal. Otherwise, they will auto-start and auto-create clusters when users start the VM.

```bash
sudo docker stop n1
sudo docker stop n2
sudo docker stop n3
sudo docker stop s1
sudo docker stop s2
sudo docker stop s3

sudo docker rm n1
sudo docker rm n2
sudo docker rm n3
sudo docker rm s1
sudo docker rm s2
sudo docker rm s3


```

36. Stop the VNC container and remove the local image one more time. The image will load and the container will run when users or instructors create VM instances.

```bash
sudo docker stop configured-vnc
sudo docker rm configured-vnc
sudo docker rmi gcr.io/redislabs-university/admin-training-vnc


```

Save your work.

37. Create a snapshot of the VM called 'admin-training-step-3'.

38. Create an image from the snapshot called 'admin-training-step-3'.

The final VM image you create will download and run the configured VNC server from its GCR image with the start-up script (see below).

You're ready to create user instances.

You have the following:
- A Docker network
- A configured VNC Docker container
- A configured DNS container and GCR image.

39. When creating user instances remember to include the startup script which runs the VNC container and name your instances 'base-vm-01', 'base-vm-02', and so on.

```bash
docker run --name configured-vnc -d -e INT_IP=`/sbin/ifconfig | grep -A 1 ens4 | grep inet | awk -F ' ' '{ print $2 }'`  -e VNC_PW=trainee! --net rlabs --hostname vnc-terminal.rlabs.org --ip 172.18.0.2 -p 80:6901  gcr.io/redislabs-university/admin-training-vnc
```
