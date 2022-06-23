**[Prerequisites](#prerequisites)** |
**[Authenticator setup](#authenticator-setup)** |
**[Build the JupyterHub Docker image](#build-the-jupyterhub-docker-image)** |
**[Spawner: Prepare the Jupyter Notebook Image](#spawner-prepare-the-jupyter-notebook-image)** |
**[Run JupyterHub](#run-jupyterhub)** |
**[Behind the scenes](#behind-the-scenes)** |
**[FAQ](#faq)**

Build based on [jupyterhub/jupyterhub-deploy-docker](https://github.com/jupyterhub/jupyterhub-deploy-docker).

# jupyterhub-deploy-docker installation steps taken
## Prerequisites
### ☑️ Docker
### ☑️ HTTPS and SSL/TLS certificate
1. domain name for JupyterHub: `imars.usf.edu`.
2. SSL certificates are in `/root/certs_imars_usf_edu/`
1. put jupyterhub.crt & jupyterhub.key in `/secrets/` dir:
    ```bash
    (base) root@manglilloo:~/jupyterhub-deploy-docker# mkdir -p secrets
    (base) root@manglilloo:~/jupyterhub-deploy-docker# cp ~/certs_imars_usf_edu/imars_usf_edu_cert.cer secrets/jupyterhub.crt
    (base) root@manglilloo:~/jupyterhub-deploy-docker# cp ~/certs_imars_usf_edu/imars_usf_edu.key secrets/jupyterhub.key
    ```

## ☑️ GH OAuth setup
create & register a GitHub OAuth app
* callback URL: `https://imars.usf.edu/hub/oauth_callback`.
- added the following lines in the `oauth.env` file:

  `oauth.env` file
  ```
  GITHUB_CLIENT_ID=<github_client_id>
  GITHUB_CLIENT_SECRET=<github_client_secret>
  OAUTH_CALLBACK_URL=https://<myhost.mydomain>/hub/oauth_callback
  ```

## ☑️ Build the JupyterHub Docker image

1. Created a `userlist` file of GitHub usernames w/ >0 `admin` users.
   ```
   7yl4r admin
   dotis
   ```

1. build JupyterHub Docker image:
   ```bash
   make build
   ```

## ☑️ Prep the default Jupyter Notebook Image

```bash
make notebook_image
```

## ☑️ Run JupyterHub
```bash
docker-compose up -d
```

☑️ able to access the JupyterHub console at

https://imars.usf.edu/hub



---


## Behind the scenes

`make build` does a few things behind the scenes, to set up the environment for JupyterHub:

### Create a JupyterHub Data Volume

Create a Docker volume to persist JupyterHub data.   This volume will reside on the host machine.  Using a volume allows user lists, cookies, etc., to persist across JupyterHub container restarts.

```bash
docker volume create --name jupyterhub-data
```

### Create a Docker Network

Create a Docker network for inter-container communication.  The benefits of using a Docker network are:

* container isolation - only the containers on the network can access one another
* name resolution - Docker daemon runs an embedded DNS server to provide automatic service discovery for containers connected to user-defined networks.  This allows us to access containers on the same network by name.

Here we create a Docker network named `jupyterhub-network`.  Later, we will configure the JupyterHub and single-user Jupyter Notebook containers to run attached to this network.

```bash
docker network create jupyterhub-network
```

---

## FAQ

### How can I view the logs for JupyterHub or users' Notebook servers?

Use `docker logs <container>`.  For example, to view the logs of the `jupyterhub` container

```bash
docker logs jupyterhub
```

### How do I specify the Notebook server image to spawn for users?

In this deployment, JupyterHub uses DockerSpawner to spawn single-user
Notebook servers. You set the desired Notebook server image in a
`DOCKER_NOTEBOOK_IMAGE` environment variable.

JupyterHub reads the Notebook image name from `jupyterhub_config.py`, which
reads the Notebook image name from the `DOCKER_NOTEBOOK_IMAGE` environment
variable:

```python
# DockerSpawner setting in jupyterhub_config.py
c.DockerSpawner.container_image = os.environ['DOCKER_NOTEBOOK_IMAGE']
```

By default, the`DOCKER_NOTEBOOK_IMAGE` environment variable is set in the
`.env` file.

**file**
```
# Setting in the .env file
DOCKER_NOTEBOOK_IMAGE=jupyter/scipy-notebook:2d878db5cbff
```

To use a different notebook server image, you can either change the desired
container image value in the `.env` file, or you can override it
by setting the `DOCKER_NOTEBOOK_IMAGE` variable to a different Notebook
image in the environment where you launch JupyterHub. For example, the
following setting would be used to spawn single-user `pyspark` notebook servers:

```bash
export DOCKER_NOTEBOOK_IMAGE=jupyterhub/pyspark-notebook:2d878db5cbff

docker-compose up -d
```

### If I change the name of the Notebook server image to spawn, do I need to restart JupyterHub?

Yes. JupyterHub reads its configuration which includes the container image
name for DockerSpawner. JupyterHub uses this configuration to determine the
Notebook server image to spawn during startup.

If you change DockerSpawner's name of the Docker image to spawn, you will
need to restart the JupyterHub container for changes to occur.

In this reference deployment, cookies are persisted to a Docker volume on the
Hub's host. Restarting JupyterHub might cause a temporary blip in user
service as the JupyterHub container restarts. Users will not have to login
again to their individual notebook servers. However, users may need to
refresh their browser to re-establish connections to the running Notebook
kernels.

### How can I backup a user's notebook directory?

There are multiple ways to [backup and restore](https://docs.docker.com/engine/userguide/containers/dockervolumes/#backup-restore-or-migrate-data-volumes) data in Docker containers.  

Suppose you have the following running containers:

```bash
    docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Names}}"

    CONTAINER ID        IMAGE                    NAMES
    bc02dd6bb91b        jupyter/minimal-notebook jupyter-jtyberg
    7b48a0b33389        jupyterhub               jupyterhub
```

In this deployment, the user's notebook directories (`/home/jovyan/work`) are backed by Docker volumes.

```bash
    docker inspect -f '{{ .Mounts }}' jupyter-jtyberg

    [{jtyberg /var/lib/docker/volumes/jtyberg/_data /home/jovyan/work local rw true rprivate}]
```

We can backup the user's notebook directory by running a separate container that mounts the user's volume and creates a tarball of the directory.  

```bash
docker run --rm \
  -u root \
  -v /tmp:/backups \
  -v jtyberg:/notebooks \
  jupyter/minimal-notebook \
  tar cvf /backups/jtyberg-backup.tar /notebooks
```

The above command creates a tarball in the `/tmp` directory on the host.
