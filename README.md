# R-project-template

R project template

## Running Rstudio Server with Docker on deNBI cloud

Start by creating a repository within `qbic-projects` from this template. For this, use the button: `Use this template`.

### deNBI instance setup

Launch a deNBI instance with the following characteristics:

* Details: name the instance
* Source: Image and select CentOS image (e.g. CentOS 8.4)
* Flavour: Choose flavour (de.NBI medium should be fine for most analyses)
* Networks: Select deNBI Tübingen external network
* Security Groups: add `external_access` security group.
* Key Pair: add your SSH key or generate a new one.
* (Leave default in the rest of the fields)

You will also need to create and attach a volume. There are step by step instructions for creating instances and attaching volumes on the [qbic pipeline docs page](https://pipeline-docs.readthedocs.io/en/latest/markdown/clusters/denbi_cloud.html).

You should do your work and computations within the mounted volume path. You can also clone your newly created repository there.

Log into the instance using the IP address as host name (user name is `centos`).

> If you expect to need more than 20GB of space, mount a cynder volume on the instance.

### Install the required software

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

Source once the `~/.bashrc` file or log out and log in again.

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

### Start up rstudio server

1. Build the rstudio container (fetches the latest version of [rocker/rstudio](https://hub.docker.com/r/rocker/rstudio) and adds some custom scripts)

   You can adapt the container name inside the docker-compose.yml file.

   ```bash
   cd r-project-template/rstudio-server-docker
   docker-compose build     
   ```

2. Add the necessary dependencies in the `code/environment.yml` file and create the conda environment. Don't add rstudio in the environment file, this is already inside the container:

   ```bash
   cd code
   conda env create -f environment.yml
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
         - /home/centos/.conda/envs/seurat-knitr:/home/centos/.conda/envs/seurat-knitr
         # mount the working directory containing your R project.
         - /home/centos/r-project-template:/home/rstudio
       environment:
         # password used for authentication
         - PASSWORD=notsafe
         # repeat the path of the conda environment (must be identical to the path in "volumes")
         - CONDAENV=/home/centos/.conda/envs/seurat-knitr
   ```

4. Run your project-specific instance of Rstudio-server

   ```bash
   docker-compose up 
   ```

5. Log into Rstudio

   * Open your server at `http://localhost:8889` (or whatever port you specified)
   * Login with the user `rstudio` and the password you specified in the `docker-compose.yml`.

6. Browse into the `code` folder and update the code as necessary. Once finished, make sure to push the changes to a new repository.

Credits: This setup for rstudio server using docker-compose is based on the setup described by [@grst](github.com/grst) on [how to run an Rstudio server in a conda environment with docker](https://github.com/grst/rstudio-server-conda).
