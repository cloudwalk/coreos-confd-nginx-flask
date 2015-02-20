# CoreOS + confd + NGINX as reverse proxy to multiple Flask applications

# Motivation
This is a test to a system composed of multiple containers distributed in a
CoreOS cluster, each running similar (but parametrized) Flask applications.

In this proof of concept we have a simple Flask application with a name as parameter.
This name is used as a prefix for the application when making calls to a NGINX
container, which works as a reverse proxy.

As an example, let's say NGINX is accessible through `ǸGINX_HOST`.We can have
applications `A`, `B` and `C` accessible through `ǸGINX_HOST/A`, `ǸGINX_HOST/B`
and `ǸGINX_HOST/C`, respectively. The work done by NGINX is to redirect the requests
to the correct application's endpoint, through the `location PATH` instruction.

The most important aspect is that the configuration for NGINX is dynamically
generated by confd. So each time a new application is started and registered in
etcd, our `nginx.conf` is updated with a new location based on the application name
and our NGINX server is reloaded.

NGINX also proxy 404 requests to a launcher container, which can launch new Flask
apps based on the tried URL. For example, if we try to access `\my_app` and it doesn't
exists, the request will be proxied do the laucher container that will launch an
app with the given name. In the launcher server, we have a method called `isavailable`
which can be implemented to filter wheter to create a new app with given name.

We used some resources as starting points:
* To configure a cluster using Vagrant: https://coreos.com/docs/running-coreos/platforms/vagrant/
* To setup confd: https://www.digitalocean.com/community/tutorials/how-to-use-confd-and-etcd-to-dynamically-reconfigure-services-in-coreos

## Instructions

### Building the needed images
We need to build the images for NGINX, our launcher and the Flask application:
```
cd nginx
docker build -t allanino/nginx
cd ..
docker build -t allanino/launcher
cd..
cd app
docker build -t allanino/flask
```

Then we push the images to registry download later on out cluster:
```
docker push allanino/nginx
docker push allanino/launcher
docker push allanino/flask
```
### Setting up the CoreOS cluster
We need to login to on machine in our clusters and create the service files available
in `unit_files/` (or they'll be already there if we used the provided `user-data`).

First, we submit out templates service files and their sidekicks to register them,
after we can them load and start the NGINX servers and the launchers:
```
fleetctl submit nginx@.service app@.service app-discovery@.service \
  launcher@.service launcher-discovery@.service
fleetctl load launcher@{1,2} launcher-discovery@{1,2} \
  nginx@{1,2}
fleetctl start nginx@{1,2}
```

Now we can load and start multiples applications:
```
fleetctl load app@A.service app-discovery@A.service
fleetctl start app@A.service

fleetctl load app@B.service app-discovery@B.service
fleetctl start app@B.service

fleetctl load app@C.service app-discovery@C.service
fleetctl start app@C.service
```

To test it out, we can run the command `fleetctl list-units` to find the IP of the
machine running NGINX (let's say it's `NGINX_HOST`) and try to access our apps:
```
curl $NGINX_HOST/A/
curl $NGINX_HOST/A/name
curl $NGINX_HOST/B/
curl $NGINX_HOST/B/name
curl $NGINX_HOST/C/
curl $NGINX_HOST/C/name
```

To dynamically create a new app, try doing:
```
curl $NGINX_HOST/D/
```

After a while, if you access it again it will be working:
```
curl $NGINX_HOST/D/
curl $NGINX_HOST/D/name
```

Just note it can take some time to start the containers for the first time it
runs in a machine, as we need to download the image prior running the containers.

We observe that NGINX communicates with the Flask containers using their machine's
private IP. The applications IP and port are values set through our discovery
service in etcd. For example:
```
> etcdctl get /services/clusters/A
172.17.8.102:49150

> etcdctl get /services/clusters/B
172.17.8.101:49152

> etcdctl get /services/clusters/C
172.17.8.101:49151
```
