# Overview
## Scope
This document describes an approach to abstracting hardware and environment setup away from code development cycle. Practically speaking that means that any project developed in an accessible location (e.g. GitHub repository) can be relatively seamlessly deployed on a computer with customizable environment setup in such a way that physical device and environment can be easily reused, altered or replaced.

Main benefits of that approach compared to traditional local development include:
* Adds flexibility for expirementing with hardware and software set up
* Removes the need for maintenance of one's computer and its environment
* Trivializes setup consistency when collaborating on programming projects with others

## Setup Outline
The approach leverages Google Cloud Platform for hardware considerations, VSCode as an editor and for facilitating remote connections to hardware, and Docker Containers to deal with environment setup. We also rely on VSCode Dev Containers framework to put things together.

# Requirements
There are three layers interacting in the suggested setup:
1. [Local Setup](#local-setup) &mdash; deals with software requirements on the physical computer
2. [Remote Host Setup](#remote-host-setup) &mdash; setup of VM on Google Cloud Platform
3. [Environment Setup](#environment-setup) &mdash; Docker container deployed on VM

By design, the complexity and flexibility of the setup increase with each next layer. The first layer is most trivial and uniform accross possible setups. The second step confines hardware customization and is rather trivial for the vast majority of needs. The third step is the most customizable and segregates all environment considerations

## Local Setup
### [Google Cloud Platform](https://cloud.google.com/gcp)
Work with Google Cloud is commonly associated to one's google account. You'll need to set up a [Google Cloud project](https://console.cloud.google.com/projectcreate). The project scopes where computing power, storage spaces and other cloud components are managed.

If it is your first time working with Google Cloud this will likely involve setting up payment method and enabling Compute Engine service. Usually the website is quite intuitive and guides new user via helpful prompts.

To facilitate remote connections to Google Cloud VMs we use OS login mechanis. Follow [Set up OS Login](https://cloud.google.com/compute/docs/oslogin/set-up-oslogin) instructions on Google Cloud website.

### [VSCode](https://code.visualstudio.com/)
[Remote SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh) and [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extensions must be installed.

### [Google CLI](https://cloud.google.com/cli)
Follow instructions on the [Google Cloud](https://cloud.google.com/sdk/docs/install) website to install Google Command Line Interface.

Some one time credential settings may need to be set up, follow directions on the website. In particular, configure [default region and zone](https://cloud.google.com/compute/docs/regions-zones/changing-default-zone-region).

### SSH Configurations
Once an appropriate Google Cloud VM is created (see [Remote Host](#remote-host-setup)) one needs to configure local files to establish SSH connection from local computer to remote host.

You can either edit your local `.ssh/config` file directly, or run _"Remote-SSH: Open SSH Configuration File"_ from VSCode Command Palette. Place the following contetst in the file replacing bracketed expressions `[VM_ID]`, `[VM_NAME]`, `[PROJECT_NAME]`, `[ZONE_NAME]`, `[USER_NAME]`, `USER_ID` with values appropriate to your setup.
```
Host compute.[VM_ID]
    HostName compute.[VM_ID]
    IdentityFile C:\Users\[USER_NAME]\.ssh\google_compute_engine
    User=[USER_ID]
    IdentitiesOnly=yes
    CheckHostIP=no
    ProxyCommand "C:\\Users\\[USER_NAME]\\AppData\\Local\\Google\\Cloud SDK\\google-cloud-sdk\\platform\\bundledpython\\python.exe" -S "C:\\Users\\[USER_NAME]\\AppData\\Local\\Google\\Cloud SDK\\google-cloud-sdk\\lib\\gcloud.py" compute start-iap-tunnel [VM_NAME] %p --listen-on-stdin --project=[PROJECT_NAME] --zone=[ZONE_NAME] --verbosity=warning
```
Hints:
1. To find `VM_ID` corresponding to `VM_NAME`, run `gcloud compute instances describe [VM_NAME] --format="yaml(id)"` in local shell
2. To find `USER_ID`, execute `whoami` on a virtual box e.g. by using cloud console
3. A helpful approach to debug ProxyCommand in `.ssh/config` is to execute from shell `gcloud compute ssh [VM_NAME] --tunnel-through-iap --dry-run`. When executed without `--dry-run` flag the user should ses a putty window opened with shell prompt on the VM

Once this is done in Remote Explorer in the ribbon on the left, there should appear an option to SSH to Google Cloud VM.

## Remote Host Setup
### Google Cloud Virtual Machine
An easy way to create a VM is to use the Google Cloud [Compute Engine webpage](https://console.cloud.google.com/compute/instances). A more programmatic (advanced) option is to execute an appropriate `gcloud compute create instances create` command and specify configurations manually.

A few suggested options:
1. Choose appropriate Name, Region and Zone
1. E2 machine with 2 cores and 16GB memory (`--machine-type=e2-standard-4`) is sufficient to utilize the setup
1. In Machine configuration section, Advanced Configurations, consider _Set a time limit for the VM_ option to avoid accidentally paying for an idle compute runtime
1. In Security section check the box _Control VM access through IAM permissions_ (`--metadata=enable-oslogin=true` option)
1. Select appropriate Boot Disk (E.g. Debian GNU/Linux 12 (bookworm) x86/64, amd64 and 30GB disk size  works OK in practice)

### [Docker](https://www.docker.com/)
The steps and commands in this section are understood to be executed in a shell on the remote host.

To install Docker on the remote host follow [official instructions](https://docs.docker.com/engine/install/debian/#install-using-the-repository).

Note that installing Docker on the VM simultaneously enables [git](https://git-scm.com/) distributed versino control system.

## Environment Setup
### .devcontainer

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

# Alternative setups
Instead of working in Dev Container spawned by VSCode, consider attaching to a custom running container on a VM. This reduces dependency on how VM chooses to set up containers

Consider utilizing [Google Container-Optimized OS](https://cloud.google.com/container-optimized-os/docs). This is achieved by using Container section when setting up a VM. This may be non-trivial because one has to SSH directly into a container using Remote SSH extension as opposed to Dev Containers extention ([see  stackoverflow _VSCode Remote SSH to a docker container running on GCP VM_](https://stackoverflow.com/questions/77705736/vscode-remote-ssh-to-a-docker-container-running-on-gcp-vm)).

Set up with no external IP address and Docker image sitting in Artifactory. This may be tricky because VM doesn't have access to the web outside of cloud setup.