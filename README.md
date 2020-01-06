# Traefik v2 </br>guide by examples

requirements

- have docker running somewhere
- have a domain `whateverblablabla.org`
- use cloudflare to manage DNS of the domain
- have 80/443 ports open

chapters

1. [traefik routing to docker containers](#1-traefik-routing-to-various-docker-containers)
2. [traefik routing to a local IP addresses](#2-traefik-routing-to-a-local-IP-addresses)
3. [middlewares](#3-middlewares)
4. [let's encrypt certificate HTTP challenge](#4-lets-encrypt-certificate-HTTP-challenge)
5. [let's encrypt certificate DNS challenge](#5-lets-encrypt-certificate-DNS-challenge-on-cloudflare)
6. [redirect HTTP traffic to HTTPS](#6-redirect-HTTP-traffic-to-HTTPS)

# #1 traefik routing to various docker containers

![traefik-dashboard-pic](https://i.imgur.com/5jKHJmm.png)

- **create a new docker network** `docker network create traefik_net`.</br>
  Traefik and the containers need to be on the same network.
  Compose creates one automatically, but that fact is hidden and there is potential for a fuck up later on.
  Better to just create own network and set it as default in every compose file.

  *extra info:* use `docker network inspect traefik_net` to see containers connected to that network

- **create traefik.yml**</br>
  This file contains so called static traefik configuration.</br>
  In this basic example there is log level set, dashboard is enabled.
  Entrypoint called `web` is defined at port 80. Docker provider is set and given docker socket</br>
  Since exposedbydefault is set to false, a label `"traefik.enable=true"` will be needed
  for containers that should be routed by traefik.</br>
  This file will be passed to a docker container using bind mount,
  this will be done when we get to docker-compose.yml for traefik.

    `traefik.yml`
    ```
    ## STATIC CONFIGURATION
    log:
      level: INFO

    api:
      insecure: true
      dashboard: true

    entryPoints:
      web:
        address: ":80"

    providers:
      docker:
        endpoint: "unix:///var/run/docker.sock"
        exposedByDefault: false
    ```

  later on when traefik container is running, use command `docker logs traefik` 
  and check if there is a notice stating: `"Configuration loaded from file: /traefik.yml"`.
  You don't want to be the moron who makes changes to traefik.yml
  and it does nothing because the file is not actually being used.

- **create `.env`** file that will contain environmental variables.</br>
  Domain names, api keys, ip addresses, passwords,... 
  whatever is specific for one case and different for another, all of that ideally goes here.
  These variables will be available for docker-compose when running
  the `docker-compose up` command.</br>
  This allows compose files to be moved from system to system more freely and changes are done to the .env file,
  so there's a smaller possibility for a fuckup of forgetting to change domain name
  in some host rule in a big ass compose file or some such.

    `.env`
    ```
    MY_DOMAIN=whateverblablabla.org
    DEFAULT_NETWORK=traefik_net
    ```

  *extra info:*</br>
  command `docker-compose config` shows how compose will look
  with the variables filled in.
  Also entering container with `docker container exec -it traefik sh`
  and then `printenv` can be useful.

- **create traefik-docker-compose.yml file**.</br>
  It's a simple typical compose file.</br>
  Port 80 is mapped since we want traefik to be in charge of what comes on it - using it as an entrypoint.
  Port 8080 is for dashboard where traefik shows info. Mount of docker.sock is needed,
  so it can actually do its job interacting with docker.
  Mount of `traefik.yml` is what gives the static traefik configuration.
  The default network is set to the one created in the first step, as it will be set in all other compose files.

    `traefik-docker-compose.yml`
    ```
    version: "3.7"

    services:
      traefik:
        image: "traefik:v2.1"
        container_name: "traefik"
        hostname: "traefik"
        ports:
          - "80:80"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **run traefik-docker-compose.yml**</br>
  `docker-compose -f traefik-docker-compose.yml up -d` will start the traefik container.

  traefik is running, you can check it at the ip:8080 where you get the dashboard.</br>
  Can also check out logs with `docker logs traefik`.

  *extra info:*</br>
  Typically you see guides having just a single compose file called `docker-compose.yml`
  with several services/containers in it. Then it's just `docker-compose up -d` to start it all.
  You don't even need to bother defining networks when it is all one compose.
  But this time I prefer small and separate steps when learning new shit.
  So that's why going with custom named docker-compose files as it allows easier separation.

  *extra info2:*</br>
  What you can also see in tutorials is no mention of traefik.yml
  and stuff is just passed from docker-compose using traefik's commands or labels.</br>
  like [this](https://docs.traefik.io/getting-started/quick-start/): `command: --api.insecure=true --providers.docker`</br>
  But that way compose files look bit more messy and you still can't do everything from there,
  you still sometimes need that damn traefik.yml.</br>
  So... for now, going with nicely structured readable traefik.yml

- **add labels to containers that traefik should route**.</br>
  Here are examples of whoami, nginx, apache, portainer.</br>

  > \- "traefik.enable=true"

  enables traefik

  > \- "traefik.http.routers.whoami.entrypoints=web"

  defines router named `whoami` that listens on entrypoint web(port 80)

  > \- "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"

  defines a rule for this `whoami` router, specifically that when url
  equals `whoami.whateverblablabla.org` (the domain name comes from the `.env` file),
  that it means for router to do its job and route it to a service.
  
  Nothing else is needed, traefik knows the rest from the fact that these labels
  are coming from context of a docker container.

  `whoami-docker-compose.yml`
  ```
  version: "3.7"

  services:
    whoami:
      image: "containous/whoami"
      container_name: "whoami"
      hostname: "whoami"
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.whoami.entrypoints=web"
        - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

  `nginx-docker-compose.yml`
  ```
  version: "3.7"

  services:
    nginx:
      image: nginx:latest
      container_name: nginx
      hostname: nginx
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.nginx.entrypoints=web"
        - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

  `apache-docker-compose.yml`
  ```
  version: "3.7"

  services:
    apache:
      image: httpd:latest
      container_name: apache
      hostname: apache
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.apache.entrypoints=web"
        - "traefik.http.routers.apache.rule=Host(`apache.$MY_DOMAIN`)"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

  `portainer-docker-compose.yml`
  ```
  version: "3.7"

  services:
    portainer:
      image: portainer/portainer
      container_name: portainer
      hostname: portainer
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock:ro
        - portainer_data:/data
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.portainer.entrypoints=web"
        - "traefik.http.routers.portainer.rule=Host(`portainer.$MY_DOMAIN`)"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK

  volumes:
    portainer_data:

  ```

- **run the damn containers**</br>
  ignore some orphans talk, it's cuz these compose files are in the same directory
  and compose uses parent directory name for name of compose project

    `docker-compose -f whoami-docker-compose.yml up -d`</br>
    `docker-compose -f nginx-docker-compose.yml up -d`</br>
    `docker-compose -f apache-docker-compose.yml up -d`</br>
    `docker-compose -f portainer-docker-compose.yml up -d`

  *extra info:*</br>
  to stop all containers running: `docker stop $(docker ps -q)`

# #2 traefik routing to a local IP addresses

  When url should aim at something other than a docker container.

  ![simple-network-diagram-pic](https://i.imgur.com/lTpUvWJ.png)

- **define a file provider, add required routing and service**
  
  What is needed is a router that catches some url and route it to some IP.</br>
  Previous examples shown how to catch whatever url, on port 80,
  but no one told it what to do when something fits the rule.
  Traefik just knew since it was all done using labels in the context of a container and
  thanks to docker being set as a provider in `traefik.yml`.</br>
  For this "sending traffic at some IP" a traefik service is needed,
  and to define traefik service a new provider is required, a file provider - just a fucking stupid
  file that tells traefik what to do.</br>
  Somewhat common is to set `traefik.yml` itself as a file provider so thats what will be done.</br>
  Under providers theres a new `file` section and `traefik.yml` itself is set.</br>
  Then dynamic configuration stuff is added.</br>
  A router named `route-to-local-ip` with a simple subdomain hostname rule.
  What fits that rule, in this case exact url `test.whateverblablabla.org`,
  is send to the loadbalancer service which just routes it a specific IP and specific port.

    `traefik.yml`
    ```
    ## STATIC CONFIGURATION
    log:
      level: INFO

    api:
      insecure: true
      dashboard: true

    entryPoints:
      web:
        address: ":80"

    providers:
      docker:
        endpoint: "unix:///var/run/docker.sock"
        exposedByDefault: false
      file:
        filename: "traefik.yml"

    ## DYNAMIC CONFIGURATION
    http:
      routers:
        route-to-local-ip:
          rule: "Host(`test.whateverblablabla.org`)"
          service: route-to-local-ip-service
          priority: 1000
          entryPoints:
            - web

      services:
        route-to-local-ip-service:
          loadBalancer:
            servers:
              - url: "http://10.0.19.5:80"
    ```

  Priority of the router is set to 1000, a very high value,
  beating any possible other routers,
  like one we use later for doing global http -> https redirect.

  *extra info:*</br>
  Unfortunately the `.env` variables are not working here,
  otherwise domain name in host rule and that IP would come from a variables.
  So heads up that you will definitely forget to change these. 

- **run traefik-docker-compose** and test if it works

    `docker-compose -f traefik-docker-compose.yml up -d`</br>
    
# #3 middlewares

Example of an authentication middleware for any container.

![logic-pic](https://i.imgur.com/QkfPYel.png)

- **create a new file - `users_credentials`** containing username:passwords pairs,
 [htpasswd](https://www.htaccesstools.com/htpasswd-generator/) style</br>
 Bellow example has password `krakatoa` set to all 3 accounts

    `users_credentials`
    ```
    me:$apr1$L0RIz/oA$Fr7c.2.6R1JXIhCiUI1JF0
    admin:$apr1$ELgBQZx3$BFx7a9RIxh1Z0kiJG0juE/
    bastard:$apr1$gvhkVK.x$5rxoW.wkw1inm9ZIfB0zs1
    ```

- **mount users_credentials in traefik-docker-compose.yml**

    `traefik-docker-compose.yml`
    ```
    version: "3.7"

    services:
      traefik:
        image: "traefik:v2.1"
        container_name: "traefik"
        hostname: "traefik"
        ports:
          - "80:80"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./users_credentials:/users_credentials:ro"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **add two labels to any container** that should have authentication</br>
  - The first label attaches new middleware called `auth-middleware`
    to an already existing `whoami` router.
  - The second label gives this middleware type basicauth,
    and tells it where is the file it should use to authenticate users.

    No need to mount the `users_credentials` here, it's traefik that needs that file
    and these labels are a way to pass info to traefik, what it should do
    in context of containers.

  `whoami-docker-compose.yml`
  ```
  version: "3.7"

  services:
    whoami:
      image: "containous/whoami"
      container_name: "whoami"
      hostname: "whoami"
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.whoami.entrypoints=web"
        - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
        - "traefik.http.routers.whoami.middlewares=auth-middleware"
        - "traefik.http.middlewares.auth-middleware.basicauth.usersfile=/users_credentials"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

  `nginx-docker-compose.yml`
  ```
  version: "3.7"

  services:
    nginx:
      image: nginx:latest
      container_name: nginx
      hostname: nginx
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.nginx.entrypoints=web"
        - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
        - "traefik.http.routers.nginx.middlewares=auth-middleware"
        - "traefik.http.middlewares.auth-middleware.basicauth.usersfile=/users_credentials"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

- **run the damn containers** and now there is login and password needed

    `docker-compose -f traefik-docker-compose.yml up -d`</br>
    `docker-compose -f whoami-docker-compose.yml up -d`</br>
    `docker-compose -f nginx-docker-compose.yml up -d`</br>

# #4 let's encrypt certificate, HTTP challenge

![letsencrypt-http-challenge-pic](https://i.imgur.com/yTshxC9.png)

  My understanding of the process, simplified.

  `LE` - Let's Encrypt. A service that gives out free certificates</br>
  `Certificate` - a cryptographic key stored in a file on the server,
   allows encrypted communication and confirms the identity</br>
  `ACME` - a protocol(precisely agreed way of communication) to negotiate certificates
  from LE. It is part of traefik.</br>
  `DNS` - servers on the internet, translate domain names in to ip address</br>

  Traefik uses ACME to ask LE for a certificate for a specific domain, like `whateverblablabla.org`.
  LE answers with some random generated text that traefik puts at a specific place on the server.
  LE then asks DNS internet servers for `whateverblablabla.org` and that points to some IP address.
  LE looks at that IP address through ports 80/443 for the file containing that random text.

  If it's there then this proves that whoever asked for the certificate controls both
  the server and the domain, since it showed control over DNS records.
  Certificate is given and is valid for 3 months, traefik will automatically try to renew
  when less than 30 days is remaining.

  Now how to actually get it done.


- **create an empty acme.json file with 600 permissions**

  This file will store the certificates and all the info about them.

  `touch acme.json && chmod 600 acme.json`

- **add 443 entrypoint and certificate resolver to traefik.yml**</br>

  In entrypoint section new entrypoint is added called websecure, port 443
  
  certificatesResolvers is a configuration section that tells traefik
  how to use acme resolver to get certificates.

    ```
    certificatesResolvers:
      lets-encr:
        acme:
          #caServer: https://acme-staging-v02.api.letsencrypt.org/directory
          storage: acme.json
          email: whatever@gmail.com
          httpChallenge:
            entryPoint: web
    ```

  - the name of the resolver is `lets-encr` and uses acme
  - commented out staging caServer makes LE issue a staging certificate,
    it is an invalid certificate and wont give green lock but has no limitations,
    so it's good for testing. If it's working it will say issued by let's encrypt.
  - Storage tells where to store given certificates - `acme.json`
  - The email is where LE sends notification about certificates expiring
  - httpChallenge is given an entrypoint, so acme does http challenge over port 80

  That is all that is needed for acme

    `traefik.yml`
    ```
    ## STATIC CONFIGURATION
    log:
      level: INFO

    api:
      insecure: true
      dashboard: true

    entryPoints:
      web:
        address: ":80"
      websecure:
        address: ":443"

    providers:
      docker:
        endpoint: "unix:///var/run/docker.sock"
        exposedByDefault: false

    certificatesResolvers:
      lets-encr:
        acme:
          #caServer: https://acme-staging-v02.api.letsencrypt.org/directory
          storage: acme.json
          email: whatever@gmail.com
          httpChallenge:
            entryPoint: web
    ```

- **expose/map port 443 and mount acme.json in traefik-docker-compose.yml** 

  Notice that acme.json is **not** :ro - read only

    `traefik-docker-compose.yml`
    ```
    version: "3.7"

    services:
      traefik:
        image: "traefik:v2.1"
        container_name: "traefik"
        hostname: "traefik"
        env_file:
          - .env
        ports:
          - "80:80"
          - "443:443"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./acme.json:/acme.json"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **add required labels to containers**</br>
compared to just plain http from first chapter,
it is just changing router's entryPoint from `web` to `websecure`
and assigning certificate resolver named `lets-encr` to the existing router

    `whoami-docker-compose.yml`
    ```
    version: "3.7"

    services:
      whoami:
        image: "containous/whoami"
        container_name: "whoami"
        hostname: "whoami"
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.whoami.entrypoints=websecure"
          - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
          - "traefik.http.routers.whoami.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    `nginx-docker-compose.yml`
    ```
    version: "3.7"

    services:
      nginx:
        image: nginx:latest
        container_name: nginx
        hostname: nginx
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.nginx.entrypoints=websecure"
          - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
          - "traefik.http.routers.nginx.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```
- **run the damn containers**</br>
  give it a minute</br>
  containers will now work only over https and have the greenlock</br>

  *extra info:*</br>
  check content of acme.json</br>
  delete acme.json if you want fresh start
 
# #5 let's encrypt certificate DNS challenge on cloudflare

![letsencrypt-dns-challenge-pic](https://i.imgur.com/dkgxFTR.png)

  My understanding of the process, simplified.

  `LE` - Let's Encrypt. A service that gives out free certificates</br>
  `Certificate` - a cryptographic key stored in a file on the server,
   allows encrypted communication and confirms the identity</br>
  `ACME` - a protocol(precisely agreed way of communication) to negotiate certificates
  from LE. It is part of traefik.</br>
  `DNS` - servers on the internet, translate domain names in to ip address</br>

  Traefik uses ACME to ask LE for a certificate for a specific domain, like `whateverblablabla.org`.
  LE answers with some random generated text that traefik puts as a new DNS TXT record.
  LE then checks `whateverblablabla.org` DNS records to see if the text is there.
  
  If it's there then this proves that whoever asked for the certificate controls the domain.
  Certificate is given and is valid for 3 months. Traefik will automatically try to renew
  when less than 30 days is remaining.

  Benefit over httpChallenge is ability to have wild card certificates.
  These are certificates that validate all subdomains `*.whateverblablabla.org`</br>
  Also no ports are needed to be open.

  But traefik needs to be able to make these automated changes to DNS records,
  so there needs to be support for this from whoever manages sites DNS.
  Thats why going with cloudflare.

  Now how to actually get it done.

- **add type A DNS records for all planned subdomains**

  [whoami, nginx, \*] are used example subdomains, each one should have A-record pointing at traefik IP

- **create an empty acme.json file with 600 permissions**

  `touch acme.json && chmod 600 acme.json`

- **add 443 entrypoint and certificate resolver to traefik.yml**</br>

  In entrypoint section new entrypoint is added called websecure, port 443
  
  certificatesResolvers is a configuration section that tells traefik
  how to use acme resolver to get certificates.</br>

    ```
    certificatesResolvers:
    lets-encr:
      acme:
        #caServer: https://acme-staging-v02.api.letsencrypt.org/directory
        email: whatever@gmail.com
        storage: acme.json
        dnsChallenge:
          provider: cloudflare
          resolvers:
            - "1.1.1.1:53"
            - "8.8.8.8:53"
    ```

  - the name of the resolver is `lets-encr` and uses acme
  - commented out staging caServer makes LE issue a staging certificate,
    it is an invalid certificate and wont give green lock but has no limitations,
    if it's working it will say issued by let's encrypt.
  - Storage tells where to store given certificates - `acme.json`
  - The email is where LE sends notification about certificates expiring
  - dnsChallenge is specified with a [provider](https://docs.traefik.io/https/acme/#providers),
    in this case cloudflare. Each provider needs differently named environment variable
    in the .env file, but thats later, here it just needs the name of the provider
  - resolvers are IP of well known DNS servers to use during challenge

  `traefik.yml`
  ```
  ## STATIC CONFIGURATION
  log:
    level: INFO

  api:
    insecure: true
    dashboard: true

  entryPoints:
    web:
      address: ":80"
    websecure:
      address: ":443"

  providers:
    docker:
      endpoint: "unix:///var/run/docker.sock"
      exposedByDefault: false

  certificatesResolvers:
    lets-encr:
      acme:
        #caServer: https://acme-staging-v02.api.letsencrypt.org/directory
        email: whatever@gmail.com
        storage: acme.json
        dnsChallenge:
          provider: cloudflare
          resolvers:
            - "1.1.1.1:53"
            - "8.8.8.8:53"
  ```

- **to the `.env` file add required variables**</br>
We know what variables to add based on the [list of supported providers](https://docs.traefik.io/https/acme/#providers).</br>
For cloudflare variables are 
  - `CF_API_EMAIL` - cloudflare login
  - `CF_API_KEY` - global api key

  `.env`
  ```
  MY_DOMAIN=whateverblablabla.org
  DEFAULT_NETWORK=traefik_net
  CF_API_EMAIL=whateverbastard@gmail.com
  CF_API_KEY=8d08c87dadb0f8f0e63efe84fb115b62e1abc
  ```

- **expose/map port 443 and mount acme.json** in traefik-docker-compose.yml 

  Notice that acme.json is **not** :ro - read only

  `traefik-docker-compose.yml`
  ```
  version: "3.7"

  services:
    traefik:
      image: "traefik:v2.1"
      container_name: "traefik"
      hostname: "traefik"
      env_file:
        - .env
      ports:
        - "80:80"
        - "443:443"
        - "8080:8080"
      volumes:
        - "/var/run/docker.sock:/var/run/docker.sock:ro"
        - "./traefik.yml:/traefik.yml:ro"
        - "./acme.json:/acme.json"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **add required labels to containers**</br>
  compared to just plain http from the first chapter
  - router's entryPoint is switched from `web` to `websecure`
  - certificate resolver named `lets-encr` assigned to the router
  - a label defining main domain that will get the certificate,
    in this it is whoami.whateverblablabla.org, domain name pulled from `.env` file
  
  `whoami-docker-compose.yml`
  ```
  version: "3.7"

  services:
    whoami:
      image: "containous/whoami"
      container_name: "whoami"
      hostname: "whoami"
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.whoami.entrypoints=websecure"
        - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
        - "traefik.http.routers.whoami.tls.certresolver=lets-encr"
        - "traefik.http.routers.whoami.tls.domains[0].main=whoami.$MY_DOMAIN"


  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

  `nginx-docker-compose.yml`
  ```
  version: "3.7"

  services:
    nginx:
      image: nginx:latest
      container_name: nginx
      hostname: nginx
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.nginx.entrypoints=websecure"
        - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
        - "traefik.http.routers.nginx.tls.certresolver=lets-encr"
        - "traefik.http.routers.nginx.tls.domains[0].main=nginx.$MY_DOMAIN"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```
- **run the damn containers**</br>
  `docker-compose -f traefik-docker-compose.yml up -d`</br>
  `docker-compose -f whoami-docker-compose.yml up -d`</br>
  `docker-compose -f nginx-docker-compose.yml up -d`</br>

- **Fuck that, the whole point of DNS challenge is to get wildcards!**</br>
  fair enough</br>
  so for wildcard these labels go in to traefik compose.
  - same `lets-encr` certificateresolver is used as before, the one defined in traefik.yml
  - the wildcard for subdomains(*.whateverblablabla.org) is set as the main domain to get certificate for
  - the naked domain(just plain whateverblablabla.org) is set as sans(Subject Alternative Name)
  
  again, you do need `*.whateverblablabla.org` and `whateverblablabla.org` 
  set in your DNS control panel as A-record pointing to IP of traefik

  `traefik-docker-compose.yml`
  ```
  version: "3.7"

  services:
    traefik:
      image: "traefik:v2.1"
      container_name: "traefik"
      hostname: "traefik"
      env_file:
        - .env
      ports:
        - "80:80"
        - "443:443"
        - "8080:8080"
      volumes:
        - "/var/run/docker.sock:/var/run/docker.sock:ro"
        - "./traefik.yml:/traefik.yml:ro"
        - "./acme.json:/acme.json"
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.traefik.tls.certresolver=lets-encr"
        - "traefik.http.routers.traefik.tls.domains[0].main=*.$MY_DOMAIN"
        - "traefik.http.routers.traefik.tls.domains[0].sans=$MY_DOMAIN"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

  Now if a container wants to be accessible as a subdomain,
  it just needs a regular router that has rule for the url,
  be on 443 port entrypoint, and use the same `lets-encr` certificate resolver

    `whoami-docker-compose.yml`
    ```
    version: "3.7"

    services:
      whoami:
        image: "containous/whoami"
        container_name: "whoami"
        hostname: "whoami"
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.whoami.entrypoints=websecure"
          - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
          - "traefik.http.routers.whoami.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    `nginx-docker-compose.yml`
    ```
    version: "3.7"

    services:
      nginx:
        image: nginx:latest
        container_name: nginx
        hostname: nginx
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.nginx.entrypoints=websecure"
          - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
          - "traefik.http.routers.nginx.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

  Here is apache but this time run on the naked domain `whateverblablabla.org`</br>
    
    `apache-docker-compose.yml`
    ```
    version: "3.7"

    services:
      apache:
        image: httpd:latest
        container_name: apache
        hostname: apache
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.apache.entrypoints=websecure"
          - "traefik.http.routers.apache.rule=Host(`$MY_DOMAIN`)"
          - "traefik.http.routers.apache.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

# #6 redirect http traffic to https

  ![padlocks-pic](https://i.imgur.com/twTDSym.png)

  http stops working with https setup, better to redirect http(80) to https(443).</br>
  Traefik has special type of middleware for this purpose - redirectscheme.

  There are several places where this redirect can be declared,
  in `traefik.yml`, in the dynamic section when `traefik.yml` itself is set as a file provider.</br>
  Or using labels in any running container, this example does it in traefik compose.

- **add new router and a redirect scheme using labels in traefik compose**

  >\- "traefik.enable=true"

  enables traefik on this traefik container,
  not that there is need of the typical routing to a service here,
  but other labels would not work without this

  >\- "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"

  creates new middleware called `redirect-to-https`, type "redirectscheme"
  and assigns it scheme `https`.</br>

  >\- "traefik.http.routers.redirect-https.rule=hostregexp(`{host:.+}`)"

  creates new router called `redirect-https`, with a regex rule that
  catches any and every incoming request

  >\- "traefik.http.routers.redirect-https.entrypoints=web"

  declares on which entrypoint this router listens - web(port 80) </br>

  >\- "traefik.http.routers.redirect-https.middlewares=redirect-to-https"

  assigns the freshly created redirectscheme middleware to this freshly created router.

  So to sum it up, when a request comes at port 80, router that listens at that entrypoint looks at it.
  If it fits the rule, and it does because everything fits that rule, it goes to the next step.
  Ultimately it should get to a service, but if there is middleware declared, that middleware goes first,
  and since middleware is there, and it is some redirect scheme, it never reaches any service,
  it gets redirected using https scheme, which I guess is stating - go for port 443.

  Here is the full traefik compose, with dns challenge labels from previous chapter included:
    
  `traefik-docker-compose.yml`
  ```
  version: "3.7"

  services:
    traefik:
      image: "traefik:v2.1"
      container_name: "traefik"
      hostname: "traefik"
      env_file:
        - .env
      ports:
        - "80:80"
        - "443:443"
        - "8080:8080"
      volumes:
        - "/var/run/docker.sock:/var/run/docker.sock:ro"
        - "./traefik.yml:/traefik.yml:ro"
        - "./acme.json:/acme.json"
      labels:
        - "traefik.enable=true"
        ## DNS CHALLENGE
        - "traefik.http.routers.traefik.tls.certresolver=lets-encr"
        - "traefik.http.routers.traefik.tls.domains[0].main=*.$MY_DOMAIN"
        - "traefik.http.routers.traefik.tls.domains[0].sans=$MY_DOMAIN"
        ## HTTP REDIRECT
        - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
        - "traefik.http.routers.redirect-https.rule=hostregexp(`{host:.+}`)"
        - "traefik.http.routers.redirect-https.entrypoints=web"
        - "traefik.http.routers.redirect-https.middlewares=redirect-to-https"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

- **run the damn containers** and now `http://whoami.whateverblablabla.org` is immediately changed to `https://whoami.whateverblablabla.org`

# stuff to checkout
  - [when file provider is used for managing docker containers](https://github.com/pamendoz/personalDockerCompose)
  - [ansible, docker and traefik](https://thoughtfuldragon.com/a-summary-of-how-i-automated-my-server-with-ansible-docker-and-traefik/)
  - [traefik v2 forums](https://community.containo.us/c/traefik/traefik-v2)
  - ['traefik official site blog](https://containo.us/blog/)
