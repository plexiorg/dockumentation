(this isn't rst yet)


πlexicluster
Dockumentation

The underlying operating system in our case is Raspian, the minimal headless version.
It is 32 bit even though the Pi 3's processor is 64-bit capable.
openSUSE provides an official ARM-64 build for the Raspberry Pi, but its docker version is pretty old (1.12).
On top of that, openSUSE's docker repo is broken: there's an unsatisfiable dependency to an older version of runc.
You can break that and manually install the dependencies (runc, containerd).

A 64-bit OS is useful if you want to play around with ceph, since ceph has an official ARM-64 build.
There are also more Docker base images for ARM-64 than for ARM-32.


Our networking setup uses the simplest configuration possible: static IPs and hard-coded DNS with the immutable flag set on /etc/resolv.conf
The network is configured via /etc/network/interfaces, using the IPs
192.168.0.1/16
192.168.0.2/16
192.168.0.3/16
Default gateway is set to 192.168.1.100

It is possible to run a dnsmasq on one or all of the Pis, but we couldn't get that running in time.
A DNS is desirable since it should allow you to trivially add additional Pis into the cluster:
Clone the micro-SD card, attach the new Pi, use a boot script to let the Pi join the swarm, and you're done.


Some form of persistent storage is necessary to for the data of the services, like the Gerrit configuration or the Jenkins build results.
Since the services shall be able to run on any of the nodes, this storage must be shared across the network.
Originally, we wanted to use ceph for this.
However, ceph requires a fair amount of storage on each node and fast storage.
We've tried extending the storage provided by the micro-SD cards by using USB pen drives, with the usual unstable results.
In the end, we dropped ceph and just deployed a NFS on pi1.
This makes pi1 a single point of failure, of course.


The github organisation
https://github.com/plexiorg
contains our repositories for the various Docker images for the services,
as well as glue code to set up a Docker swarm and deploy the services.

The repository
https://github.com/plexiorg/nostalgia-swarm
contains glue code to create a Docker swarm, and deploy openLDAP, Gerrit and Jenkins as services into the swarm
(using existing Docker images).


In order to get any Docker image running on a Raspberry Pi, you usually have to use a different base image with ARM support.
For example, the official Jenkins Docker image is based on openjdk-alpine.
There are official Docker base image folders for ARM:
https://hub.docker.com/arm32v6
https://hub.docker.com/arm32v7
https://hub.docker.com/arm64v8
For example, arm64v8/openjdk-alpine can be used as a base image for Jenkins.
Replacing the base image is often enough to make a Docker image work on ARM.

Specifically for Jenkins, there's the `tini` executable used for some zombie management.
It is downloaded as part of the Docker build process and used later to start Jenkins.
There are ARM-builds of tini, so you can replace the URL and the checksum mentioned in the Docker image.
It is also possible to remove the tini-stuff from the Dockerfile and start Jenkins directly.


[Docker Swarm tutorial link]
TL;DR
On the designated Swarm Master, run
$ docker swarm init --advertise-addr <Swarm-Master-IP>
to create the swarm.
Any other node can then join via the command printed by docker swarm init.
Please note that most things related to Docker Swarm are persistent, meaning that the swarm will survive restarts of all nodes, including the service configurations.

It is useful to create a network for the swarm,
$ docker network create -t overlay --name <Swarm-Network-Name>
For some reason, the integrated Swarm DNS is only available when using an additional overlay network.

The integrated Swarm DNS makes each Swarm Service reachable via its service name as the host name (see below), within that overlay network.

Docker Swarm also allows you to publish ports of the deployed services.
Those ports will be bound on ALL of the nodes which are part of the swarm.
When you connect to any of the nodes on such a port, the Docker Swarm internal ingress load balancing redirects the request to the service, wherever it is running.
For example, you can reach the Gerrit web UI on all these IPs:
http://192.168.0.1:8081
http://192.168.0.2:8081
http://192.168.0.3:8081

To deploy services in the swarm, on the Swarm Master:
$ docker service create <options> --name <Swarm-Service-Name> --network <Swarm-Network-Name> <Docker-Image>
The docker images used to create services should be available on all nodes of the swarm, otherwise Docker will still attempt to start the container and fail repeatedly.
It seems after some attempts to try on a different node though, so it should work if not all nodes have all images.

To ensure all nodes have all images, a Docker Registry would probably be best.
Since we didn't have time for that (finding an ARM Docker Registry Docker image wasn't easy),
we tried building the Docker images on one node, and then using docker export + docker import.
However, this lead to strange problems, where the images appeared to be corrupted (no start command, executable not found).
So in the end, we built each image on each node.

The Jenkins image caused us some trouble, since it was "pinned" to the amd64 architecture (Placement.Architecture).
Docker Swarm did not start the image on any node, instead it kept the service in the pending state.
We guess that Swarm was waiting for an amd64 worker to join the Swarm.
It's still unclear to us why the image was referring to amd64, and this caused the delay on July 31st.
In the end, we built the Docker image again but using the name jenkins2 instead of jenkins, and it started working.


The openfrontier project,
https://github.com/openfrontier
provides a docker-based CI solution with Gerrit and Jenkins.
We used its Gerrit and LDAP Docker container.
Probably, it's possible to use some more of it, but by default, it also features certain services which were not relevant to us (e.g. Maven),
and we needed a very lightweight setup because of the restricted resources on the Pis.
Please note that openfrontier is made for amd64, so adjustments are necessary for ARM.


Gerrit offers at least three authentication mechanisms:
- HTTP
- OAuth
- LDAP

For HTTP authentication, it seems you cannot use Gerrit's internal HTTP server.
We couldn't get OAuth to authenticate us.
So we ended up with LDAP.
Of course, LDAP requires yet another service to deploy in the Swarm.
However, LDAP surprisingly is the most lightweight and least troublesome of the services.

For the lack of time, we didn't really think about the configuration of LDAP, but used some random configuration from the internets.
The container starts up with an admin user cn=admin,dc=ldap,dc=example,dc=org and the usual password.
The docker image provides with a way to prepopulate the LDAP database with additional entries.
It is also possible to directly talk to LDAP and create new users.
We couldn't figure out in time how to set a password in the preconfiguration, so this step was performed using an LDAP shell command.
It is not really necessary to expose LDAP, so no port has to be published.
Of course, it's somewhat easier for debugging to publish LDAP's port, so that's what we did.
The image however is not meant for this; it doesn't use TLS or any other form of encryption for the LDAP traffic.
Note that if you don't want to publish LDAP's port, the other services still need a way to communicate with it.
This can be achieved with the additional overlay network in Docker Swarm, since then you can use the LDAP service name as a host name to talk to the container.

After starting Gerrit, we realized that openfrontier's Gerrit configuration uses the internal Docker host name.
This causes Gerrit to publish various links using that weird internal host name, which is not reachable from outside the swarm.
For example, the redirection after login is broken, as well as the URL reported to download the commit hook which inserts change ids.
We've fixed this manually by manipulating the Gerrit configuration and restarting the service.

Note that Gerrit takes somwhere between 5 and 10 minutes to start before the web UI becomes accessible.


Jenkins takes a very, very long time to start up; a great deal of that time is spent extracting the jenkins.war file.
It might be possible to extract this prior to start-up.
It is also possible that using a 64-bit Docker image would improve performance here.
Note that 32-bit images might help reduce memory consumption, which might be a bigger issue.

We did not have time to configure Jenkins (> 30 minutes for the first start-up).
Here's an overview of the steps still necessary:
- Find out and provide the initial admin password. Since Jenkin's home directory is mounted as a volume from NFS, this is fairly simple.
- Perform the initial setup. We suggest reducing the number of plugins. The Gerrit plugin cannot be installed as part of the initial setup,
  however installation can be automated as part of the Dockerfile.
- Set up LDAP as authentication scheme. As mentioned before, you can reach the LDAP service via the hostname ldap://ldap-service
  The admin user can be used for communication with LDAP: cn=admin,dc=ldap,dc=example,dc=org
- Configure SSH key for communication with Gerrit: Jenkins has to be able to access it (location, access rights) and Gerrit has to have it assigned to some user (corny).
- Set up the Gerrit trigger, including Gerrit server communication.
- Create a test project with a gerrit trigger, push a change to Gerrit, and hope for the best.
