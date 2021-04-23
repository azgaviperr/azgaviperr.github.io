
# Installing GlusterFS Server
![GlusterFS](../../images/Gluster.png)
## Adding Gluster Repository
```
# dnf install centos-release-gluster`
```

## Installing GlusterFS Server
```
# dnf install glusterfs-server
```

## A bit of simplification for management.

### Generate ssh-key
```
# ssh-keygen -t ed25519
```

## Push ssh-key to all nodes
```
# ssh-copy-id root@swarm2
# ssh-copy-id root@swarm3
```
This should be done on all nodes.

### Stop firewall

For now we will stop the firewall so it's not going to interfere in this configuration. But I higly recommend to look on the documentation to know what port should be open.

I will not give the command in here to do so, RTFM.

## Configure GlusterFS

### generate a private key for each node
```
# openssl genrsa -out /etc/ssl/glusterfs.key 4096`
```

### generate a signed certificate for each node
```
# openssl req -new -x509 -key /etc/ssl/glusterfs.key -subj "/CN=`hostname -f`" -out /etc/ssl/glusterfs.pem
```

hostname -f : for full fqdn (swarm1.lab.local)
hostname -s : for short version (swarm1)

This is important to understand what are you going to use on all the nodes as it's going to be used  to identify the allowed host to mount Gluster's volume.


## Enable and start service

```
# systemctl enable glusterd --now
```
### Peer nodes

From swarm1 run :

```
# gluster peer probe swarm2
# gluster peer probe swarm3
```

Verify peers:

```
# gluster peer status
Number of Peers: 2
Hostname: swarm2
Uuid: 1085c1ff-6b79-4021-b6d4-93db50b8709a
State: Peer in Cluster (Connected)

Hostname: swarm3
Uuid: 68525dfd-e91c-4563-9e16-76131025dc68
State: Peer in Cluster (Connected)

```
## Create GlusterFS Volumes

We are going to create two separated volumes.
One for the different compose files and on for the env volumes.
We could have made only one GlusterFS volume for both but I like to separate elements.

One thing you have to keep in mind, GlusterFS use bricks in between nodes that are folder. Do not write directly in it but on a mount of it.

## Compose and env Volumes

We gonna start by creating the brick
```
# mkdir -p /var/glusterfs/no-direct-write-here/vol_configuration/brick1
# ssh root@swarm2 mkdir -p /var/glusterfs/no-direct-write-here/vol_configuration/brick1
# ssh root@swarm3 mkdir -p /var/glusterfs/no-direct-write-here/vol_configuration/brick1
```
Create a volume from it.

```
# gluster volume create vol_configuration replica 3 transport tcp swarm{1..3}:/var/glusterfs/no-direct-write-here/vol_configuration/brick1 force

volume create: vol_configuration: success: please start the volume to access data
```
Let's activate tcp encryption.
```
# gluster volume set vol_configuration client.ssl on
volume set: success

# gluster volume set vol_configuration server.ssl on
volume set: success
```
Time to allow our nodes to mount the volume as client and brick data as server.
```
# gluster volume set vol_configuration auth.ssl-allow swarm1.lab.local,swarm2.lab.local,swarm3.lab.local
volume set: success
```
And now we start it.
```
# gluster volume start vol_configuration
volume start: vol_configuration: success
```
Now we need to create a mount point for it.
```
# mkdir  /mnt/configuration_data/
# ssh root@swarm2 mkdir  /mnt/configuration_data/
# ssh root@swarm3 mkdir  /mnt/configuration_data/
```
And now we mount it.
```
# mount -t glusterfs localhost:/vol_configuration /mnt/configuration_data
# ssh root@swarm2 mount -t glusterfs localhost:/vol_configuration /mnt/configuration_data
# ssh root@swarm3 mount -t glusterfs localhost:/vol_configuration /mnt/configuration_data
```
We check this is working as it should:
```
# touch /mnt/configuration_data/youpi
# ssh root@swarm2 ls /mnt/configuration_data
# ssh root@swarm3 ls /mnt/configuration_data
youpi
```

And that ssl is activated
```
grep ssl /var/log/glusterfs/mnt-configuration_data.log
```
We want to have it mounted on startup so we add this to /etc/fstab of each node:
```
echo "localhost:/vol_configuration /mnt/configuration_data                glusterfs defaults,_netdev,noauto,x-systemd.automount,x-systemd.requires=gulsterd.service,x-systemd.after=gulsterd.service 0 0
```
All the x-systemd option are here to make sure the volume is mounted after glusterFS service is running. For more information read it here https://www.freedesktop.org/software/systemd/man/systemd.mount.html .

## Docker Data Volumes

Same as above but with the brick made here : ```/var/glusterfs/no-direct-write-here/vol_docker/brick1```
And mountpoint: ```/mnt/docker_data/```

## Funny stuff

Imagine you got your brick (/var/glusterfs/no-direct-write-here/vol_configuration/brick1) and swarm1 on a dedicated hard drive and this hard drive died for some reason.
You still could use the hosts to run containers, just mount the volume from an other node  ```mount -t glusterfs localhost:/vol_configuration /mnt/configuration_data```

You could have also use two bricks or more instead of one to create the GlusterFS volume quite like you will have done it with Raid1. But this is to be read on the Official documentation of the project.


## Troubleshoot no GlusterFS mount on startup

I encountered issue for the glusterFS saying a fresh peer couldn't be added because it had existing volumes. This was due to the fact I forgot to recreate the gluster.ca file to add it's public certificate.


Official Documentation : https://docs.gluster.org/en/latest/
