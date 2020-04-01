# Admin Training VM Setup

This documents provides steps to create a VM image from scratch.

Instructors spin up VMs from the image.

Each VM provides:
- 6 RE nodes
- A DNS server
- A VNC desktop
- Redis Insight.

as follows: https://drive.google.com/open?id=17tM53iHHTu-DQNPD48dYQEAias0qmqz30hpP2embUV4

Nodes run in containers, but they look like they run on VMs.

Users access the VM by VNC on port 80.  All they need is the VM public IP and VNC password.

Setup is in 2 stages:
1. Start Docker/VNC and configure DNS
2. Configure VNC.

# Stage 1

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
Name | admin-training-stage-1
CPU | 4
Memory | 15 GB
OS | Ubuntu 18.04 LTS
Disk | 30 GB
Networking | admin-training-vpc
  
4. SSH to the base VM from GCP console to finish setup.

5. Install vim and add 'trainee' user to the 'docker' group so it can start, stop, and SSH to containers.

```bash 
sudo su
apt -y update
apt -y install vim
 
update-alternatives --config editor <<< 3

adduser --disabled-password --gecos "" trainee
groupadd docker
usermod -aG docker trainee
 
```

6. Install Docker.

```bash
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common <<< Y
    
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

apt-key fingerprint 0EBFCD88
   
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

apt-get update
apt-get -y install docker-ce
 
```

7. Run 'sudo visudo' and add the following line so 'trainee' can start and stop containers without 'sudo'.

```bash
trainee ALL=(ALL) NOPASSWD:ALL
```


8. Switch to 'trainee' to create the Docker network, add scripts, and build/run the VNC image.

```bash
sudo su - trainee
 
```

9. Uncomment the line '#force_color_prompt' in the user's .bashrc file. This sets VM user prompt to 'green' so you can tell it apart from the VNC user.

10. Generate keys so students can 'silently' SSH from VNC user to base VM user and start/stop/SSH RE nodes. 

```bash
mkdir .ssh
ssh-keygen -q -t rsa -N '' -f .ssh/id_rsa 2>/dev/null <<< y >/dev/null
cp -r .ssh/id_rsa.pub .ssh/authorized_keys 

mkdir vnc_docker
cp -r .ssh/ vnc_docker/ssh 
 
```

11. Create bashrc with aliases to run necessary commands.

```bash
cat << EOF > vnc_docker/bashrc
source \$STARTUPDIR/generate_container_user

export PS1='\e[1;33m\u@\h\e[m:\e[1;34m\w\e[m\$ '

#ssh-keygen -f "/headless/.ssh/known_hosts" -R 172.18.0.1

alias create_north_cluster="ssh -t trainee@172.18.0.1 ./scripts/create_north_cluster.sh "
alias create_south_cluster="ssh -t trainee@172.18.0.1 ./scripts/create_south_cluster.sh "

alias run_dnsutils="ssh -t trainee@172.18.0.1 ./scripts/run_dnsutils.sh "

alias run_redis_start="ssh -t trainee@172.18.0.1 docker run -it --name redis -h redis -w / redis bash"
alias run_redis_stop="ssh -t trainee@172.18.0.1 docker container rm \$\(docker container ls -q -f '\''status=exited'\''\)"

alias ssh_base-vm="ssh -t trainee@172.18.0.1"        # only used by admins when building VMs

alias start_north_nodes="ssh -t trainee@172.18.0.1 ./scripts/start_north_nodes.sh "
alias start_south_nodes="ssh -t trainee@172.18.0.1 ./scripts/start_south_nodes.sh "

alias start_n1="ssh -t trainee@172.18.0.1 docker start n1 "
alias start_n2="ssh -t trainee@172.18.0.1 docker start n2 "
alias start_n3="ssh -t trainee@172.18.0.1 docker start n3 "
alias start_s1="ssh -t trainee@172.18.0.1 docker start s1 "
alias start_s2="ssh -t trainee@172.18.0.1 docker start s2 "
alias start_s3="ssh -t trainee@172.18.0.1 docker start s3 "

alias stop_n1="ssh -t trainee@172.18.0.1 docker stop n1 "
alias stop_n2="ssh -t trainee@172.18.0.1 docker stop n2 "
alias stop_n3="ssh -t trainee@172.18.0.1 docker stop n3 "
alias stop_s1="ssh -t trainee@172.18.0.1 docker stop s1 "
alias stop_s2="ssh -t trainee@172.18.0.1 docker stop s2 "
alias stop_s3="ssh -t trainee@172.18.0.1 docker stop s3 "
EOF

mkdir scripts

cat << EOF > scripts/start_north_nodes.sh
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

cat << EOF > scripts/start_south_nodes.sh
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

cat << EOF > scripts/create_north_cluster.sh
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

cat << EOF > scripts/create_south_cluster.sh
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
chmod 755 scripts/start_north_nodes.sh
chmod 755 scripts/start_south_nodes.sh
chmod 755 scripts/create_north_cluster.sh
chmod 755 scripts/create_south_cluster.sh
chmod 755 scripts/run_dnsutils.sh
 
```

12. Create the Docker network.

```bash
mkdir resolve
echo 'nameserver 172.18.0.20' > resolve/resolv.conf

docker network create --subnet=172.18.0.0/16 rlabs
 
```

13. Build the VNC Docker image and run it as a container.

```bash
cat << EOF > vnc_docker/Dockerfile
## Custom Dockerfile
FROM consol/ubuntu-xfce-vnc

# Switch to root user to install additional software
USER 0

## Install SSH, DNS Utils, and the VNC user's bashrc and chromium-browser.init
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

docker run --name vanilla-vnc  -d -e VNC_PW=trainee! --restart=always --net rlabs --hostname vnc-terminal.rlabs.org --ip 172.18.0.2 -p 80:6901  vanilla-vnc
 
```

You have VNC running and can configure Bind DNS with its GUI.

14. Point your browser to the VM public IP (it's in GCP console).

15. Sign in to VNC with password 'trainee!'.

16. Open a terminal shell window.

17. Run Bind DNS as a container.

```bash
ssh_base-vm
docker run --name vanilla-dns -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --net rlabs --hostname ns.rlabs.org --ip 172.18.0.20 -p 10000:10000/tcp  sameersbn/bind
 
```

Someday, you may use CoreDNS with Corefile and rlabs.db.

```bash
docker run --name vanilla-dns -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf -v /home/trainee/coredns/:/root/ --restart=always --net rlabs -hostname ns.rlabs.org --ip 172.18.0.20  coredns/coredns -conf /root/Corefile
```

Configure Bind DNS.

18. Open Chrome browser on the VNC desktop.

19. Point it to https://172.18.0.20:10000 (this is Bind's GUI).

20. Sign in as 'root' with 'password'.

21. Configure the server using these steps.

https://drive.google.com/open?id=1QBFCyuU9uUikC5jEy1QwiEe4ln2S7MzekM0qke1Y9Jg 

22. Return to the VNCs shell terminal.

23. Run the following to test DNS is working (n1 = .21, n2 = .22).

```bash
exit
./scripts/run_dnsutils.sh
nslookup n1.rlabs.org
nslookup s1.rlabs.org
 
```

24. If not:
- SSH to the base VM
- Stop and remove DNS container and try again.

25. From GCP console, SSH to the VM and push DNS changes to a GCR image. This requires 'write' access to the 'redislabs-university' repo. For this, a service account key was created and stored in GCS.

NOTE: If you accidentally run this command as 'trainee', you'll get 'config.json' errors later when try to run containers because the 'trainee' user will check the GCR repo. If you run this as 'trainee' by mistake, you can always log in as 'root' and remove the directory /home/trainee/.docker which contains the contfig.json file.

```bash
# In your GCP employee account, download key and authenticate Docker to GCR
# You have to run exit twice to get from 'trainee' to 'root' and 'root' to <your employee GCP account>
exit
exit
gsutil cp gs://admin-training-bucket/ru-gcr-write-key.json /tmp
cat /tmp/ru-gcr-write-key.json | sudo docker login -u _json_key --password-stdin https://gcr.io

# Commit DNS changes to image, tag it for upload, and push the image.
sudo docker commit vanilla-dns admin-training-dns
sudo docker tag admin-training-dns gcr.io/redislabs-university/admin-training-dns
sudo docker push gcr.io/redislabs-university/admin-training-dns 
 
```

26. Replace the DNS container with the GCR image.

```bash
sudo docker stop vanilla-dns
sudo docker rm vanilla-dns
sudo docker rmi sameersbn/bind
sudo docker rmi admin-training-dns

sudo docker run --name configured-dns -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --net rlabs --hostname ns.rlabs.org --ip 172.18.0.20 -p 10000:10000/tcp  gcr.io/redislabs-university/admin-training-dns
 
```

27. Return to the VNCs shell terminal and run the following to make sure DNS still works.

```bash
run_dnsutils
nslookup n1.rlabs.org
 
```

28. SSH to the base VM and add Redis Insight as a container so students can view database contents in a GUI.

```bash
exit
ssh_base-vm
docker run --name insight -d -v redisinsight:/db -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --net rlabs --hostname insight.rlabs.org --ip 172.18.0.4  redislabs/redisinsight
 
```

You have the following:
- A vanilla VNC container
- A configured DNS server
- Redis Insight.

Save your work.

29. Create a snapshot of the VM called 'admin-training-stage-1'.

30. Create an image from the snapshot called 'admin-training-stage-1'.





# Stage 2

You have the following:
- A vanilla VNC container
- A configured DNS container
- Redis Insight.

Now you'll configure the VNC container with:
- A background image and 2 workspaces
- A Chrome launcher
- 2 terminal launchers to nodes
- 2 terminal launchers to VNC and the base VM.

1. Create a new VM called 'admin-training-stage-2' from image 'admin-training-stage-1'.

NOTE: May need to run with startup script:
```bash
ssh-keygen -f "/headless/.ssh/known_hosts" -R 172.18.0.1
```

Configure VNC.

2. Sign in to VNC desktop from your laptop browser with password 'trainee!'.

3. Open a terminal shell window.

Before configuring VNC, make sure:
- Nodes start and join clusters
- DNS resolves cluster names
- Databases get accessed by cluster names from command line and Insight.


4. Start all Redis Enterprise nodes in containers (3 north, 3 south).

```bash
start_north_nodes
start_south_nodes
 
```

5. Build Redis Enterprise clusters from nodes.

```bash
create_north_cluster
create_south_cluster
 
```

6. Run the following to make sure DNS is resolving cluster names.

NOTE: Cluster names only resolve after you join nodes to clusters.

```bash
run_dnsutils
dig @ns.rlabs.org north.rlabs.org
dig @ns.rlabs.org south.rlabs.org
 
```

7. Open Chrome browser in VNC and point it to admin consoles on port 8443 as follows:

```bash
https://n1:8443
https://n2:8443
https://n3:8443
https://s1:8443
https://s2:8443
https://s3:8443
```

8. Create a database from node 'n1' listening on port '12000'.

9. Run the following in a terminal to SSH to node 'n2'.

```bash
ssh_n2
 
```

10. Connect redis-cli to the database from node n2.

```bash
redis-cli -p 12000 -h north.rlabs.org
 
```

Test the database connection from Redis Insight.

11. Open another tab in Chrome and point it the Redis Insight at:

```bash
http://insight:8001
```

12. Click Add Database to add the database to Redis Insight and enter the following:
Name | anything
Host | north.rlabs.org
Port | 12000
Password | blank

13. Make sure it connects.

14. Click Browse to see the key you set.

Now you know DNS, Redis Insight, and your node and cluster scripts work.

15. Open another tab in Chrome and point it to the DNS UI and sign in with 'root' and 'password'.

```bash
https://172.18.0.20:10000
```

16. Save the 8 tabs as bookmarks and save the pages to open on startup to the current pages and then fix the setting to always open like that.

17. Follow steps here to set up the VNC desktop with a Redis background image, 2 workspaces, and launchers for opening Chrome and shell terminals in specific locations and opening tabs to node command line shells.

https://docs.google.com/document/d/1X8K2jZTwBLr_jG9a01-u_dxRrTwb2URNCy0zXoQZUG4/edit#heading=h.ihyyp7do45r3

Push the VNC container to a GCR image. Then download and test it.

18. SSH to the VM 'from GCP console' so you're using your GCP account to perform the next steps.

19. Download the service account key again and authenticate Docker to GCR with the following commands.

```bash
gsutil cp gs://admin-training-bucket/ru-gcr-write-key.json /tmp/
cat /tmp/ru-gcr-write-key.json | sudo docker login -u _json_key --password-stdin https://gcr.io
 
```

20. Commit changes to the VNC container in a local image, tag it for GCR, and upload it.

```bash
sudo docker commit vanilla-vnc admin-training-vnc
sudo docker tag admin-training-vnc gcr.io/redislabs-university/admin-training-vnc
sudo docker push gcr.io/redislabs-university/admin-training-vnc
 
```

21. Replace the VNC container and local image with the GCR image and test it.

```bash
sudo docker stop vanilla-vnc
sudo docker rm vanilla-vnc
sudo docker rmi vanilla-vnc
sudo docker rmi consol/ubuntu-xfce-vnc
sudo docker rmi admin-training-vnc

sudo docker run --name configured-vnc  -d -e VNC_PW=trainee! --restart=always --net rlabs --hostname vnc-terminal.rlabs.org --ip 172.18.0.2 -p 80:6901  gcr.io/redislabs-university/admin-training-vnc
 
```

22. Test VNC still works.


23. Retart nodes in containers so they are running but not configured in a cluster.

```bash
start_north_nodes
start_south_nodes
 
```

You have the following:
- DNS container and image
- VNC container and image
- Redis Insight container
- Nodes running.

You're ready to create user instances.

Save your work.

24. Create a snapshot of the VM called 'admin-training-stage-2'.

25. Create an image from the snapshot called 'admin-training-stage-2'.

