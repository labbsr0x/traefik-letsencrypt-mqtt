# MQTT over WSS
Traefik with lets encrypt enabled proxing MQTT over WSS connections to VerneMQ and Mosquitto brokers

## Running using Docker Standalone

Just copy and paste the following contents to a **docker-compose.yml** file and type `docker-compose up` and open `http://localhost:8080` to see the traefik panel.

```
version: '3.5'

services:

  traefik:
    image: tiagostutz/traefik-letsencrypt:1.6
    network_mode: bridge
    ports:
      - 80:80
      - 443:443
      - 8080:8080
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - SWARM_MODE=false
      - LETS_ENCRYPT_EMAIL=you@yourawesomedomain.com
      - LETS_ENCRYPT_TEST_MODE=true
    labels:
      - traefik.enable=false

  vernemq:
    image: erlio/docker-vernemq
    network_mode: bridge
    environment:
      - DOCKER_VERNEMQ_ALLOW_ANONYMOUS=on
      - DOCKER_VERNEMQ_LOG__CONSOLE__LEVEL=info
    labels:
      - traefik.backend=vernemq
      - traefik.frontend.rule=Host:verne.mqtt.yourawesomedomain.com
      - traefik.frontend.entryPoints=https
      - traefik.port=8080
      - traefik.enable=true

  mosquitto:
    image: flaviostutz/mosquitto
    network_mode: bridge
    labels:
      - traefik.backend=mosquitto
      - traefik.frontend.rule=Host:mosquitto.mqtt.yourawesomedomain.com
      - traefik.frontend.entryPoints=https
      - traefik.port=9001
      - traefik.enable=true
```

## Running using Docker Swarm

### Usage

Create a network outside of the compose to prevent issues with the `traefik.docker.network` label. The issue happens because its value must be the complete name of the network, but when you create the network in the compose file, the complete name of the network will depend on the name of the deployed stack, which you can't be sure what will be. If the value provided is not the complete name of the network, traefik will have issues to address the proxied containers in case they are attached to more than one network.

`docker create network --driver=overlay traefik-net`

Then bring the stack up:

`docker stack deploy --compose-file docker-compose.yml secure-mqtt-broker`


```
version: '3.5'

services:

  traefik:
    image: tiagostutz/traefik-letsencrypt:1.6
    ports:
      - 80:80
      - 443:443
      - 8080:8080
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - SWARM_MODE=true
      - LETS_ENCRYPT_EMAIL=test@yourawesomedomain.com
      - LETS_ENCRYPT_TEST_MODE=false

  vernemq:
    image: erlio/docker-vernemq
    ##
    #  PORT MAPPING NOT ALLOWED
    #  BUG: If ports are mapped the service will be attached to the ingress network 
    #  and will have issues on the connection handshake
    ##
    environment:
      - DOCKER_VERNEMQ_ALLOW_ANONYMOUS=on
      - DOCKER_VERNEMQ_LOG__CONSOLE__LEVEL=info
    deploy:
      labels:
        - traefik.backend=vernemq
        - traefik.frontend.rule=Host:vernemq.yourawesomedomain.com
        - traefik.frontend.entryPoints=https
        - traefik.port=8080
        - traefik.enable=true
        - traefik.docker.network=traefik-net


  mosquitto:
    image: flaviostutz/mosquitto
    # ports:
    #   - 1885:1883
    #   - 9005:9001
    deploy:
      labels:
        - traefik.backend=mosquitto
        - traefik.frontend.rule=Host:mosquitto.yourawesomedomain.com
        - traefik.frontend.entryPoints=https
        - traefik.port=9001
        - traefik.enable=true
        - traefik.docker.network=traefik-net

networks:
  default:
    name: traefik-net
    external: true

```
## Preventing docker_gwbridge network issues


```
{
  "debug" : true,
  "bip" : "192.168.128.1/19",
  "experimental" : true
}
```


`docker network rm docker_gwbridge`

```
docker network create  \
--subnet 192.168.160.1/19     \
--gateway 192.168.160.1   \
-o com.docker.network.bridge.enable_icc=false \
-o com.docker.network.bridge.name=docker_gwbridge \
docker_gwbridge
```

`docker network inspect docker_gwbridge`

`docker swarm init`


## Setup without TLS/WSS
To run this setup without TLS support you don't need the Let's Encrypt part. The "vanilla" traefik will do the trick for you. Check this: https://hub.docker.com/_/traefik/
