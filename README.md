# Deploy to a VPS via Docker Stack (Nov 2024)

**resources**: 
- https://www.youtube.com/watch?v=fuZoxuBiL9o
- https://github.com/dreamsofcode-io/zenstats.git

---

# Why not use Docker Compose?

When it comes to redeploying my applications, I used to manually SSH into my VPS, and run the `docker compose up` command.  
This method works but is not exactly the best developer experience, especially when compared to using platforms such as Vercel, or Netlify.  

Not only is this a bad developer experience, but because of the way that Docker Compose works, it also comes with some undesired side effects.  
The most major of them is that, when you redeploy your application stack using the `docker compose up` command, it can cause downtime.  

Because when Docker Compose redeploys, it begins by shutting down your already running services, before attempting to deploy the upgraded ones.  
So if there's a problem with your new application code or configuration, then these services won't be able to start back up, which will cause your app to have an outage.  

Additionally, by needing to ssh in and copy over the compose.yaml file in order to redeploy, I find this prevents me from being able to ship fast.  
Instead of this manual process, I'd rather have a solution that allows me to easily ship remotely through either my local machine or via **CI/CD**.  

However, rather than using one of the platforms that provide such a service, I did some research and found a solution.  
A solution that not only solves my issue with Docker Compose, but also allows me to use the same Docker compose files I already have set up.  

That solution is **Docker Stack**, which has quickly become my favorite way to deploy my apps to a VPS.  

# How does Docker Stack work?

Docker Stack allows you to deploy your Docker compose files on a node that has **Docker Swarm** mode enabled.  
Which is much better suited for production services compared to Docker Compose.  

It has support for many features that are important when it comes to running a production-grade application, such as:
- Blue/Green Deployments
- Rolling Releases
- Secure Secrets
- Service Rollbacks
- Clustering

Not only this, but when combined with **Docker Context**, I'm able to remotely manage and deploy multiple VPS instances from my own workstation.  
All in a **secure** and **fast** way.  

Let's say I want to make a change to an existing Web App Service Stack by adding in a **Valkey** instance.  
This Web app is running on a VPS and is deployed via Docker Stack.  

All I need to do is edit my docker-compose.yaml file and add in the following lines:
```yaml
valkey:
  image: "valkey/valkey:8"
  volumes:
    - valkey: /data
```
- The first line defines a new service named "valkey"
- The second one specifies that this service should use the Valkey image version 8 from the Docker Hub repository
- The third and fourth lines create a named volume called "valkey" and mounts it to the /data directory inside the container.

Then, in order to deploy this, I can run the `docker stack deploy` command, passing in the docker-compose file that I want to use, and the name of my stack.
```bash
docker stack deploy -c ./docker-compose.yaml <stackName>
```

The `docker stack deploy -c` command is used to deploy a stack of services to a Docker swarm.  
Here's how it works:
- The `-c` flag is short for `--compose-file` and is used to specify the path to a Compose file.
- The command syntax is `docker stack deploy -c <path-to-compose-file> <stack-name>`

After deployment, you can check the status of your services using:
```bash
docker service ls
```

---

## Side note about Valkey

Valkey is an **open-source**, high-performance, in-memory key-value datastore that serves as a **successor to Redis**.   
It functions as a **distributed database**, cache, and message broker with optional durability.  
Valkey was created in **2024** as a fork of Redis 7.2.4 in **response to Redis Ltd.'s switch to proprietary licensing**.  

---

# Other benefits from using Docker Stack

Besides being able to deploy and redeploy my application services on a remote VPS from my local machine, I'm also able to manage  
and monitor my application from my local machine as well, such as:
- being able to view the different services logs
- adding secrets securely

I've also managed to set up Docker Stack to work with my CI/CD pipeline using **GitHub actions**.  
Meaning whenever I push a code change to the main branch of my repo, it'll automatically deploy my entire stack.  

# How difficult is it to get set up?

It's actually pretty simple, and we'll see how to deploy a simple Web application on a VPS.  
- Open VS Code
- Run `git clone https://github.com/dreamsofcode-io/zenstats.git`

If we open the `compose.base.yaml` file, we'll see that we have 2 distinct services:
- the web application
- a postgres database

In addition to having the Dockerfile and docker-compose files already defined, this application has some GitHub actions setup as well.  
These actions can be found in the `pipeline.yaml` file that is located in the .github/workflows folder.  

The `pipeline.yaml` file performs 2 different automations:
- running the automated tests, and if they pass, move on to the second step
- building and pushing a new Docker image of the application to the **GitHub Container Registry (GHCR)**

The interesting thing to note here is that the Docker image is **tagged** with both `latest` and the same `commit hash` that is found at the repo at the time the image is built:  
![image](https://github.com/user-attachments/assets/db506650-8257-4677-92d2-4b9c65fb0d8b)  

Which makes it very easy to correlate the docker image with the code that is was built from.  

---

# Deploy our app to a VPS instance

## VPS setup

- purchase a VPS hosting plan on a platform such as Hostinger - https://www.hostinger.fr/vps#pricing (10% off with 'DREAMSOFCODE')
- select Ubuntu 24.04
- create a pwd for the root user
- give your VPS a hostname
- generate and add an SSH public key
- if you have a spare domain name, add a `DNS A record` to your VPS (by adding the public IP address in the 'Points to' field)
- if not, then you can buy one from hostinger

Once your VPS is set up, ssh into it as your root user.  
Then, install the Docker engine, and follow the post-install steps: 
- https://docs.docker.com/engine/install/ubuntu/
- https://docs.docker.com/engine/install/linux-postinstall/

Note that we don't have to install the docker-buildx and docker-compose plugins, we won't need them.  

Once Docker is installed, we can check that it's working by running the `docker ps` command.  

---

### Setting up a production-ready VPS 

If you're going to use this VPS as a production machine, then I would recommend going through the steps described in the following video:  
https://www.youtube.com/watch?v=F-9KWQByeU0  

Alternatively, you can also find a step-by-step guide over here:  
https://github.com/dreamsofcode-io/zenstats/blob/main/docs/vps-setup.md

---

## Deploying our app to the VPS

With our VPS set up, we can exit out of SSH.  
Now, we need to change our Docker host to be our VPS instance, which can be done in a couple of different ways.  

- The first method is to set the DOCKER_HOST environment variable so it points to our VPS.  
For example, we could run something like `export DOCKER_HOST=ssh://root@zenstats.com`  

- The other (preferred) way to change the Docker host is by using the `docker context` command, which allows you to store and manage multiple Docker hosts.  
This second method makes it easy to switch between your Docker hosts when you have multiple VPSs.  

### Using docker context

To create a new Docker context, we'll use the `docker context create` command, passing in the name we want to give it.  
For example: `docker context create zenstats-app`  

Then, we can define the Docker endpoints by using the `--docker` flag:  
`docker context create zenstats-app --docker "host=ssh://root@zenstat.com"`  

The general syntax is:  
`docker context create <contextName> --docker "host=ssh://<userName>@<VPS_hostname_or_IP_address>`  

If you dont' have a domain name set up, you can just use the VPS's IP address instead.  

Once we've created our Docker context, we can make use of it via the `docker context use` command.  
Passing in the name of the context we've just created: `docker context use zenstats-app`

Now, whenever we perform a docker command, instead of taking place on our local machine, it will run on the docker instance of our VPS.  

With our context defined, we're now ready to set up our node to use Docker Stack.  

## Setting up our node

We first need to enable **Docker Swarm** mode on our VPS via the `docker swarm init` command.  
Upon running this command, you should then receive a token that will allow you to connect other VPS instances to this machine,  
in order to form a **Docker swarm cluster**.  

With swarm mode enabled, we can now deploy our application using the `docker stack deploy` command.  
Passing in the path to our Docker compose file and the stack name: `docker stack deploy -c ./compose.yaml <stackName>`  

When executing the above command, we should see some output letting us know that the different services of our stack are being deployed.  
Once completed, we can open up a browser window and head over to our domain name or to the IP address of our VPS, to check that our app is now deployed.  

## Private images

This remote deployment also works when it comes to private images.  

If we change our compose.yaml file to make use of a private image, when we'll use the `docker stack deploy` command, we can see that it's deployed successfully.  
However, we'll get a warning message in our terminal:  
![image](https://github.com/user-attachments/assets/1b43a6bc-a941-4132-af69-f08b52c45bcb)  

This message is only really an issue if you're running a Docker swarm cluster.  

To resolve this, we just need to use the `--with-registry-auth` flag when running the `docker stack deploy` command.  

---

# Secrets Management

## Docker Compose limitations

We could actually make remote deployments combining the use of Docker context (or DOCKER_HOST) with Docker compose.  
However, you would face an issue that would cause your deployments to fail when running the `docker compose -f compose.yaml up` command.  
That's because of how docker compose manages secrets.  
![image](https://github.com/user-attachments/assets/6750dae8-c99d-4ffa-945e-f1b1b94a3ff5)

Docker Compose implements secrets by binding local files to the container.  
When using secrets in a Compose file, it expects the secret files to exist on the host machine where the docker compose command is executed.   
This works well for local development but becomes problematic in remote deployment scenarios.  

When deploying to a remote host, the local file paths specified in the Compose file may not exist on the remote system.  
For example:
- A secret defined as /run/secrets/secret_key.txt on a local Linux machine
- When deployed from a Windows system to a remote Linux host, Docker Compose might look for C:/run/secrets/secret_key.txt
This mismatch in file paths leads to errors such as "invalid mount config" or "bind source path does not exist".

Additionally, there's no easy way to manage the files on the remote machine without resorting to SSH.  

For these reasons, it makes sense to prefer Docker Swarm over Docker Compose, as they have more robust secrets management.  

## Docker Swarm is better

With Docker Swarm, secrets are managed via the `docker secret` command.  
This command allows to create a secret inside of our Docker host in a way that's both encrypted at rest, and encrypted during transit.  

### Creating a secret

`docker secret create db-password ./password.txt` or `docker secret create db-password -`
- The first argument of this command is the name of the secret
- The second argument is the secret's actual value 

Since Docker Secret is very secure, we can't just enter in the secret's value, we need to either:
- load this value in from a file, and then delete this file
- or load this in through the standard input, using just a dash at the end of the command

To add a secret via STDIN on a MacOS or Linux system, you can use something such as the `printf` command: 
```bash
 printf 'mySecretPassword' | docker secret create db-password -
```
Note the space at the beginning of the above command. It's explained in the following note.

---

### IMPORTANT NOTE

Running a command with a leading space is a simple technique to prevent it from being saved in the shell history.  
This behavior is controlled by the HISTCONTROL environment variable, which is typically set to "ignoreboth" or "ignorespace" by default in many Linux distributions.  

When HISTCONTROL includes "ignorespace", any command that begins with a space character will not be recorded in the shell's history file.   
This feature allows users to execute sensitive commands or those containing confidential information without leaving a trace in the command history.

---

To display our secrets name and ID: `docker secret ls`  
We can also run `docker secret inspect <secret_ID>` to get information about a specific secret.  

But there's no way for us to retrieve a secret's value from Docker, which is why we should **store them securely somewhere else** (in a pwd manager).  

### Setting up our newly created secret

Once we've securely created and stored our secret, we can then use it similar to how we would with Docker Compose.  
However, rather than setting the secret as a file, we define it as external.  

With Docker Compose
```yaml
secrets:
  db-password:
    file: db-password.txt
```

With Docker Swarm
```yaml
secrets:
  db-password:
    external: true
```

Then, we must do two things:
- add the secret to the services that need access to it
- set the secret in the associated environment variables

In our example, the compose.yaml will look something like that:
```yaml
services:
  web:
    image: ghcr.io/dreamsofcode-io/zenstats:<image_tag>
    secrets:
      - db-password
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_PWD_FILE=/run/secrets/db-password
      - POSTGRES_USER=postgres
      - POSTGRES_DB=app
      - POSTGRES_PORT=5432
      - POSTGRES_SSLMODE=disable
    ports:
      - "80:8080"
    deploy:
      update_config:
        order: start-first
    depends_on:
      - db

  db:
    image: postgres
    user: postgres
    volumes:
      - db-data: /var/lib/postgresql/data
    secrets:
      - db-password
    environment:
      - POSTGRES_DB=app
      - POSTGRES_PWD_FILE=/run/secrets/db-password
```

### Redeploying our stack

Now, if we run `docker stack deploy -c compose.yaml zenfulstats`, our database will be working fine.  
However, if we run `docker ps -a`, we'll notice that the new version of our web application is failing.  

This is because we're actually using an old image version that doesnt have the environment variable set up with the db-password file.  
So it's unable to connect to the database and exiting early.  

And if you open up a web browser and head over to our application, you'll see it's still running...  
This is because Docker Stack has support for **rolling releases**, which means it's still running the old configuration from before the secret's creation.  

The `docker stack deploy` command basically acts as a very simple blue/green deployment.  
This behavior is configured using the following 3 lines:
```yaml
deploy:
  update_config:
    order: start-first
```

Let's explain how these 3 lines are working:  
- The `update_config` section specifies how the service should be updated when changes are applied.
- The `order: start-first` option determines the sequence of operations during an update.
- With `start-first`, Docker will:
  - Start the new task (container) with the updated configuration
  - Briefly allow the new and old tasks to run simultaneously
  - Stop the old task once the new one is running

This approach aims to minimize downtime during updates by ensuring the new version is up and running before stopping the old one.   
It's particularly useful for achieving **zero-downtime deployments**, especially when you have only one replica of the service.  
However, it's **important** to note that using `start-first` may temporarily **consume more resources** during the update process, as both the old and new versions will be running concurrently for a short period.  

To fix this "running the old web app version" issue, we need to: 
- go to the GitHub repo: https://github.com/dreamsofcode-io/zenstats/pkgs/container/zenstats
- and copy the **latest image tag** that is the alphanumeric string at the end of the `docker pull` command
  - currently, this tag is: 025f001c2b1a80d7a577c73185a1ed9efe8e575e
- then paste this tag at the end of the following line in the compose.yaml file
```yaml
web:
  image: ghcr.io/dreamsofcode-io/zenstats:<latest_image_tag>
```

Updating the image tag allows our web app to connect to the database via the POSTGRES_PWD_FILE environment variable.  
Now, we can redeploy our services via `docker stack deploy -c compose.yaml zenfulstats`  

And now the `docker ps` command will show that both our services (web app and postgres database) are running successfully.  

---

## Load balancing

Another feature that Docker Compose doesn't support is load balancing.  
Fortunately, both Docker Stack and Docker Swarm natively support that feature.  

To show this in action, we can scale up the web app to 3 replicas.  
For that, we need to run `docker service scale zenfulstats_web=3`  
The syntax is `docker service scale <stackName_serviceName>=<numberOfReplicas>`  

Next, we can tail the logs via `docker service logs zenfulstats_web -f`  
Which shows us that the built-in load balancer is distributing the requests between our 3 replicas (zenfulstats_web.1, zenfulstats_web.2 and zenfulstats_web.3)  
The default load balancing method is "**Round Robin**".  

Whilst we could scale up replicas with Docker Compose, it's only able to bind a single instance on a given port.  
Which means, in order to effectively use load balancing, we would need to use an **external proxy** such as **Traefik** or **Nginx**.  

- https://github.com/traefik/traefik - reverse proxy + load balancer

In addition to handling load balancing, **Traefik** also provides the ability for **HTTPS** (automatically generates certificates), and does a great job at forwarding client IPs.  

Example implementation of Traefik in a compose.yaml file:
```yaml
services:
  traefik:
    image: traefik:v3.1
    command:
      - "--providers.docker"
      - "--providers.docker.exposed by default=false"
      - "--entryPoints.websecure.address=:443"
      - "--certificates resolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email-elliott@zenful.site"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
    ports:
      - mode: host
        protocol: tcp
        published: 80
        target: 80
      - mode: host
        protocol: tcp
        published: 443
        target: 443
    volumes:
      - letsencrypt: /letsencrypt
      - /var/run/docker.sock: /var/run/docker.sock
  web:
    image: ghcr.io/dreamsofcode-io/zenstats:<latestImageTag>
    labels:
      - "traefik.enable=true"
      - "traefik.http.services. whoami.loadbalancer.port=80"
      - "traefik.http.routers.web.rule=Host(`zenful.site`)"
      - "traefik.http.routers.web.entrypoints=websecure"
      - "traefik.http.routers.web.tls.certresolver=myresolver"
```

---

## Docker Swarm issue

Because of the way that Docker Compose handles load balancing (only able to bind a single instance on a given port), it prevents the original client IP from being forwarded to your services.  

There is however an unofficial solution called **docker-ingress-routing-daemon (DIRD)**.  
https://github.com/newsnowlabs/docker-ingress-routing-daemon  

---

## Services Rollback with Docker Swarm

syntax: `docker service rollback <stackName_serviceName>`  
example: `docker service rollback zenfulstats_web`

---

## Automated Deployments - Docker Stack + GitHub Actions

### The .yaml file

- Go to https://github.com/dreamsofcode-io/zenstats/blob/main/.github/workflows/pipeline.yaml
- We have 3 jobs here: "run-tests", "build-and-push-image", and "deploy"
- The third job is where the actual `docker stack` deployment takes place

### The "deploy" job

In this "deploy" job, you'll notice that we're defining the build-and-push step in the "needs" field.  
Which means it's required pass in order for this job to run.  
```yaml
deploy:
  runs-on: ubuntu-latest
  needs:
    - build-and-push-image
  steps:
  - name: Checkout code
    uses: actions/checkout@v2

  - name: create env file
    run: |
      echo "GIT_COMMIT_HASH=${{ github.sha }}" >> ./envfile

  - name: Docker Stack Deploy
    uses: cssnr/stack-deploy-action@v1
    with:
      name: zenfulstats
      file: docker-stack.yaml
      host: zenful.site
      user: deploy
      ssh_key: ${{ secrets.DEPLOY_SSH_PRIVATE_KEY }}
      env_file: ./envfile
```

There are 2 steps inside of this job:
- Checkout the code at the current commit, which is pretty standard in GitHub actions
- a third-party action to deploy the Docker stack

You can find the documentation for these actions on the GitHub Actions Marketplace:  
https://github.com/marketplace/actions/docker-stack-deploy  

Notice the file property value, which is set to `docker-stack.yaml`.  
This filename is commonly used to differentiate a docker-compose configuration from a docker-stack configuration.  

The **user** property is set to "deploy".  
The **ssh_key** property is set to a GitHub secret.   

**IMPORTANT**: In order for this to work, we need to set both of these up (user and ssh_key) inside of our VPS.  

---

### Setting up the user and SSH public key on our VPS

It's a good practice to create a new user for our deployments because it allows us to limit the permissions.  
It's a good security measure to limit the amount of damage if the SSH private key happens to be compromised.

- ssh into your VPS
- once logged in as the root user, create a new user called "deploy": `adduser deploy`
- add this user to the 'docker' group: `usermod -aG docker deploy`
This last cmd will allow the user to perform any docker cmd without using `sudo`.

- to switch from root to the new user: `su - deploy`, ssing the hyphen (-) creates a new environment for the user you're switching to
- exit the VPS

Next, we need to create an SSH key pair for this new user: `ssh-keygen -t ed25519 -C "deploy@zenful.site"`  
The creation of this key pair must be done on your local machine, not on the VPS.  

After that, let's add the public key to our user's authorized keys:
- once you've create the key pair on your local machine, ssh back into your VPS
- switch from root to the "deploy" user
- create a `.ssh` folder inside this user's home directory via `mkdir .ssh`
- copy the ssh public key from your local machine to the VPS `/home/deploy/.ssh` folder
  - for that, copy the public key to your local machine's clipboard
  - then run this cmd on your VPS: `echo '<paste_here_the_public_key>' > /home/deploy/.ssh/authorized_keys`
  - you can check it was correctly copied with `cat /home/deploy/.ssh/authorized_keys`
 
With that, we should now be able to ssh into our VPS as the "deploy" user: `ssh deploy@zenful.site -i <path_to_private_key>`  
Of course the zenful.site domaine name can be replaced with the VPS IP address.  
The -i option in the ssh command is to specify the private key for authentication (which succeeds if the private key matches the public one).  

Next, we need to restrict what commands this user can actually perform. To do so:
- while being logged in to the VPS as the "deploy" user, run `vim ~/.ssh/authorized_keys`
- press `i` to enter insert mode
- add the following text before the actual ssh key: `command="docker system dial-stdio"`, with a space between this txt and the key
- save and quit by pressing the Escape key, typing `:wq`, and pressing Enter

This will restrict the user to only being able to perform the `docker stack deploy` command when using ssh with this key.  
We can test that this is the case by attempting to ssh in as our "deploy" user: `ssh deploy@zenful.site -i <path_to_private_key>`, which should be rejected.  

From now on, only the `docker stack deploy -c docker-stack.yaml zenfulstats` command should work.

---

### Adding the private key to our GitHub repository

- navigate over to the GitHub repo
- go to Settings > Secrets and variables > Actions
- click the 'new repository secret' button
- name it 'DEPLOY_SSH_PRIVATE_KEY', as in our 'pipeline.yaml' file
- for the secret value, paste in the contents of the private key

### Push the code to GitHub in order to redeploy your stack

Now, if we go ahead and commit+push our code to GitHub, when we navigate over to that repo, we should see the deployment automation in action:  
![image](https://github.com/user-attachments/assets/dbf5b86b-8d1c-4969-af42-a3d086056be5)  

With that, our automated deployment setup is complete.

---

## Specifying which image to use for each deployment

If your remember (line 109), for each image that we build in this pipeline, we're tagging it with both the 'latest' tag, and the git commit hash:  
![image](https://github.com/user-attachments/assets/3504ce44-2f08-45c7-8d18-331361058303)  
This is the Git commit hash of the code that this image is built from.  

In order to make our deployments more deterministic, we want to make sure to use the same image tag.  
To achieve this, we can use an environment variable, replacing the reference to our Docker image tag in our docker-stack.yaml file with the following syntax:  
```yaml
web:
  image: ghcr.io/dreamsofcode-io/zenstats:${GIT_COMMIT_HASH}
```
Which will cause the tag to be loaded from an environment variable named `GIT_COMMIT_HASH`.  

If I go ahead and set this environment variable to the last image's hash:  
![image](https://github.com/user-attachments/assets/900ecfa6-903f-4cae-be72-f6f2de369bd3)

And then run the `docker stack deploy` command:  
![image](https://github.com/user-attachments/assets/2e93af8b-2ba7-4b09-b5ba-a8701a9d96cc)
We can see our stack is being deployed with the image hash we specified.  

In addition to this, we can also specify a default value for this environment variable in the case where it's not set:  
```yaml
web:
  image: ghcr.io/dreamsofcode-io/zenstats:${GIT_COMMIT_HASH:-latest}
```

Which, in this case, would set the default value to "latest":  
![image](https://github.com/user-attachments/assets/1d03eb1f-0c56-4906-a311-c63aa95ba573)  

## Setting the environment variable in our pipeline

The last thing to do now is to set this environment variable in our deployment pipeline.  

This is done by setting the `env_file` option in the stack deploy step:  
![image](https://github.com/user-attachments/assets/4404a026-8b62-416f-904d-7b5fa7beae0b)  

Then, we can go ahead and create the `envfile` by adding a step before this one:  
![image](https://github.com/user-attachments/assets/eba99b00-211f-4134-8744-08ce4950de74)  

Here we're creating a file with a GIT_COMMIT_HASH environment variable set to the current github.sha value.  

Now, if we commit and push this code, we should see (in our GitHub repo) the pipeline working without issue:  
![image](https://github.com/user-attachments/assets/66a46a52-ac7f-4e98-ab2d-872d6d722c1e)  

And we can test that it's using the correct image by adding a version info to the webpage title, for example.

