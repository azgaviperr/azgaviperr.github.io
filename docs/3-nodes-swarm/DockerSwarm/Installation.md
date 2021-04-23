# Docker swarm

![Docker Swarm](../../images/docker-swarm-containers.png)


# Installing Docker and Docker compose

## Remove runc
```
# dnf remove runc
```
## Adding docker repo
```
# dnf install -y yum-utils
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`
# dnf install docker-ce docker-ce-cli containerd.io
```

## Installing docker-compose
```
# curl -L "https://github.com/docker/compose/releases/download/1.29.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
# chmod +x /usr/local/bin/docker-compose

```
## Optimizing Hosts for para-virtualization

Now that Swarm is ready, there are some system requirements that need to be met before really enjoying Swarm.

Indeed, it is necessary to disable the SWAP memory, increase the maximum number of open file descriptors, increase the number of mmap or increase the number of connections that can be requested.

To do this, edit the /etc/sysctl.conf file and add the following lines:

```
# echo >> /etc/sysctl.conf
vm.swappiness=1           # 1: minimum amount of swapping without disabling it entirely
net.core.somaxconn=65535  # Increase number of incoming connections This is the max a system can handle for concurrent connection. Not recommended for directly exposed server
vm.max_map_count=262144   # This is to avoid out of memory exceptions on stuff like elasticsearch and index tech
fs.file-max=518144        # Up the default limit  of concurrent open files, this issue happen more than expected on huge container environment

```
## Let's start the whale

```
# systemctl enable docker --Now

```

## And now create a zerg warm from it

From swarm1:
```
# docker swarm init --advertise-addr 10.0.0.51
Swarm initialized: current node (ksdjlqsldjqsd2516685485) is now a manager.
To add a worker to this swarm, run the following command:

    docker swarm \
     join --token SWMTKN-1-5pykfhyfvtsij0tg4ewrtqk7hz2twuq21okeqv54p1gw2ufdde-814yer1z55vmyk2mwdhvjbob1 \
     10.0.0.51:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
We are going to add the two other nodes as manager:
```
docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-5pykfhyfvtsij0tg4ewrtqk7hz2twuq21okeqv54p1gw2ufdde-2k0vay9aub5eheikw7qi9v82o 10.0.0.51:2377
```
Run the command provided on your other nodes to join them to the swarm as managers.
After addition of a node, the output of docker node ls (on either host) should reflect all the nodes:
```
# docker node ls
ID                            HOSTNAME           STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
p424u0yvmu0vvc8nsnspv83zw *   swarm1.lab.local   Ready     Active         Leader           20.10.6
kg6w6ucpb2jf8v8xqai23pv3a     swarm2.lab.local   Ready     Active         Reachable        20.10.6
lam7mgs5wus40iaydvp8u3ss7     swarm3.lab.local   Ready     Active         Reachable        20.10.6
```

You are now ready to swarm.

Official Documentation : https://docs.docker.com/engine/swarm/
