#Portainer
![Portainer](https://raw.githubusercontent.com/portainer/portainer/develop/app/assets/images/logo_alt.png)
Portainer is a lightweight management UI which allows you to easily manage your different Docker environments (Docker hosts or Swarm clusters). Portainer is meant to be as simple to deploy as it is to use. It consists of a single container that can run on any Docker engine (can be deployed as Linux container or a Windows native container, supports other platforms too). Portainer allows you to manage all your Docker resources (containers, images, volumes, networks and more!) It is compatible with the standalone Docker engine and with Docker Swarm mode.

# Preparaton

## Setup Docker Swarm
Deploying Portainer and the Portainer Agent to manage a Swarm cluster is easy! You can directly deploy Portainer as a service in your Docker cluster. Note that this method will automatically deploy a single instance of the Portainer Server, and deploy the Portainer Agent as a global service on every node in your cluster.

```
# mkdir -p /mnt/configuration_data/compose_files/administration
# curl -L https://downloads.portainer.io/portainer-agent-stack.yml -o mnt/configuration_data/compose_files/administration/portainer-agent-stack.yml
```
## Customisation

By default portainer stack is using a normal docker volume. So if the node where the container will run fail, then we lost the data.
To counter this we are going to use a local file shared accross nodes by GlusterFS.

Edit portainer-agent-stack.yml

Change:
 `portainer_data:/data ` for  `/mnt/docker_data/administration/portainer_data:/data `

Remove:
```
volumes:
  portainer_data:
```

The file should look like this (I added some notes as comment):
```
version: '3.2'

services:
  agent:
    image: portainer/agent
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - agent_network
    deploy:
      mode: global # Will be deploy on all nodes manager or worker accross the swarm.
      placement:
        constraints: [node.platform.os == linux] # Will only run on Linux OS.

  portainer:
    image: portainer/portainer-ce
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      - "9000:9000"
      - "8000:8000"
    volumes:
      - /mnt/docker_data/administration/portainer_data:/data
    networks:
      - agent_network
    deploy:
      mode: replicated
      replicas: 1 # Only one container of this type gonna run at a time.
      placement:
        constraints: [node.role == manager] # Will only be deployed on a manager node.

networks:
  agent_network:
    driver: overlay
    attachable: true

```
## Setup data locations

One issue you gonna have with swarm is that it's not creating folder for declared volume on it's own so you will have to create it by yourself.

```
# mkdir -p /mnt/docker_data/administration/portainer_data
```

If like me you are using the same base directory for docker data, you can run this on the compose file to automatically build the folder.

```
grep "/mnt/docker_data" /mnt/configuration_data/compose_files/administration/portainer-agent-stack.yml | awk -F "- " '{print $2'} | awk -F ":" '{ system("mkdir -p "$1"") }'
```
This could be made as a command but I like to run first and echo instead of the "mkdir -p" to be sure I didn't made a mistake.

# Deploy Portainer stack

Deploy the Portainer stack by running `docker stack deploy -c <path -to-docker-compose.yml> portainer`

Log into your new instance at any nodes on port 9000. You'll be prompted to set your admin user/password on first login. Start at "Home", and click on "Primary" to manage your swarm (you can manage multiple swarms via one Portainer instance using the agent

###Todo
- Add image
- Add Some basic usage
