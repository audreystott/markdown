# Charliecloud

Charliecloud is another service tool that enables containerisation of software stacks that are, similar to Singularity, deployable on HPC systems. Charliecloud containers are built from Dockerfiles, and these containers can then be copied into and run at scale on Pawsey HPC systems.

A key feature of Charliecloud containers is the non-isolation mode they are run in, that are not available with Singularity and other container services:

- Sharing of resources from the host is allowed as Charliecloud containers share the same network namespace as the host
- Charliecloud containers allow files to be written in the container's directory path, if a host directory is mounted on it

Simply put, reading and writing between host and container becomes seamless.

## Workflow
Charliecloud is installed on all Pawsey systems. The workflow for Charliecloud is simple:
1. Build a Charliecloud container
2. Run the container

## Set up
### Building a Charliecloud container

Before you being, start an interactive session on a Zeus node:

    salloc -n 1 -t 1:00:00

- Note that if you are normally running containers on your jobs, you will be submitting a Slurm batch script instead of running the container in an interactive session.
  
The workflow for building a Charliecloud container involves a few steps:
1. Building an image from a Dockerfile (`ch-build`)
2. Flattening the builder image into a Charliecloud image tarball (`ch-builder2tar`)
3. Unpacking the tarball (`ch-tar2dir`)
  
As an example, we have a Dockerfile containing the following located in the directory we want to build from: 

    FROM debian:stretch

    RUN    apt-get update \
    && apt-get install -y openssh-client \
    && rm -rf /var/lib/apt/lists/*

    COPY . hello

    RUN touch /usr/bin/ch-ssh

To build the image, we use the ch-build command:

    >ch-build -t hello .

Flatten the image to a directory, e.g. /var/tmp:

    >ch-builder2tar hello /var/tmp

Then unpack the tarball:

    >ch-tar2dir /var/tmp/hello.tar.gz /var/tmp

## Using the container

You should now be able to run commands from the Charliecloud container `hello`:

    >ch-run /var/tmp/hello -- echo hello
    hello

Note that the argument  `--` separates options from non-option arguments, and is required to run the command.

### Working with images

Charliecloud images are located in the directory where you unpacked the tarball. To make it easy to track, we recommend that you unpack all images in the same directory. In this case, we use `/var/tmp`.

### Bind mounting host directories

Host directories can be bind mounted to the container so that files can be read and written to the specific host directory. For example, create a directory called `foo`:

    >mkdir /var/tmp/foo

Then add a file called `bar` that contains the word `hello`:

    >echo hello > /var/tmp/foo/bar

Bind the directory and run a bash shell from the container. You should see the file `bar` in the `/mnt` folder in the container as such:
    
    >ch-run -b /var/tmp/foo:/mnt /var/tmp/hello -- bash
    >ls /mnt
    /bar
    >echo /mnt/bar
    hello

### Running your container as a Slurm job

You will run your Charliecloud containers as [Slurm jobs](https://support.pawsey.org.au/documentation/display/US/Submitting+and+Monitoring+Jobs). Below is an example Slurm batch script for running a container called `openfoam_v1812`. MPI processes are spawned from outside the container, that is, on the host. `mpi-OpenFOAMCase.sh`:

    #!/bin/bash -l
 
    #SBATCH --account=projectID
    #SBATCH --job-name=mpi-OpenFOAMCase
    #SBATCH --ntasks=480
    #SBATCH --ntasks-per-node=24
    #SBATCH --time=00:20:00
    #SBATCH --export=none

    # Load Charliecloud
    module load charliecloud

    # cd into the case directory
    cd $MYSCRATCH/myOpenFOAMCase

    # execute the simpleFoam parallel solver with MPI
    srun --export=all -n $SLURM_NTASKS ch-run --set-env=/var/tmp/openfoam_v1812/ch/environment /var/tmp/openfoam_v1812 -- simpleFoam -parallel -fileHandler collated | tee log.simpleFoam

- Change the projectID accordingly
- `--set-env=/var/tmp/openfoam_v1812/ch/environment` is where the original Dockerfile's environment will be applied to the container. Without using `--set-env`, the container will use the environment of the host

You will then submit your batch script as follows:

    >sbatch mpi-OpenFOAMCase.sh

### Running JupyterHub with Charliecloud

Running JupyterHub with Charliecloud is easy, and any packages installed or workflows run is saved on the host to come back to later.

For example, we start by saving the [Dockerfile](https://hub.docker.com/r/jupyter/datascience-notebook/dockerfile) for jupyter/datascience-notebook on our host computer. 

We then run ch-build, ch-builder2tar and ch-tar2dir to create a Charliecloud container called `jupyter.datascience-notebook`.

    >ch-build -t jupyter/datascience-notebook .
    >ch-builder2tar jupyter/datascience-notebook /var/tmp
    >ch-tar2dir /var/tmp/jupyter.datascience-notebook.tar.gz /var/tmp

To run the notebook, we will run the following Slurm batch script, `jupyter-notebook.sh`:

    #!/bin/bash -l
 
    #SBATCH --account=[your-project-name]
    #SBATCH --partition=workq
    #SBATCH --nodes=1
    #SBATCH --time=02:00:00
    #SBATCH --job-name=jupyter_notebook
    #SBATCH --export=NONE

    # Get the hostname of the Zeus node
    # We'll set up an SSH tunnel to connect to the Juypter notebook server
    host=$(hostname)
    
    # Set our working directory
    # This is the directory we'll mount to /home/jovyan in the container
    # Must be under /group or /scratch
    dir=${MYSCRATCH}/jupyter-dir
    cd ${dir}

    # Load Charliecloud
    module load charliecloud

    # Launch our container 
    ch-run -b ${dir}:/home/jovyan -b /usr:/usr --no-home --set-env=/var/tmp/jupyter.datascience-notebook/ch/environment /var/tmp/jupyter.datascience-notebook -- jupyter-notebook

- `-b ${dir}:/home/jovyan` binds your working directory on the host to the `/home/jovyan` directory in the container. This is important as the notebook creates caches and installs packages in this directory. Without the bind mount of the host directory, they are not possible.
- `-b /usr:/usr` binds the `/usr` directory of the host to the `/usr` directory of the container, so that resources on the host can be shared. This is helpful if the package you are trying to install on JupyterHub requires certain system dependencies.
- `--no-home` prevents binding of the host's `/home` directory so that `/home/jovyan` on the container can be bind mounted
- `--set-env=/var/tmp/jupyter.datascience-notebook/ch/environment` is where the original Dockerfile's environment will be applied to the container. Without using `--set-env`, the container will use the environment of the host
- `-- jupyter-notebook` starts a notebook on a server

After submitting your Slurm batch script:

    sbatch jupyter-notebook.sh

you will get an output similar to this:

    [I 07:35:40.634 NotebookApp] JupyterLab extension loaded from /opt/conda/lib/python3.7/site-packages/jupyterlab
    [I 07:35:40.634 NotebookApp] JupyterLab application directory is /opt/conda/share/jupyter/lab
    [I 07:35:42.833 NotebookApp] Serving notebooks from local directory: /
    [I 07:35:42.834 NotebookApp] The Jupyter Notebook is running at:
    [I 07:35:42.834 NotebookApp] http://z052:8888/?token=a6a428f75584b2d14ba5d4fcf0105fd78bfb8f8b3332178f
    [I 07:35:42.834 NotebookApp]  or http://127.0.0.1:8888/?token=a6a428f75584b2d14ba5d4fcf0105fd78bfb8f8b3332178f
    [I 07:35:42.834 NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
    [C 07:35:42.842 NotebookApp] 
    
    To access the notebook, open this file in a browser:
        file:///home/jovyan/.local/share/jupyter/runtime/nbserver-23096-open.html
    Or copy and paste one of these URLs:
        http://z052:8888/?token=a6a428f75584b2d14ba5d4fcf0105fd78bfb8f8b3332178f
     or http://127.0.0.1:8888/?token=a6a428f75584b2d14ba5d4fcf0105fd78bfb8f8b3332178f

First, run the following on a new terminal:

    >host=$(hostname)
    >ssh -N -f -L 8888:$(host):8888 $USER@zeus.pawsey.org.au 


Then, open a web browser and enter `http://127.0.0.1:8888` on the address line. You will then be asked to enter a password or token, of which the latter is the string of letters and numbers e.g. in this case is `token=a6a428f75584b2d14ba5d4fcf0105fd78bfb8f8b3332178f`. Copy and paste this on the web page to enter your notebook.

![](/home/ubuntu/markdown/jupyter-screenshot.png)

Once you're in, navigate to `/home/jovyan` and you will see your files from your host working directory on there. You can then start a new notebook or open an existing one.

### Storing your software in the image

If you have a software on the host that can be stored in the image for portability, or if the software is stable and not under active development,you can build your container with the software stored in it. 

The example here is `openmpi`. The Dockerfile below copies the files for the `openmpi` software into a container:

    FROM openmpi
    COPY . /hello
    WORKDIR /hello
    RUN make clean && make

You would then run the following commands to build the container that will now store the `openmpi` software:

    >ch-build -t mpihello .
    >ch-builder2tar mpihello /var/tmp
    >ch-tar2dir /var/tmp/mpihello.tar.gz /var/tmp

You will find the copied files in the container by running the following command:

    >ch-run /var/tmp/mpihello -- ls -lh /hello

### Compiling software that is stored on the host

If you have a software that is under active development on the host and would prefer not to have it stored in the container, you can bind the directory to the container, and compile it within the container. The following example involves a Makefile and other support files for the software titled `mpihello`. To compile it using `make` within the container, run the following command:

    ch-run -b . --cd /mnt/ /var/tmp/mpihello -- make

- `-b` is the argument for binding, in this case, the current directory on the host
- `--cd` is the argument for setting an initial current directory for the container, which is `/mnt` in this case
- `/var/tmp/mpihello` is the Charliecloud container we are using in this example
- `make` is the command for compiling this particular software

