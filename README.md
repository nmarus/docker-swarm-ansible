# Docker Swarm with GlusterFS, Traefik, HAProxy, and Portainer

The goal of this repo is to demonstrate using Ansible to build out Docker Swarm architecture that can be used to simply and reliably deploy and manage container workloads. This setup is intended for a small Development Lab and to get familiar with how high availability, scaling, and routing works with Docker. This can be scaled to use in production and there are some notes at the end of this README that will point you in the right direction if you desire to do so.

This setup will provide a 3 node Docker Swarm with Layer 7 routing, Web UI management, and a distributed file system for Container volumes. A HAProxy node is setup to load balance traffic between each of the Layer 7 routers on the Docker Swarm nodes.

### Technologies in Use

#### Docker
Containers, nuff said.

#### Docker Swarm
Uses Docker to create a swarm of Docker hosts where you can deploy application services. A Swarm is managed like a single Docker node and allows service networks to span across multiple hosts.

#### Gluster
Gluster is a Distributed Filesystem to allow shared persistent storage volumes across Docker Swarm Cluster. This is used as an alternative to NFS or other shared storage technologies for simplicity and minimal hardware footprint. This example will install the Gluster Storage Plugin for Docker.

#### Traefik
Traefik is a Layer 7 Router that automatically discovers services in Docker Swarm and routes traffic to the appropriate container(s) in the cluster. Similar to the functionality provided by Docker EE UCP with Interlock. The nice thing about Traefik is that it integrates into the Docker Swarm API and will dynamically create the Layer 7 routes based on labels you add to you containers or services. There is no administrative action in Traefik itself.

#### Portainer
Portainer is a web UI for managing Docker Swarm. Functionality is similar to Docker EE UCP but provides more features and is an open source project.

#### HAProxy
HAProxy is a load balancer and SSL off loader. This is typically placed in front of the application servers in a Cluster. In this setup, HAProxy does SSL Offloading and load balances requests to each of the Traefik services running on the Docker Swarm nodes. It additionally monitors each of the hosts and if one is no longer available, will remove that host from the distribution of traffic until it is brought back online.

## Deployment

### Requirements

**Setup Ansible**

This makes use of Ansible to automate the deployment of this setup. To run the Ansible commands, your workstation should have the Ansible CLI utilities installed.

_For OSX with Homebrew:_

```bash
brew install ansible
```

**Setup Vagrant**

If wanting to utilize Vagrant, you will need to make Vagrant and Virtualbox are both installed.

_For OSX with Homebrew:_

```bash
brew cask install vagrant
brew cask install virtualbox
```

### Setup Machines to Host the Swarm Cluster

#### Vagrant

If you wish to try this setup with Vagrant the following steps can be used to get up and running fast. The Vagrant file deploys 4 hosts.

* 1 HAParoxy Host
* 3 Docker Swarm Hosts

To bring up the infrastructure, run:

```bash
# install disk resize plugin
vagrant plugin install vagrant-disksize

# launch vagrant environment
vagrant up
```

To shut down the infrastructure in Vagrant, run:

```bash
vagrant halt
```

To tear down all the infrastructure in Vagrant, run:

```bash
vagrant destroy -f
```

#### External Hosts or Manual Setup

If wanting to deploy on your own defined infrastructure you will need to provision 4 hosts (ie, VMWare Fusion / Workstation / ESXi / AWS / etc...)

This has been tested with Ubuntu 18.04, but should* work with any Debian flavor of linux.

* 4 Hosts running Ubuntu 18.04
* Primary User of 'ubuntu' with password 'ubuntu' (modifiable in hosts definition)
* SSH Service available that above credentials can log in to
* The 'ubuntu' user should have access to run `sudo`
* Each host should have:
  * 1 Core
  * 1 GB Ram
  * Primary drive of 20GB where OS is installed
* The 3 Docker Swarm hosts should also have a econdary non-initialized 10GB+ Drive. This will be used for XFS file system and Gluster Volumes.

## Initialize Infrastructure

Ansible is used to provision the infrastructure.

All customizations are found in the Ansible Inventory that is defined in the files:

* `hosts` - Inventory of hosts
* `host-vagrant` - Inventory of hosts for vagrant setup
* `playbooks/config.yml` - Application config options

Inventory Groups:

* `[haproxy]` SINGLE node that will be configured as the front end load balancer with HAProxy
* `[swarm_managers]` group in hosts MUST define at minimum 3 Nodes
* `[swarm_workers]` group in hosts is optional and can include any number of non Manager Nodes
* `[gluster_nodes]` group in host MUST define ONLY 3 Nodes

### Deployment Overview

The following high level actions are performed by the Ansible script:

* Updates hosts packages
* Installs required dependencies
* Installs Docker CE on swarm nodes
* Sets up Ansible user to have permission to run Docker without sudo access
* The first node defined in `[swarm_managers]` is setup as the leader in the Swarm Cluster
* The remaining nodes are setup as Swarm Managers (3 required for clustering, ie 1 Leader, 2 managers)
* If `[swarm_workers]` is defined, joins these to the Swarm Cluster with the Worker role
* On `[gluster_nodes]`, configures the second drive (sdb) with a XFS file system
* Sets up a GlusterFS Cluster on nodes in `[gluster_nodes]` (Ansible will assume there are 3 total, others in this group will not be initialized)
* Install Docker Gluster Storage Plugin on all `[swarm_managers]` and `[swarm_workers]`
* Installs [Traefik](https://traefik.io) stack as a global Service on all `[swarm_managers]`
* Install [Portainer](https://www.portainer.io) Agents as a global service on all `[swarm_managers]` and `[swarm_workers]`
* Install Portainer UI with a replica set of 1 on `[swarm_managers]`
* Install HAProxy on dedicated node
* Creates a self signed certificate for SSL offloading with HA Proxy
* Routes traffic received on port 80 and 443 to the Traefic Services running on the Swarm manager Nodes.

### Setup DNS

For the Layer 7 routing to work, the DNS names must be used when accessing the apps through HAProxy and in turn, the Traefik Layer 7 Router.

Each of the deployed applications must resolve to the host running HAProxy. You can accomplish this in one of two ways.

1. DNS - If you a local DNS server, or are deploying into a public cloud, create a DNS entry that points to the HAProxy Host.

2. Local Dev via Hosts File - If you are testing on a local network or in a dev environment, it may be easier to simply modify your hosts file. Example entries below will match the default names defined in the variables of the Ansible hosts file.

**Example Entries in local Hosts File:**

```
10.10.10.10 portainer.docker.local
10.10.10.10 traefik.docker.local
10.10.10.10 wordpress.docker.local
```

### Test Connectivity to Hosts

This should return ok and ensure that Ansible config is able to login and execute commands on the hosts.

**For Vagrant Infrastructure**

```bash
ansible all --inventory-file=hosts-vagrant -a "/bin/echo hello"
```

**For External Hosts or Manual Setup on Ubuntu 18.04**

```bash
ansible all -a "/bin/echo hello"
```

### Run Ansible Playbook

Once everything is configured and verified, run the playbook to configure the clusters.

**For Vagrant Infrastructure**

```bash
ansible-playbook --inventory-file=hosts-vagrant playbooks/install.yml
```

**For External Hosts or Manual Setup on Ubuntu 18.04**

```bash
ansible-playbook playbooks/install.yml
```

### Test Connectivity to Traefik

If using the default Ansible variables defined in the hosts file, you can navigate to:

* http://traefik.docker.local

Username/Password = admin:password1234

### Test Connectivity to Portainer

If using the default Ansible variables defined in the hosts file, you can navigate to:

* http://portainer.docker.local

Username/Password = admin:password1234

### Test Connectivity to HAPRoxy Status Page

If using the default Ansible variables defined in the hosts file, you can navigate to:

* http://10.10.10.10/stats

Username/Password = admin:password1234

### Test Deploying a Simple Application Stack to the Swarm

**wordpress-stack.yml**

_Note: This assumes that you are using the default Ansible variables in the hosts file._

```yaml
version: '3.5'
services:
  wordpress:
    image: wordpress
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    ports:
      - "80"
    networks:
      - wordpress
      - web
    volumes:
      - wordress_data:/var/www/html/wp-content
    deploy:
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.wordpress.rule=Host(`wordpress.docker.local`)"
        - "traefik.http.services.wordpress.loadbalancer.server.port=80"
        - "traefik.docker.network=web"

  db:
    image: mysql:5.7
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    networks:
      - wordpress
    volumes:
      - mysql_data:/var/lib/mysql

networks:
  wordpress:
    driver: overlay
    attachable: true
    name: wordpress
  web:
    external: true
    name: web

volumes:
  wordress_data:
    driver: glusterfs
    name: "gfs/wordpress_data"
  mysql_data:
    driver: glusterfs
    name: "gfs/wordpress_mysql"
```

**Deploying stack from CLI**

From a Docker Swarm Node (Manager or Worker):

```bash
# Create Directories on Mounted GFS volume for storage
sudo mkdir /mnt/gfs/wordpress_data
sudo mkdir /mnt/gfs/wordpress_mysql

# Deploy above wordpress stack
docker stack deploy --compose-file=wordpress-stack.yml wordpress
```

**Deploying Stack from Portainer UI**

1. Access Portainer UI at http://portainer.docker.local
2. Select Stacks
3. Name the stack 'wordpress'
4. Copy and Paste above yaml into Web Editor
5. Click 'Deploy Stack'

**Test Wordpress**

Navigate to: http://wordpress.docker.local

_Note: This may take a few minutes to respond after creating stack due to docker downloading images referenced and them starting up._

### Breakdown of Required Elements Defined in Stack File

**Services That Will Require a Eternal Web Route (Layer 7 via Traefik)**

_Note: For a full example reference Wordpress config above._

Ensure the following:

1. The service declaration in your stack yaml file should have the port that will be used to access the web application exposed. This does not require it to be mapped to a host port.

   Example:

   ```yaml
   services:
     myservice:
       [...]
       ports:
          - "80"
       [...]
   ```

2. The deploy option on the exposed service must have the Traefik network assigned and Traefik labels added. The router name and service are arbitrary, but must be unique in Traefik.

   Example:

   ```yaml
   services:
     myservice:
       [...]
       networks:
         <traefik_network_name>:
           external: true
           name: <traefik_network_name>
       [...]
       deploy:
         labels:
           - "traefik.enable=true"
           - "traefik.http.routers.<router_name>.rule=Host(`<web_url>`)"
           - "traefik.http.services.<service_name>.loadbalancer.server.port=<container_port>"
           - "traefik.docker.network=<traefik_network_name>"
   ```

3. The Traefik Network must be defined in your compose file in order to assign it to a service. This network has already been created if you have already installed Traefik so it must be reference as an external service.

   Example:

   ```yaml
   [...]
   networks:
     <traefik_network_name>:
       external: true
       name: <traefik_network_name>
   ```

**Services That Will Require Persistent Storage (via GlusterFS)**

_Note: For a full example reference Wordpress config above._

Ensure the following:

1. The volumes are defined that use the Gluster Storage Driver.

   Example:

   ```yaml
   volumes:
     <volume_name>:
       driver: glusterfs
       name: "gfs/<volume_dir>"
   ```

2. The volumes of the service use the defined volumes.

   Example:

   ```yaml
   services:
     myservice:
       [...]
       volumes:
         - <volume_name>:<container_path>
   ```

3. The volume dir path has been created on in Gluster volume.

   Example:

   ```bash
   sudo mkdir <gluster_mount_path>/<volume_dir>
   ```

## Re-running Ansible Playbook

It is possible to re-run the playbook. Most of the tasks check to see if they were already completed and will be skipped. The most applicable use case to re-run the playbook is when adding additional workers or managers. Re-running the playbook after changing the Gluster node assignment and/or most of the environment variables however is not supported and will likely break stuff or cause the Ansible playbook to fail.

Look to the playbooks for `upgrade-docker`, `upgrade-packages`, and `redeploy-apps` for some maintenance operations.

## Semi Production Setup

If deploying this public cloud, or are looking for a more robust infrastruture, you will probably want to increase the total number of nodes, make use of a load balancer, and implement SSL. This is not a complete guide on how to do this, but offers some initial insights on where to start. When referencing Public Cloud below, this will refer to Amazon Web Services, however this can also be applied to a hybrid (on premise data center) or other cloud provider with minor changes.

**Ansible Modifications**

When deploying this to public cloud and/or in an environment with DNS, you will want to switch out the inventory from using ip address to using host names. Additionally, you will probably want to not use a username / password method of connecting over SSH and should rather use SSH keys.

For the DNS name modifications, change the entries in the hosts file to use those names. These names MUST also be resolvable from each host internally. For example if you can access hosts at myhost1.mycorp.net, myhost2.mycorp.net, etc. Each of those hosts internally should also be able to access the other hosts by the same DNS name.

To setup the Ansible script to use SSH keys vs password, you will want to remove `ansible_ssh_pass` and `ansible_become_pass` variable declaration and add `ansible_ssh_private_key_file` in the Ansible host file:

```
ansible_ssh_private_key_file=/path/to/ssh_key.pem
```

In addition to this, you want to make sure that the user can run `sudo` without a password. This will be the typical setup when using an Ubuntu AMI from AWS with EC2.

**Load Balancer and SSL**

While Traefik can be configured to do SSL offloading, often a better better approach is to make use of an AWS Application Load Balancer (ALB). The advantages to this is that it can do the SSL offloading as well as distribute traffic between each of your Traefik router nodes.

For each domain you wish to host Docker applications under:

1. Create a AWS Route 53 DNS domain and assign it as the name server from your domain registrar.
2. Create a AWS ALB and specify each one of your Traefik Nodes (Docker Managers) as targets.
3. Apply Healthchecks to the ALB so that it can determine if a specific host is up or down.
4. Add your SSL cert (or optionally a wildcard cert if hosting multiple apps on the same domain) to the ALB
4. In your Route 53 DNS Domain, create DNS CNAME records to point the application(s) name to you ALB. (i.e `www.mydomain.com`, `file.mydomain.com`)
5. Alternatively, if you are hosting all your applications for a domain in Docker Swarm you can create a DNS CNAME wildcard record (i.e. `*.mydomain.com`) and point that at your ALB. The advantage to this is that you will not need to create individual Route 53 records each time you add a new application. However, this assumes that all application records are under the same domain.

**Scaling Docker Swarm**

While this 3 node setup can handle a decent workload, you may eventually want to increase the number of nodes that run your application, or split out the Gluster Cluster.

To do this you can modify the Ansible hosts file. For example, to split out Gluster into its own dedicated compute and add 3 worker nodes to the swarm, you can modify the following:

```
[swarm_managers]
10.10.10.10
10.10.10.20
10.10.10.30

[swarm_workers]
10.10.10.40
10.10.10.50
10.10.10.60

[gluster_nodes]
10.10.10.70
10.10.10.80
10.10.10.90
```

**Other Considerations**

While Gluster is a good way to have a distributed filesystem for use with Swarm, if you are deploying on a Cloud provider, it would be advisable to use another volume storage Driver. What you choose for this will depend on performance, cost, and complexity requirements. Amazons EFS (NFS) is one of many ways to accomplish this.

For high traffic sites, you will also want to break out the Traefic router to run on dedicated hosts. This is out of the scope of this Ansible playbook, but should be fairly easy to modify. An option would be to setup 2 dedicated Swarm Workers and deploy Traefik services only to those. However, to keep other workloads off these workers, or to make sure Traefik services are only assigned here, you will have to make use of Docker labels and constraints.

If running in environments where you do not have access to inherent SSL offloading and load balancing that is provided from services like AWS ALB, you could make use of a setup that implements HAProxy as an alternative. A simple example of this is provided with this setup.

Traefik also has this capability as integrations for LetsEncrypt SSL certs (good service if you want free SSL) however, you would still have to load balance between each of your Traefik nodes (assuming you want more than one). Note that if you go down the Traefik route with LetsEncrypt, you will need to also implement a shared key store between the nodes (i.e. Consul).
