# Prerequisites
> *Before we can start we need the following things*
 
- A server running linux
- A Gitlab-account
- A domain name to direct to your server
- Basic knowledge of Docker, Dockerfiles, and Docker-compose-files
 
# Secure your server
This [DigitalOcean](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04) post provides a overview of things you can do to secure your new server. 
We will create a non-root user, add that user to the sudoers-group and disable ssh for the root user.

# New user
> *The first thing we going to do is to add a new user*

#### Add a new user:

```console
adduser lovelace
```

#### Provide a password and some other information for your new user:

```console
Adding user 'lovelace' ...
Adding new group 'lovelace' (1002) ...
Adding new user 'lovelace' (1001) with group 'lovelace' ...
Creating home directory '/home/lovelace' ...
Copying files from '/etc/skel' ...
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
Changing the user information for lovelace
Enter the new value, or press ENTER for the default
	Full Name []: Ada Lovelace
	Room Number []:
	Work Phone []:
	Home Phone []:
	Other []:
Is the information correct? [Y/n]
```

#### Add user to the sudoers group:

```console
usermod -aG sudo lovelace
```

#### Disable ssh for root user:
> *You can now ssh into your server with the new user*

To disable ssh on root we need to edit `/etc/ssh/sshd_config` and change `PermitRootLogin yes` or `PermitRootLogin without-password` to `PermitRootLogin no`. Bellow is the code you need to run.

```console
ssh username@ipaddress
sudo nano /etc/ssh/sshd_config
sudo service ssh restart
```

# Install docker, docker-compose and the loadbalancer
> *So now that the boring part is done, we can focus on why we’re actually here*

#### Install [docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-using-the-repository):

```console
sudo apt-get update

sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt-get update

sudo apt-get install docker-ce
```

Check if the installation is successful `sudo docker ps -a`

#### Install [docker-compose](https://docs.docker.com/compose/install/#install-compose):

```console
sudo curl -L https://github.com/docker/compose/releases/download/1.18.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version

docker-compose version 1.18.0, build 8dd22a9
```

#### Manage [docker as a non-root user](https://docs.docker.com/install/linux/linux-postinstall/#manage-docker-as-a-non-root-user):
> *It’s a nightmare to run everything that has to do with docker with `sudo`*

```console
sudo groupadd docker
sudo usermod -aG docker $USER
```

> *Log out and run `docker ps -a`, you’ll notice it now works without sudo*

#### Install [loadbalancer](https://docs.traefik.io/#the-traefik-quickstart-using-docker):
> *Connect our docker containers to port 80/443 and automatically get SSL/HTTPS with Let’s Encrypt*

```console
mkdir /srv/docker
sudo chown -R lovelace:docker /srv/docker/
mkdir /srv/docker/lb
mkdir /srv/docker/lb/data
touch /srv/docker/lb/docker-compose.yml
nano /srv/docker/lb/docker-compose.yml
```

> *Paste the following into `docker-compose.yml`*

```console
version: '3'

services:
  traefik:
    image: traefik:1.7.3 #check for the latest version https://github.com/containous/traefik/releases
    restart: always
	command: --api --docker # Enables the web UI and tells Traefik to listen to docker
    ports:
      - 80:80 #normal traffic
      - 443:443 #ssl traffic
      - 8080:8080 # The Web UI (enabled by --api)
    networks:
      - web
    volumes: #host:container
      - /var/run/docker.sock:/var/run/docker.sock #connect to the docker instance
      - ./data/traefik.toml:/traefik.toml
      - ./data/acme.json:/acme.json #let's encrypt settings
    container_name: traefik

networks:
  web:
    external: true
```

> *A loadbalancer and your application need to be on the same network, otherwise the traffic can’t be routed to the proper container*

#### Create network for the containers:

```console
docker network create web
```

> *We need to add some settings for the **loadbalancer**, and run the following commands*

```console
touch /srv/docker/lb/data/acme.json && chmod 600 /srv/docker/lb/data/acme.json
touch /srv/docker/lb/data/traefik.toml
nano /srv/docker/lb/data/traefik.toml
```

> *Paste the following into `traefik.toml`, don’t forget to change the `[acme]` email field*

```console
debug = false

logLevel = "ERROR"
defaultEntryPoints = ["https","http"]

[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
  [entryPoints.https.tls]

[retry]

[docker]
endpoint = "unix:///var/run/docker.sock"
domain = "my-awesome-app.org"
watch = true
exposedByDefault = false

[acme]
email = "your-email-here@my-awesome-app.org"
storage = "acme.json"
entryPoint = "https"
onHostRule = true
[acme.httpChallenge]
entryPoint = "http"
```

> *Run `docker-compose up -d` inside the `/srv/docker/lb` directory, visit your Traefik Admin ui via `ipaddress:8080`*

# Prepare your server for auto deployment
> *We’re going to create a special deploy user on our server that takes care of the deployment and has minimal rights on our server*

#### Add `deploy` user:

```console
sudo adduser deploy
```

#### Add `deploy` user to the docker group:

```console
sudo usermod -aG docker deploy
```

#### Generate ssh keys on our own machine for `deploy` user:

```console
ssh-keygen -f /Users/lovelace/Desktop/deploy/id_rsa
```

> *We get two files, copy content of `id_rsa.pub`*

#### Create ssh key on server for `deploy` user:

```console
su deploy
Password:
mkdir ~/.ssh
nano ~/.ssh/authorized_keys
```

> *Paste the content of `id_rsa.pub`*

#### docker-compose.yml:
> *The next step is to prepare the `docker-compose.yml` file for our project*

```console
mkdir /srv/docker/project-v1
touch /srv/docker/project-v1/.env
sudo chown deploy:docker /srv/docker/project-v1/.env
nano /srv/docker/project-v1/docker-compose.yml
```

> *Paste the following into `docker-compose.yml`*

```console
version: '3'
services:
    project-v1:
        restart: always
        image: "${CI_REGISTRY}/${CI_PROJECT_NAMESPACE}/${CI_PROJECT_NAME}/${CI_COMMIT_REF_NAME}:${IMAGE_TAG}"
        labels:
        - "traefik.enable=true"
        - "traefik.frontend.rule=Host:v1.my-awesome-app.org"
        - "traefik.port=80"
        networks:
        - web
networks:
    web:
    external:
        name: web
```

> *You need to edit the service name `project-v1` and replace `v1.my-awesome-app.org` with the webaddress you want to use*

# Prepare your gitlab project for auto deployment
> *We need to place a `.gitlab-ci.yml` file into our gitlab project*

#### DNS records and domain setup:
> *Create `A` record for `v1.my-awesome-app.org`* 

#### Deploying with gitlab:
> *The deployment flow looks as following*

- We have a project on gitlab, we can push and pull
- When we push our commits to our project, gitlab starts to look for the `.gitlab-ci.yml` file inside the repository, and will run it if the branches match (`master`)
- In the `.gitlab-ci.yml` file create two steps: build and deploy step
- In the build step we use the included `Dockerfile` to create a docker image, and push that image to the gitlab image registry
- In the deploy step, log in to our server via SSH (we created the deploy user for this), and pull our image from the gitlab image registry
- Then we run `docker-compose up -d` on our created `docker-compose` file to run the image we created

#### Create `.gitlab-ci.yml` file in the root of the project:

```console
image: docker:git

services:
    - docker:dind

variables:
    IMAGE_TAG: $CI_COMMIT_SHA

stages:
    - build
    - deploy

build:
    stage: build
    script:
        - docker build --build-arg NODE_ENV=prod -t $CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME/$CI_COMMIT_REF_NAME:$IMAGE_TAG .
        - docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN $CI_REGISTRY
        - docker push $CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME/$CI_COMMIT_REF_NAME:$IMAGE_TAG
    only:
        - master

deploy:
    stage: deploy
    before_script:
        - mkdir -p ~/.ssh
        - echo "$PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa
        - chmod 600 ~/.ssh/id_rsa
        - eval "$(ssh-agent -s)"
        - ssh-add ~/.ssh/id_rsa
        - ssh-keyscan -H $DEPLOYMENT_SERVER >> ~/.ssh/known_hosts
    script:
        - echo -e "IMAGE_TAG=${IMAGE_TAG}\n CI_REGISTRY=${CI_REGISTRY}\n CI_PROJECT_NAMESPACE=${CI_PROJECT_NAMESPACE}\n CI_PROJECT_NAME=${CI_PROJECT_NAME}\n CI_COMMIT_REF_NAME=${CI_COMMIT_REF_NAME}" > .env
        - scp ./.env $DEPLOYMENT_USER@$DEPLOYMENT_SERVER:$$DEPLOYMENT_LOCATION/.env
        - ssh $DEPLOYMENT_USER@$DEPLOYMENT_SERVER "docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN $CI_REGISTRY"
        - ssh $DEPLOYMENT_USER@$DEPLOYMENT_SERVER "cd $$DEPLOYMENT_LOCATION && docker-compose stop"
        - ssh $DEPLOYMENT_USER@$DEPLOYMENT_SERVER "cd $$DEPLOYMENT_LOCATION && docker-compose up -d"
    only:
        - master
```

> *A `.env` file is generated on our server and some variables are pushed into the file. Those variables are read by our `docker-compose` command when `docker-compose up -d` is ran*

#### Create `Dockerfile` in the root of the project

```console
FROM exiasr/alpine-yarn-nginx:8.9.4
WORKDIR /usr/share/nginx/www
ADD ./ /usr/share/nginx/www
RUN yarn install
RUN yarn global add gulp
RUN gulp sass
RUN mv nginx/default.conf /etc/nginx/conf.d
EXPOSE 80
```

> *You can’t just copy paste this `Dockerfile` for your project, but I wanted to share how simple a `Dockerfile` is*
> *You don’t have to expose port 443 for HTTPS traffic, the `loadbalancer` takes care of SSL termination, and routes the traffic to port 80 of the container*

#### Environment variables for gitlab project
> *In your gitlab project go to `Settings > CI / CD` and expand the `Variables` tab, and add*

```console
DEPLOYMENT_SERVER
```
> *Your server `ipaddress`*

```console
PRIVATEKEY
```
> *The contents of `idrsa` file we created in the previous step*

```console
DEPLOYMENT_USER
```
> *The name of our server user we want to deploy, so in our case `deploy`*

```console
DEPLOYMENT_LOCATION
```
> *The directory in which we created our `docker-compose.yml` file in the previous step. So in our case `/srv/docker/project-v1`*

> *All we need to do, is push our files in the `master` branch to gitlab*
> *If every is setup correctly, gitlab will start processing your `gitlab-ci.yml` file*
