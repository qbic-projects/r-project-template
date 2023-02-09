# R-project-template

R project template

## Running Rstudio Server with Docker on deNBI cloud

Start by creating a repository within `qbic-projects` from this template. For this, use the button: `Use this template`.

### deNBI instance setup

Launch a deNBI instance with the following characteristics:

- Details: name the instance
- Source: Image and select CentOS image (e.g. CentOS 7.9)
- Flavour: Choose flavour (de.NBI medium should be fine for most analyses)
- Networks: Select deNBI Tübingen external network
- Security Groups: add `external_access` security group.
- Key Pair: add your SSH key or generate a new one.
- (Leave default in the rest of the fields)

You will also need to create and attach a volume. There are step by step instructions for creating instances and attaching volumes on the [qbic pipeline docs page](https://pipeline-docs.readthedocs.io/en/latest/markdown/clusters/denbi_cloud.html).

You should do your work and computations within the mounted volume path. You can also clone your newly created repository there.

Log into the instance using the IP address as host name (user name is `centos`).

> If you expect to need more than 20GB of space, mount a cynder volume on the instance.

### Install the required software

To run the Rstudio server via docker we will require [Docker](https://www.docker.com/), [docker-compose](https://github.com/docker/compose), [conda](https://docs.conda.io/en/latest/miniconda.html) or [mamba](https://github.com/conda-forge/miniforge#mambaforge).

This can be installed via ansible. Install ansible and other necessary software via yum:

```bash
sudo yum install epel-release ansible vim git wget -y
sudo yum-config-manager --add-repo https://cli.github.com/packages/rpm/gh-cli.repo
sudo yum install gh
```

Update dependencies:

```bash
sudo yum update
```

Install necessary ansible roles (for docker, docker-compose and miniconda):

```bash
ansible-galaxy install geerlingguy.docker
ansible-galaxy install andrewrothstein.miniconda
```

Then run the `install_docker_conda.yml` ansible playbook in this repository:

```bash
ansible-playbook install_docker_conda.yml
```

Source once the `~/.bashrc` file or log out and log in again.

```bash
source ~/.bashrc
```

Verify docker installation:

```bash
sudo docker run hello-world
```

Post installation steps:

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
# In case you have any permission problems make sure your user (centos) has the correct rights
# You can check this with:
ls -l

# change ownership only if necessary
sudo chown -r centos:centos /home/centos/
sudo chown -r centos:centos /mnt/volume/
```

> **_NOTE:_** Then log out and log back into the instance, after which you should be able to run `docker run hello-world` without using sudo.

### Start up rstudio server

1. Build the rstudio container (fetches the latest version of [rocker/rstudio](https://hub.docker.com/r/rocker/rstudio) and adds some custom scripts)

   You can adapt the container name inside the `docker-compose.yml` file.

   ```bash
   cd r-project-template/rstudio-server-docker
   docker-compose build
   ```

2. Add the necessary dependencies in the `code/environment.yml` file and create the conda environment. Don't add rstudio in the environment file, this is already inside the container:

   ```bash
   cd code
   conda env create -f environment.yml
   ```

   > **_NOTE:_** In case of missing permissions for writing packages, run: `sudo chown -R $USER ~/.conda`

   If you need to update the environment throughout the project, you can add the dependency to the file and run:

   ```bash
   conda env update --file environment.yml --prune
   ```

3. Update if necessary the `docker-compose.yml` file to your project paths and conda environment name.

   You may want to add additional volumes with your data.

   ```yml
   [...]
      ports:
         # port on the host : port in the container (the latter is always 8787)
         - "8889:8787"
       volumes:
         # mount conda env into exactly the same path as on the host system - some paths are hardcoded in the env.
         # TODO: Adapt conda environment name
         - /home/centos/.conda/envs/<environment-name>:/home/centos/.conda/envs/<environment-name>
         # mount the working directory containing your R project.
         # TODO: adapt </home/centos/r-project-template> to the path your data and code is under
         - /home/centos/r-project-template:/home/rstudio/data
       environment:
         # password used for authentication
         - PASSWORD=notsafe
         # repeat the path of the conda environment (must be identical to the path in "volumes")
         # TODO: Adapt conda environment name
         - CONDAENV=/home/centos/.conda/envs/<environment-name>
   ```

4. Run your project-specific instance of Rstudio-server

   ```bash
   docker-compose up
   ```

5. Log into Rstudio

   - There are two options, to ensure port forwarding works to your local machine:

     1. Using Visual Studio code:
        - `Terminal > New Terminal`
        - Press the tab `Ports`
        - Fill out the fields: `Port = 8889`, `Local address = localhost:8889`
     2. Opening a _new_ terminal window:
        > **_NOTE:_** This requires a properly set up ssh config file under `~/.ssh/config`
        - `ssh -L 8889:localhost:8889 user@host`

   - Open your server at `http://localhost:8889` (or whatever port you specified)
   - Login with the user `rstudio` and the password you specified in the `docker-compose.yml`.

6. Browse into the `code` folder and update the code as necessary. Once finished, make sure to push the changes to a new repository.

Credits: This setup for rstudio server using docker-compose is based on the setup described by [@grst](github.com/grst) on [how to run an Rstudio server in a conda environment with docker](https://github.com/grst/rstudio-server-conda).

### Finalize your analysis

Once the environment is completely setup and you don't plan to make changes anymore to the conda environment, it is time to build the container that natively includes the conda environment. The container can then be stored and used to reproduce any results later on.

For this, do the following steps:

1. Update the `docker-compose.yml`: The conda environment does not need to be mounted anymore. The data volume however needs to be mounted to an existing path. If you change the mount point, it will require further updates in the Dockerfile:

```bash
version: "3.8"
services:
  rstudio:
    build: .
    # add the image name of your container
    image: qbicprojects/rstudio-template:latest
    ulimits:
      nofile: 10000
    ports:
        - "8889:8787"
    volumes:
      # mount the working directory containing your R project.
      # TODO: adapt </home/centos/r-project-template> to the path your data and code is under
      - /mnt/volume/r-project-template/data:/home/rstudio/data
    environment:
      - PASSWORD=notsafe
```

2. Update your `Dockerfile` by copying below code and addressing the _TODO_ statements:

```bash
FROM rocker/rstudio

ENV ROOT=TRUE
ENV LANG=C.UTF-8 LC_ALL=C.UTF-8

# Create directory to mount data to
# This path needs to be identical to the one your working directory is mounted to in the docker-compose.yml under 'volumes'
RUN mkdir /home/rstudio/data

# install dependencies
RUN apt-get update --fix-missing \
  && apt-get install -y wget bzip2 ca-certificates libglib2.0-0 libxext6 libsm6 libxrender1 git \
  && apt-get clean

# install conda & setup conda environment
RUN wget \
    https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O /home/rstudio/miniconda.sh \
    && bash /home/rstudio/miniconda.sh -b -p /home/rstudio/conda \
    && rm -f /home/rstudio/miniconda.sh

ENV CONDA_DIR /home/rstudio/conda
ENV PATH=$CONDA_DIR/bin:$PATH
RUN which conda
RUN conda --version

COPY environment.yml /
RUN conda env create -f /environment.yml && conda clean -a -y

RUN conda env list
# TODO: Set the correct environment name
ENV PATH=$CONDA_DIR/envs/<environment-name>/bin:$PATH

# Settings required for conda+rstudio
# TODO: Set the correct environment name
ENV RSTUDIO_WHICH_R=${CONDA_DIR}/envs/<environment-name>/bin/R
RUN echo rsession-which-r=${RSTUDIO_WHICH_R} > /etc/rstudio/rserver.conf
RUN echo rsession-ld-library-path=${CONDA_DIR}/lib >> /etc/rstudio/rserver.conf
RUN echo "R_LIBS_USER=${CONDA_DIR}/lib/R/library" > /home/rstudio/.Renviron

# Set root password (with podman, we need to login as root rather than "rstudio")
RUN echo "root:${PASSWORD}"
RUN echo "auth-minimum-user-id=0" >> /etc/rstudio/rserver.conf

# Custom settings
RUN echo "session-timeout-minutes=0" > /etc/rstudio/rsession.conf
RUN echo "auth-timeout-minutes=0" >> /etc/rstudio/rserver.conf
RUN echo "auth-stay-signed-in-days=30" >> /etc/rstudio/rserver.conf

CMD ["/init"]
```

3. Copy the `environment.yml` to the folder `r-studio-server-docker` (where the `Dockerfile` and `docker-compose.yml` are)

4. Execute:

```bash
docker-compose build
```

> **_Note_**: It might be necessary to remove any previous built and cached containers. This can be done with: `docker rm $(docker ps -a -q); docker rmi $(docker images -q); docker system prune`.

> :warning: This command removes _all_ local containers and images! Use with caution, if you have multiple projects setup on one denbi instance.

to create this new docker file.

5. To open Rstudio and continue coding and run your analysis with the final container:

```bash
docker-compose up
```

and follow the steps described [above](#start-up-rstudio-server) in step 5.

6. Push container to docker hub:

   > **_TODO_:\_**: describe how and where to
