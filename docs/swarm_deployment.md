# Production Deployment Using Docker Swarm

Using Docker Swarm to deploy this codebase on more than one machine
can significantly improve performance. This document explains the steps required to do so.

## Familiarize Yourself with the Non-Swarm Production Setup
Read [this tutorial](./docs/production_non_swarm_setup.md) first.
Many of the steps and required config file changes are detailed there.
Read and follow them EXCEPT for the "Run the Production Stack" section.

## Requirements and Recommendations
- At least 3 servers running Ubuntu 16.04 or 18.04
    - The server(s) hosting high-load parts of the system (e.g. database, webserver)
    should have a healthy amount of RAM and a fast processor.
    - The servers used as tiny grading workers can be small (e.g. a quad-core CPU, 16GB RAM).
      For grading workers with higher concurrency, you'll want more than that.
    - Disk space is most important for the output and submitted files stored by the
    Django application and for the database. The grading workers don't need much disk space.
- The servers should all be able to communicate with each other over a network.
Only the swarm node labelled `webserver` (more on labels later) needs to accept incoming
traffic on ports 80 and 443.

## Install Docker Community Edition
Install Docker on every server you want to have in the swarm.

https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce-1

Here is a summary of the commands you'll need from the above link:
```
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install -y docker-ce
sudo usermod -aG docker $(logname)
```

## Initialize the Swarm
See https://docs.docker.com/engine/swarm/swarm-tutorial/ for full details.

Here is a summary of the steps from the above link:
1. Get the IP address of the machine that will be the swarm manager.
1. Initialize the swarm:
    ```
    docker swarm init --advertise-addr <MANAGER-IP>
    # Print a summary of nodes in the swarm
    docker node ls
    ```
1. Get the command needed for workers to join the swarm:
    ```
    # Copy the printed command to your clipboard
    docker swarm join-token worker
    ```
1. On each of your machines that will be worker nodes, run the command
printed in the previous step.
1. On the swarm manager node, run this command to create the swarm network:
```
docker network create ag-swarm-network
```

## Set Up NFS
First, clone the source code on one of your servers:
```
git clone --recursive git@github.com:eecs-autograder/autograder-full-stack.git
```

See https://help.ubuntu.com/lts/serverguide/network-file-system.html for full details.

Here is a summary of the steps from the above link:
1. Install NFS
    ```
    sudo apt install nfs-kernel-server
    ```
1. On the machine where you cloned the source code, edit `/etc/exports` and add
the following entry (all one line), replacing {home} with the absolute path to your home directory:
    ```
    /{home}/autograder-full-stack    *(rw,sync,no_root_squash,no_subtree_check)
    ```
1. Start the NFS server
    ```
    sudo systemctl enable nfs-kernel-server.service
    sudo systemctl start nfs-kernel-server.service
    ```
1. Create the autograder-full-stack directory on all the remaining servers:
    ```
    mkdir $HOME/autograder-full-stack
    ```
1. Mount the exported directory on those servers by adding the following entry to `/etc/fstab` on those servers, replacing {home} with the absolute path to your home directory and {server} with the hostname of the server exporting the directory:
    ```
    {server}:/{home}/autograder-full-stack /{home}/autograder-full-stack nfs rw,hard,intr 0 0
    ```
    Then, reboot the machine or run `sudo mount -a`.

## Label the Nodes
In order to avoid extra config file changes, we will apply labels to our nodes
to tell Docker Swarm which machine(s) to run which service(s) on.

Here is a list of all the labels that should be applied as needed:
- `registry`: Stores and serves Docker images needed to create services.
- `webserver`: The reverse proxy that handles incoming web traffic and the webserver that serves the frontent application.
- `django_app`: The Django backend application.
- `database`: The Postgres database.
- `cache`: The Redis cache.
- `rabbitmq_broker`: The Rabbitmq async task broker.
- `small_tasks`: Various background tasks such as queueing submissions, updating submission statuses, and building grade spreadsheets and zips of submitted files.
- `grader`: Grading worker for non-deferred tests.
- `tiny_grader`: Grading worker for non-deferred tests that only grades one submission at a time.
- `deferred_grader`: Grading worker for deferred tests.
- `tiny_deferred_grader`: Grading worker for deferred tests that only grades one submission at a time.

To apply a label to a node, run this command, replacing `{label}` and `{node hostname}`
with the label to apply and the hostname of the node:
```
docker node update --label-add {label}=true {node hostname}
```

To remove a label, run:
```
docker node update --label-rm {label} {node hostname}
```

### Label Recommendations
- The `database`, `registry`, `cache`, and `rabbitmq_broker` labels should be applied to __ONE NODE EACH__. If you ever change which node has the `database` and `registry` labels, you will need to migrate that data to the new server.
- Do NOT put `grader`/`tiny_grader`/`deferred_grader`/`tiny_deferred_grader` on the same server. Each node intended as a grading worker should have at most one of these labels.

## Create the Registry Service
Run the following command:
```
docker service create --name registry --publish 5000:5000 --mount type=volume,source=ag-registry,destination=/var/lib/registry --constraint 'node.labels.registry == true' registry:2
```