# Docker the Zerg way

For a resilient installation we are going to build a docker swarm of 3 master nodes with also a worker role.

(In swarm node can be master or worker - the one running containers - or both)

It's important to understand the split brain concept. In the following cluster a quorum have to be kept to assure things don't go sideways.
For the quorum to be achieve you should have 2 master nodes up. This is true for Swarm and GlusterFS but apply in many cluster concept.

Nodes gonna be called: swarm1, swarm2, swarm3

```
+----------------------+          |          +----------------------+
| [Node #1] |10.0.0.51 |          |          | [Node #1] |10.0.0.52 |
|   swarm1.lab.local   +----------+----------+   swarm2.lab.local   |
|                      |          |          |                      |
+----------------------+          |          +----------------------+
                                  |                  
                                  |               
                                  |
+----------------------+          |
| [Node #3] |10.0.0.53 |          |
|   swarm3.lab.local   +----------+
|                      |
+----------------------+
```


Why Swarm ?

Swarm is an orchestration tool directly provided by docker from the version 1.12. It is easy to use compare to Kubernetes and easier to maintain for a small team.

- Highly-available (can tolerate the failure of a single component)
- Scalable (can add resource or capacity as required)
- Portable (run it on your home today, run it in everywhere tomorrow)
- Automated (requires minimal care and feeding)

Why GlusterFS ?

While Docker Swarm is great for keeping containers running and providing scaling capabilities, it does lack direct integration of persistent storage across nodes. This means if you actually want your containers to keep any data persistent after restarts , you need to provide shared storage to every docker node. This also means you shouldn't use docker volume declaration in you docker files but prefer bind declaration (the fact to directly declare local directory binding into a container).
## Installing the host component

Installation is based on a fresh Centos Stream minimal server and should be executed on all nodes.
