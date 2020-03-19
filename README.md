# REAT VM Setup

Setup and test a base VM that runs:
- Docker network
- VNC container
- DNS container
- 6 Redis Labs node containers that appear to students as VMs.

Each student gets a VM with 6 RL nodes running in 2 clusters (north and south). Each node runs in a container but we make it look like a separate VM with its own IP. Each student gets a VNC desktop to their main VM where they can open browser sessions to admin consoles and SSH terminals to node shells and run rladmin.

The DNS and VNC containers need to be configured as follows:
- DNS - https://docs.google.com/document/d/1pDRZ8rHaR05UF4bU5SvwVkbM6FFj58apNsfxByibwoA/edit#heading=h.2gwmy0vc9jkp
- VNC - https://docs.google.com/document/d/1X8K2jZTwBLr_jG9a01-u_dxRrTwb2URNCy0zXoQZUG4/edit#

Build the following:
- Base VM, snapshot, and image with no DNS server
- Base VM, snapshot, and image with DNS
- Test instance with VNC.

Steps:

1. Create a VPC with subnet 172.18.0.0/16 in the region where you want to run VMs.

Requirement | Specification
------------|--------------
Name | rat-vpc
Subnet Creation Mode | Custom
Subnet Name | rat-subnet
Subnet IP Address Range | 172.18.0.0/16

2. Create a firewall rule that allows ingress on all ports from all sources (0.0.0.0/0) to all targets.
 
3. Create the base VM in the region and VPC where you want to run instances.
  
Requirement  | Specification  
------------ | -------------
Name | rat-no-dns
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

8. Edit trainee's .bashrc file to uncomment the line '#force_color_prompt' so 'trainee' user shell prompt on the 'controller' VM is green. Shell prompt color for 'default' user on the VNC container is yellow. Shell prompt color for 'root' users on RL nodes is white.

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

10. Create bashrc with aliases, and a set of scripts for students so they can:
- Start, stop, and SSH to RL nodes as if they were on machines instead of containers
- Run a Redis server in a container for lab 2 that can be discarded when done
- Run DNS Utils in a container to diagnose DNS on the Docker network.

```bash
cat << EOF > vnc_docker/bashrc
source \$STARTUPDIR/generate_container_user

export PS1='\e[1;33m\u@\h\e[m:\e[1;34m\w\e[m\$ '

alias ssh_installer="ssh trainee@\$INT_IP"

alias start_redis='ssh -t trainee@\$INT_IP docker run -it --name redis -h redis -w / redis bash'
alias stop_redis='ssh -t trainee@\$INT_IP docker container rm \$\(docker container ls -q -f '\''status=exited'\''\)'

alias run_dnsutils="ssh -t trainee@\$INT_IP ./scripts/run_dnsutils.sh "

alias ssh_n1="ssh -t trainee@\$INT_IP docker exec -it n1 bash "
alias ssh_n2="ssh -t trainee@\$INT_IP docker exec -it n2 bash "
alias ssh_n3="ssh -t trainee@\$INT_IP docker exec -it n3 bash "
alias ssh_s1="ssh -t trainee@\$INT_IP docker exec -it s1 bash "
alias ssh_s2="ssh -t trainee@\$INT_IP docker exec -it s2 bash "
alias ssh_s3="ssh -t trainee@\$INT_IP docker exec -it s3 bash "

alias start_n1="ssh -t trainee@\$INT_IP docker start n1 "
alias start_n2="ssh -t trainee@\$INT_IP docker start n2 "
alias start_n3="ssh -t trainee@\$INT_IP docker start n3 "

alias stop_n1="ssh -t trainee@\$INT_IP docker stop n1 "
alias stop_n2="ssh -t trainee@\$INT_IP docker stop n2 "
alias stop_n3="ssh -t trainee@\$INT_IP docker stop n3 "

alias start_s1="ssh -t trainee@\$INT_IP docker start s1 "
alias start_s2="ssh -t trainee@\$INT_IP docker start s2 "
alias start_s3="ssh -t trainee@\$INT_IP docker start s3 "

alias stop_s1="ssh -t trainee@\$INT_IP docker stop s1 "
alias stop_s2="ssh -t trainee@\$INT_IP docker stop s2 "
alias stop_s3="ssh -t trainee@\$INT_IP docker stop s3 "

alias start_north_nodes="ssh -t trainee@\$INT_IP ./scripts/start_north_nodes.sh "
alias start_south_nodes="ssh -t trainee@\$INT_IP ./scripts/start_south_nodes.sh "

alias create_north_cluster="ssh -t trainee@\$INT_IP ./scripts/create_north_cluster.sh "
alias create_south_cluster="ssh -t trainee@\$INT_IP ./scripts/create_south_cluster.sh "

EOF

mkdir scripts

cat << EOF > scripts/start_north_nodes.sh
docker kill n1; docker rm n1;
docker kill n2; docker rm n2;
docker kill n3; docker rm n3;
docker run -d --cap-add=ALL --name n1 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname n1.rlabs.org --net rlabs --ip 172.18.0.21 redislabs/redis
docker run -d --cap-add=ALL --name n2 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname n2.rlabs.org --net rlabs --ip 172.18.0.22 redislabs/redis
docker run -d --cap-add=ALL --name n3 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname n3.rlabs.org --net rlabs --ip 172.18.0.23 redislabs/redis
docker exec --user root n1 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root n2 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
docker exec --user root n3 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
sleep 60
EOF

cat << EOF > scripts/start_south_nodes.sh
docker kill s1; docker rm s1;
docker kill s2; docker rm s2;
docker kill s3; docker rm s3;
docker run -d --cap-add=ALL --name s1 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname s1.rlabs.org --net rlabs --ip 172.18.0.31 redislabs/redis
docker run -d --cap-add=ALL --name s2 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname s2.rlabs.org --net rlabs --ip 172.18.0.32 redislabs/redis
docker run -d --cap-add=ALL --name s3 -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --hostname s3.rlabs.org --net rlabs --ip 172.18.0.33 redislabs/redis
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

11. Create the Docker network so you can run RL nodes, Redis Insight, and an internal DNS server on a private network.

```bash
mkdir resolve
echo 'nameserver 172.18.0.20' > resolve/resolv.conf

docker network create --subnet=172.18.0.0/16 rlabs


```

12. Build a Docker image for the VNC container. The container runs from the VM startup script when you create student instances. It receives the VM's internal IP for aliases so students can 'silently' SSH from VNC container to base VM and RL nodes as if they were on their own machines. 

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
docker build -t re-vnc .


```

You're finished creating the base VM minus a DNS server. Now it's time to configure the DNS server and test your configuration. This may change over time so it's good to save your work in a GCP image so you don't have to do these steps again.

13. Create a snapshot from the VM called 'rat-no-dns'.

14. Create an image from the snapshot called 'rat-no-dns'.

This image has the following on it:
- Docker network 'rlabs'
- Docker images 'console/vnc' and 're-vnc'
- No Docker containers running yet.

15. Create a new VM called 'rat-dns' from the image 'rat-no-dns' with the following startup script. This starts the VNC container so you can sign in to the desktop and configure the DNS server with its WebMin UI.

NOTE: There seems to be two different sets of 'awk' commands that work on GCP instances these days.

Here's the original script with one 'awk' command that still works.

```bash
docker run -e INT_IP=`/sbin/ifconfig | grep -A 1 ens4 | grep inet | awk -F ' ' '{ print $2 }'` -p 80:6901 -e VNC_PW=trainee! --net rlabs --ip 172.18.0.2  --name controller -h controller.rlabs.org -d re-vnc
```

Here's the newer one with two 'awk' commands that works on newer and larger VMs.

```bash
docker run -e INT_IP=`/sbin/ifconfig | grep -A 1 ens4 | grep inet | awk -F : '{ print $2 }' | awk -F ' ' '{ print $1}'` -p 80:6901 -e VNC_PW=trainee! --net rlabs --ip 172.18.0.2  --name controller -h controller.rlabs.org -d re-vnc
```

There are two ways to access the VM at this point and you'll use them both for different reasons. First, you'll use the VNC desktop to start and configure the DNS server running in a container. Second, you'll SSH to VM from GCP console to save the modified DNS container and push it up to GCR (Google Container Registry). It helps to save our DNS configuration work and make it available to everyone is RedisLabs so others can reconfigure the lab environment if they need to.

16. Point your laptop browser to the VM's public IP on port 80. You can get the public IP from GCP admin console.

17. Sign in to VNC desktop with password 'trainee!'.

18. Open a terminal shell window.

This puts you in a shell running in the VNC container. It's on the 'rlabs' Docker network and cannot start and stop other containers on the network. For that, you need to SSH back down to base VM as the 'trainee' user. This user has sudo permissions to manage containers on the Docker network. You need to start the default, empty DNS Bind server in a container running on the Docker network.

19. Run the following command to make sure the VMs internal IP address got set properly.

```bash
echo $INT_IP


```

20. If it doesn't match the interal IP address listed for the VM in the GCP console, you'll have to start another VM using the other startup script with different 'awk' commands.

21. Run the following command to see the list of alias commands you and students can run from the VNC terminal (host named 'controller').

```bash
alias


```

22. Make sure the VMs internal IP is entered correctly in the alias commands as well. 

23. Run the following command to SSH down to the base VM running the Docker network and containers (at this point, VNC is the only running container).

```bash
ssh_installer


```

24. Enter the following command to run a default DNS Bind server on the Docker network.

Perform the following steps to configure the DNS server so it can resolve hostnames on the private Docker network.

Below are steps to run a BIND DNS server or a CoreDNS server. Only BIND works right now. CoreDNS doesn't resolve cluster names yet.

BIND DNS

```bash
docker run --name bind -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always --net rlabs --hostname ns.rlabs.org --ip 172.18.0.20 -p 10000:10000/tcp sameersbn/bind


```

CoreDNS - Create Corefile and rlabs.db, put them in /home/trainee/coredns/, and run:

```bash
docker run --name coredns -d -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf -h ns.rlabs.org --net rlabs --restart=always  -v /home/trainee/coredns/:/root/ --ip 172.18.0.20 coredns/coredns -conf /root/Corefile


```

25. Using BIND DNS:
- Open a Chrome browser on the VNC desktop
- Point it to the address: https://172.18.0.20:10000
- Sign in as 'root' with 'password'
- Configure the server according to steps here:
https://docs.google.com/document/d/1pDRZ8rHaR05UF4bU5SvwVkbM6FFj58apNsfxByibwoA/edit#heading=h.2gwmy0vc9jkp
  
Now you should have a DNS server configured to resolve host and cluster names.

26. Run the following command to return the VNC container on Docker network. 

```bash
exit


```

27. Run this script to start a container with DNS Utils in it so you can test your DNS config. This script is run from the default user in the VNC container, which transparently SSH's down to the base VM as the 'trainee' user and starts the container there. This is a common practice that students will use to start and stop containers on the Docker network so it looks to them like they're using VMs instead of containers.

```bash
./scripts/run_dnsutils.sh


```

28. Run the following commands to make sure DNS is working

```bash
nslookup n1.rlabs.org
nslookup s1.rlabs.org
dig @ns.rlabs.org north.rlabs.org
dig @ns.rlabs.org south.rlabs.org


```

If working, these commands should resolve to 172.18.0.21 for 'n1', and 'n1', 'n2', and 'n3' for 'north'.

If not, you'll need to stop and remove the DNS server container and start again until you get it working.

29. Once working, stop and take a Docker image of the changes to the DNS server container so you're work can be saved and made available to others at RedisLabs. You'll first commit the changes to the local Docker repo. Then you'll upload that image to the GCR repo.

30. Using the second way to access your VM, SSH to it in GCP console. This signs you in with RedisLabs employee credentials so you can upload the Docker image to GCR.

31. To do this, you'll need to:
- Commit container changes to the local Docker repo
- Stop the original DNS server container
- Rerun the command above that you used to start the default DNS server, substituting the Docker image for the new one in the local repo
- Retest DNS to make sure the new DNS Docker image works.
- Get a service account JSON key and apply it so Docker can authenticate to GCR and upload your new image to the remote repo
- Upload the new image to the GCR repo.

See instructions here for details:
https://docs.google.com/document/d/1pDRZ8rHaR05UF4bU5SvwVkbM6FFj58apNsfxByibwoA/edit#heading=h.2gwmy0vc9jkp

Now you'll test the new DNS Docker image download from GCR to make sure your DNS changes work in a new VM.

32. Start a new VM using the same steps as before for starting 'rat-dns'. Make sure to include the startup script to run the VNC container.

33. Use the same command you used before to start the default DNS BIND server, substituting in the new GCR Docker image 'gcr.io/redislabs-university/rat-dns' for 'sameersbn/bind'.

34. Rerun your DNS Utils tests above.

If it works, you're ready to take a backup of your work into another GCP VM snapshot and image so you don't have to configure DNS anymore unless it changes. This step is a bit tricky because you need to remove the VNC container so you can re-add when you start a new VM with that VMs internal IP (otherwise you'll try to use the old VM's IP which won't work).

35. From the terminal you SSH'd in from GCP console, stop and remove the VNC container.

```bash
docker stop controller
docker rm controller


```

36. Make sure the container is no longer present.

```bash
docker ps -a


```

37. Create a snapshot of the VM called 'rat-dns-ready'.

38. Create an image from the snapshot called 'rat-dns-ready'.

39. Start a new VM from the 'rat-dns-done' image and re-enter the startup script from before to run the VNC container.

40. Get your new VM's public IP and sign in to the VNC console on port 80 from a laptop browser.

41. Open a terminal shell and run the following to make sure you got your new VMs internal IP.

```bash
echo $INT_IP


```

42. Run the following to make sure your DNS resolves hostnames.

```bash
run_dnsutils
nslookup n1.rlabs.org


```

43. Run scripts to start Redis Labs nodes running in their containers (3 north, 3 south).

```bash
start_north_nodes
start_south_nodes


```

44. Run scripts to build Redis Labs clusters from their nodes.

```bash
create_north_cluster.sh
create_south_cluster.sh


```

45. Run the following to make sure DNS is resolving cluster names.

NOTE: Cluster names will not resolve until you start the nodes and create the cluster because pDNS needs to be running.

```bash
dig @ns.rlabs.org north.rlabs.org
dig @ns.rlabs.org south.rlabs.org


```

46. In VNC desktop, open Chrome browser and point it to RL admin consoles on port 8443. You can use either hostnames or IPs.

```bash
n1 = 172.18.0.21
n2 = 172.18.0.22
n3 = 172.18.0.23

s1 = 172.18.0.31
s2 = 172.18.0.32
s3 = 172.18.0.33
```

47. Create a database from node 'n1' listening on port '12000'.

48. Run the following in the terminal to SSH to node 'n2'.

49. Run the following and make sure the proxy is listening on 'node:1'.

50. Run the following to connect to the redis-cli using DNS resolution:

```bash
redis-cli -p 12000 -h north.rlabs.org


```

51. Set a key in the database.

52. Run the Redis Insight utility app as a container so students can view database contents in a GUI.

```bash
ssh_installer
docker run -d --name insight -v redisinsight:/db -v /home/trainee/resolve/resolv.conf:/etc/resolv.conf --restart=always  --hostname insight.rlabs.org --net rlabs --ip 172.18.0.4  redislabs/redisinsight


```

Test that Redis Insight can connect to the database using DNS as well.

53. Point the Chrome browser on the VNC desktop to http://insight:8001.

54. Accept the terms and click Add Database.

55. Give the DB any name

56. Enter hostname: north.rlabs.org or redis-12000.north.rlabs.org

57. Enter port: 12000

58. Click connect.

59. Click Browse - you should see the key you created.

Now you done and ready to create student instances, but first take another VM snapshot and image to save all your work.

60. Stop and remove the VNC container.

61. Take another VM snapshot and image called 'rat-ready'.

'rat-ready' is the image you'll use to creeate student instances.

When creating student instances remember to include the startup scripts which runs the VNC container.


Here are some additional commands you can run as the 'default' user on the VNC terminal.

```bash
# SSH to RL nodes so you can run rladmin
ssh_n1
rlcheck
rladmin status
redis-cli -p 12000 -h redis-12000.north.rlabs.org
exit
exit
ssh_n2
exit
ssh_n3
exit

# start, test, and stop OSS Redis container
start_redis
> redis-server &
> redis-cli
127.0.0.1:6379> keys *
(empty list)
127.0.0.1:6379> exit
exit
stop_redis

# SSH to the main VM as user 'trainee' with sudo/root permissions
ssh_installer
sudo docker images
sudo docker ps
sudo docker network list
sudo docker network inspect rlabs
sudo docker image inspect redislabs/redis
exit
```

If you want to add dnsutils to test DNS on the rlabs network run
```bash
run_dnsutils
```

