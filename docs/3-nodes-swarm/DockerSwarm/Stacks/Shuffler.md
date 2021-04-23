# Shuffle
![Shuffle](https://shuffler.io/images/logo.png)
Shuffle is an Open Source SOAR I really appreciate as it does focus on CyberSecurity.
To know more about it go there: [https://medium.com/shuffle-automation](https://medium.com/shuffle-automation/introducing-shuffle-an-open-source-soar-platform-part-1-58a529de7d12)
Or there : [Official Website](https://shuffler.io/)

Hi Frikky !!! Hoipefully, you will appreciate what you are seing here ^^

# Preparaton

## Setup data locations

## Setup Docker Swarm
Deploying Portainer and the Portainer Agent to manage a Swarm cluster is easy! You can directly deploy Portainer as a service in your Docker cluster. Note that this method will automatically deploy a single instance of the Portainer Server, and deploy the Portainer Agent as a global service on every node in your cluster.

```
# mkdir -p /mnt/configuration_data/compose_files/SOAR
```
## Customisation

At the time of this writing I may be the only one shufflying around with swarm :p
So there is not yet a stack ready compose file.

Let's create one `/mnt/configuration_data/compose_files/SOAR/shuffle.stack.yml`
```
version: '3.3'
services:
  backend:
    image: ghcr.io/frikky/shuffle-backend:0.8.75
    environment:
      DATASTORE_EMULATOR_HOST: shuffle-database:8000
      HTTPS_PROXY: ''
      HTTP_PROXY: ''
      ORG_ID: Shuffle
      SHUFFLE_APP_DOWNLOAD_LOCATION: https://github.com/frikky/shuffle-apps
      SHUFFLE_APP_FORCE_UPDATE: 'false'
      SHUFFLE_APP_HOTLOAD_FOLDER: /shuffle-apps
      SHUFFLE_DEFAULT_APIKEY: zaCELgL.0imfnc8mVLWwsAawjYr4Rx-Af50DDqtlx
      SHUFFLE_DEFAULT_PASSWORD: Fr1kky1sN0t@D0g
      SHUFFLE_DEFAULT_USERNAME: admin
      SHUFFLE_DOWNLOAD_AUTH_BRANCH: ''
      SHUFFLE_FILE_LOCATION: /shuffle-files
    ports:
     - 5001:5001
    volumes:
     - /var/run/docker.sock:/var/run/docker.sock
     - /mnt/docker_data/SOAR/shuffle_data/shuffle-demo/shuffle-apps:/shuffle-apps
     - /mnt/docker_data/SOAR/shuffle_data/shuffle-demo/shuffle-files:/shuffle-files
    networks:
     - shuffle
    logging:
      driver: json-file
    deploy:
      replicas: 3
  database:
    image: frikky/shuffle:database
    ports:
     - 30002:8000
    volumes:
     - /mnt/docker_data/SOAR/shuffle_data/shuffle-demo/shuffle-database:/etc/shuffle
    networks:
     - shuffle
    logging:
      driver: json-file
    deploy:
      replicas: 1
  frontend:
    image: ghcr.io/frikky/shuffle-frontend:0.8.75
    environment:
      BACKEND_HOSTNAME: shuffle-backend
    ports:
     - 3001:80
     - 3443:443
    networks:
     - shuffle
    logging:
      driver: json-file
    deploy:
      replicas: 3
  orborus:
    image: ghcr.io/frikky/shuffle-orborus:0.8.75
    environment:
      BASE_URL: http://shuffle-backend:5001
      CLEANUP: 'false'
      DOCKER_API_VERSION: '1.40'
      ENVIRONMENT_NAME: Shuffle
      HTTPS_PROXY: ''
      HTTP_PROXY: ''
      ORG_ID: Shuffle
      SHUFFLE_APP_SDK_VERSION: 0.8.75
      SHUFFLE_BASE_IMAGE_NAME: frikky
      SHUFFLE_BASE_IMAGE_REGISTRY: ghcr.io
      SHUFFLE_BASE_IMAGE_TAG_SUFFIX: -0.8.75
      SHUFFLE_ORBORUS_EXECUTION_TIMEOUT: '600'
      SHUFFLE_PASS_WORKER_PROXY: 'TRUE'
      SHUFFLE_WORKER_VERSION: 0.8.75
    volumes:
     - /var/run/docker.sock:/var/run/docker.sock
    networks:
     - shuffle
    logging:
      driver: json-file
    deploy:
      replicas: 3
networks:
  shuffle:
    driver: overlay
```
## Setup data locations

This could be made as a command but I like to run first and echo instead of the "mkdir -p" to be sure I didn't made a mistake.

```
# grep "/mnt/docker_data" /mnt/configuration_data/compose_files/SOAR/shuffle-stack.yml | awk -F "- " '{print $2'} | awk -F ":" '{ system("mkdir -p "$1"") }'
```


# Deploy Shuffle stack

Deploy the Shuffle stack by running `docker stack deploy -c <path -to-docker-compose.yml> shuffle`

Log into your new instance at any nodes on port https://node:3443.

# Note
Have you may have seen, all except database have a replica set to 3. This is done so we don't create a messy mess on DB if writing are done from different shuffle-database container at the same time on the same value which will lead to database corruption. Anyway, this container is up in 7 second in case of failure of the node ^^ .

App gonna be slow to run on it's first usage on all new nodes where Orborus is going to use the base image for the first time. As swarm distribute request around all replicas, sometime request gonna be fast as the node already run the image and sometime not.
This all Replica and Clustering of Shuffle is not yet production ready as it is not yet properly implemented.

I recommend using only one database and one Orborus for now.
