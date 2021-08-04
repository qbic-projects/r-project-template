# R-project-template

R project template

## Running Rstudio Server with Docker on deNBI cloud

Credits: This setup is based on the setup described by [@grst](github.com/grst) on [how to run an Rstudio server in a conda environment with docker](https://github.com/grst/rstudio-server-conda).

### deNBI instance setup

Launch a deNBI instance with the following characteristics:

* Details: name the instance
* Source: Image and select CentOS image (e.g. CentOS 8.4)
* Flavour: Choose flavour (de.NBI medium should be fine for most analyses)
* Networks: Select deNBI Tübingen external network
* Security Groups: add `external_access` security group.
* Key Pair: add your SSH key or generate a new one.
* (Leave default in the rest of the fields)

Log into the instance using the IP address as host name (user name is `centos`).

> If you expect to need more than 20GB of space, mount a cynder volume on the instance.

#### Install the required software

To run the Rstudio server via docker we will require:

* [Docker](https://www.docker.com/)
* [docker-compose](https://github.com/docker/compose)
* [conda](https://docs.conda.io/en/latest/miniconda.html) or [mamba](https://github.com/conda-forge/miniforge#mambaforge)

This can be installed via ansible. Install ansible and other necessary software via yum:

```bash
sudo yum install epel-release -y
sudo yum install ansible -y
sudo yum install vim -y
sudo yum install git -y
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

Source once the `~/.bashrc` file or log out and log in again:

Verify docker installation:

```bash
sudo docker run hello-world
```

Post installation steps:

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```

Then log out and log back into the instance.

### Usage

1. Clone this repository

   ```bash
   git clone git@github.com:qbic-projects/r-project-template.git
   ```

2. Build the rstudio container (fetches the latest version of [rocker/rstudio](https://hub.docker.com/r/rocker/rstudio) and adds some custom scripts)

   ```bash
   cd rstudio-server-conda/docker
   docker-compose build     
   ```

3. Copy the docker-compose.yml file into your project directory and adjust the paths.

   You may want to add additional volumes with your data.

   ```yml
   [...]
      ports:
         # port on the host : port in the container (the latter is always 8787)
         - "8889:8787"
       volumes:
         # mount conda env into exactely the same path as on the host system - some paths are hardcoded in the env.
         - /home/sturm/anaconda3/envs/R400:/home/sturm/anaconda3/envs/R400
         # Share settings between rstudio instances
         - /home/sturm/.local/share/rstudio/monitored/user-settings:/root/.local/share/rstudio/monitored/user-settings
         # mount the working directory containing your R project.
         - /home/sturm/projects:/projects
       environment:
         # password used for authentication
         - PASSWORD=notsafe
         # repeat the path of the conda environment (must be identical to the path in "volumes")
         - CONDAENV=/home/sturm/anaconda3/envs/R400
   ```

4. Run your project-specific instance of Rstudio-server

   ```bash
   docker-compose up 
   ```

5. Log into Rstudio

 * Open your server at `http://localhost:8889` (or whatever port you specified)
 * Login with the user `rstudio` (when using Docker) or `root` (when using Podman) and the password you specified 
   in the `docker-compose.yml`. If you are using Podman and login with `rstudio` you won't have permissions to 
   access the mounted volumes. 