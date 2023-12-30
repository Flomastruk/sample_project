# Overview
## Scope
This document describes an approach to abstracting hardware and environment setup away from code development cycle. Practically speaking that means that any project developed in an accessible location (e.g. GitHub repository) can be relatively seamlessly deployed on a computer with customizable environment setup in such a way that physical device and environment can be easily reused, altered or replaced.

Main benefits of that approach compared to traditional local development include:
* Added flexibility for expirementing with hardware and software set up
* Minimal maintenance of one's local computer and its environment
* Trivialized setup consistency when collaborating on programming projects with others

## Setup Outline
The approach leverages Google Cloud Platform for hardware considerations, VSCode as an editor and for facilitating remote connections to hardware, and Docker Containers to deal with environment setup. In addition, VSCode Dev Containers framework is utilized to put things together.

# Requirements
There are three layers interacting in the suggested setup:
1. [Local Setup](#local-setup) &mdash; deals with software requirements on the physical computer
1. [Remote Host Setup](#remote-host-setup) &mdash; setup of VM on Google Cloud Platform
1. [Environment Setup](#environment-setup) &mdash; Docker container deployed on VM

By design, the complexity of the setup is pushed away to latter layers where more frequent changes are more common. The first layer is most trivial and uniform accross possible setups. The second step confines hardware customization and is rather trivial for the vast majority of needs. The third step is the most customizable and isolates all environment considerations.

## Local Setup
### [Google Cloud Platform](https://cloud.google.com/gcp)
Work with Google Cloud is commonly associated to one's google account. You'll need to set up a [Google Cloud project](https://console.cloud.google.com/projectcreate). The project scopes where computing power, storage spaces and other cloud components are managed.

If it is your first time working with Google Cloud this will likely involve setting up payment method and enabling Compute Engine service. Usually the website is quite intuitive and guides new user via helpful prompts.

To facilitate remote connections to Google Cloud VMs we use OS login mechanis. Follow [Set up OS Login](https://cloud.google.com/compute/docs/oslogin/set-up-oslogin) instructions on Google Cloud website.

### [VSCode](https://code.visualstudio.com/)
Visual Studio Code is freely available from their website, its installation process is self-explanatry. [Remote SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh) and [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extensions must be installed.

### [Google CLI](https://cloud.google.com/cli)
Follow instructions on the [Google Cloud](https://cloud.google.com/sdk/docs/install) website to install Google Command Line Interface locally.

Some one time credential settings may need to be configured once, follow directions on the website. In particular, configure [default region and zone](https://cloud.google.com/compute/docs/regions-zones/changing-default-zone-region).

### SSH Configurations
Once an appropriate Google Cloud VM is created (see [Remote Host](#remote-host-setup)) one needs to configure local files to establish SSH connection from local computer to remote host.

You can either edit your local `.ssh/config` file directly, or run _"Remote-SSH: Open SSH Configuration File"_ from VSCode Command Palette. Place the following content in the file replacing bracketed expressions `[VM_ID]`, `[VM_NAME]`, `[PROJECT_NAME]`, `[ZONE_NAME]`, `[USER_NAME]`, `USER_ID` with values appropriate to your setup.
```
Host [VM_NAME]
    HostName compute.[VM_ID]
    IdentityFile C:\Users\[USER_NAME]\.ssh\google_compute_engine
    User=[USER_ID]
    IdentitiesOnly=yes
    CheckHostIP=no
    ProxyCommand "C:\\Users\\[USER_NAME]\\AppData\\Local\\Google\\Cloud SDK\\google-cloud-sdk\\platform\\bundledpython\\python.exe" -S "C:\\Users\\[USER_NAME]\\AppData\\Local\\Google\\Cloud SDK\\google-cloud-sdk\\lib\\gcloud.py" compute start-iap-tunnel [VM_NAME] %p --listen-on-stdin --project=[PROJECT_NAME] --zone=[ZONE_NAME] --verbosity=warning
```
Hints:
1. To find `VM_ID` corresponding to `VM_NAME`, run `gcloud compute instances describe [VM_NAME] --format="yaml(id)"` in local shell
1. To find `USER_ID`, execute `whoami` on a virtual box e.g. by using cloud console
1. A helpful approach to debug ProxyCommand in `.ssh/config` is to execute from local shell `gcloud compute ssh [VM_NAME] --tunnel-through-iap --dry-run`. When executed without `--dry-run` flag the user should see a putty window opened with shell prompt on the VM
1. The top-level Host parameter can be given any name for display purposes in VSCode

Once this is done in Remote Explorer in the ribbon on the left, there should appear an option to SSH to Google Cloud VM from VSCode.

## Remote Host Setup
### Google Cloud Virtual Machine
An easy way to create a VM is to use the Google Cloud [Compute Engine webpage](https://console.cloud.google.com/compute/instances). A more programmatic (advanced) option is to execute an appropriate `gcloud compute create instances` command and specify configurations manually.

A few suggested options:
1. Choose appropriate Name, Region and Zone
1. E2 machine with 2 cores and 16GB memory (`--machine-type=e2-standard-4`) is sufficient to utilize the setup
1. In Machine configuration section, Advanced Configurations, consider _Set a time limit for the VM_ option to avoid accidentally paying for an idle compute runtime
1. In Security section check the box _Control VM access through IAM permissions_ (`--metadata=enable-oslogin=true` option)
1. Select appropriate Boot Disk (E.g. Debian GNU/Linux 12 (bookworm) x86/64, amd64 and 30GB disk size  works OK in practice)

### [Docker](https://www.docker.com/)
The steps and commands in this section are understood to be executed in a shell on the remote host.

To install Docker on the remote host follow [official instructions](https://docs.docker.com/engine/install/debian/#install-using-the-repository). Note that installing Docker on the VM simultaneously enables [git](https://git-scm.com/) distributed version control system.

After installing Docker on the VM, add current user to `docker` to be able to execute Docker commands as yourself `sudo usermod -aG docker $USER`. Note that for this step to take effect one has to restart VM.

## Environment Setup

### Required Files
To set up [VSCode Dev Container](https://code.visualstudio.com/docs/devcontainers/containers) on a VM, create a `.devcontainer` directory on your VM (.e.g under `/home`) with the contents as shown below.
```
sample_project
  |--.devcontainer
  |  |--devcontainer.json
  |  |--docker-compose.yml
  |  |--Dockerfile
  |--reqs
  |  |--requirements.txt
  |--src
  |--Notes.md
  |--LICENSE
```
Files above are described in more detail in subsequent subsections.
1. `devcontainer.json` is the file required by Dev Containers framework for setting VSCode-specific configurations
1. `Dockerfile` is the file with specifications of required Docker image
1. `docker-compose.yml` is the file following Docker Compose framework (not required for simplistic setups)
1. `requirements.txt` is a sample file with python package requirements if necessary
1. `src`, `Notes.md`, `LICENSE` etc, are sample names of files that may appear in a project

Note that `.devcontainer` directory can be a part of git repository in which case VSCode will automatically suggest an option to open project in a Dev Container.

### [`.devcontainer.json`](https://containers.dev/implementors/json_reference/)
There are three alternatives (in roughly increasing complexity) for specifying Dev Container settings depending on how Docker is to be invoked:
1. Using an existing Docker image (e.g. `python:latest` or any user defined image accessible from VM)
1. Using a `Dockerfile` with Docker image specifications
1. Using Docker Compose and referensing `docker-compose.yml` file

Configurations in `.devcontainer.json` must contain appropriate specifications through either `image`, `build` or `dockerComposeFile` keywords. In addition any desired VSCode extensions to be installed _within_ the container must be specified.

If Docker Compose setup is used, `service` keword must reference a name of the service from `docker-compose.yml`.

### [`Dockerfile`](https://docs.docker.com/engine/reference/builder/)
The file should be specified following official Dockerfile reference. Its settings entirely depend on project requirements, e.g. OS version, python version and packages, LaTex setup etc.

One useful option is to reference `requirements.txt` with python package specications in Dockerfile to be installed with `pip install -r requirements.txt`.

### [`docker-compose.yml`](https://docs.docker.com/compose/compose-file/)
For file should be specified following official Docker Compose file reference. One of the purposes of using Docker Compose framework is the ability to combine multiple interacting components. For instance, defining [Docker Volumes](https://docs.docker.com/storage/volumes/) or [Bind Mounts](https://docs.docker.com/storage/bind-mounts/) to make data available and reusable outside of the container.

Important keywords of `docker-compose.yml` are `build` or `image` specifications depening on whether an image should be taken directly or build from a Dockerfile; `tty` set to true; `restart` set to always (technicality of Dev Container setup) and `volumes` section for custom volumes and bind mounts.

Importantly, one of the services running a Docker container within `docker-compose.yml` must have the name mtching  `service` keyword value in `devcontainer.json`. This is the container that will be utilized by VSCode as Dev Container.

Technical note &mdash; when using `build` with `context` keyword, Dockerfile will be run with the its own working directory following the default behavior, however, relative paths within context will be enabled.
```
...
services:
# "custom_service" must be referenced in devcontainer.json
  custom_service:
    image: python:3.12-bookworm
...
```
### Running Dev Container
Once all configuration files are in place, invoke from Command Palette _"Dev Containers: Open Folder in Container"_ to open the directory enclosing `.devcontainer`. Read logs for debugging if necessary.

# Alternative Setups
Instead of working in Dev Container spawned by VSCode, consider attaching to a custom running container on a VM. This reduces dependency on how VM chooses to set up containers.

Consider utilizing [Google Container-Optimized OS](https://cloud.google.com/container-optimized-os/docs). This is achieved by using Container section when setting up a VM. This may be non-trivial because one has to SSH directly into a container using Remote SSH extension as opposed to Dev Containers extention (see  stackoverflow [_VSCode Remote SSH to a docker container running on GCP VM_](https://stackoverflow.com/questions/77705736/vscode-remote-ssh-to-a-docker-container-running-on-gcp-vm)).

Set up with no external IP address and Docker image sitting in Artifactory. This may be tricky because VM doesn't have access to the web outside of cloud setup.

# Other
Tex setup for VSCode can be done following [LaTeX-Workshop](https://github.com/James-Yu/LaTeX-Workshop) instructions. In particular, they have sample `.devcontainer` configurations. Corresponding VSCode extension must be added to `.devcontainer.json`.