# Create a simple Cloud Init service for your homelab

Cloud Init is an industry standard, widely utilized method for initializing cloud instances. Cloud providers use Cloud Init to customize instances with network configuration, instance information, and even user-provided configuration directives. Cloud Init is a great tool to add to your Private Cloud at Home to learn more about how the large cloud providers work, and to add a little automation to the initial setup and configuration of both virtual and physical machines within your homelab. For a bit more detail, check out my previous article on [what Cloud Init is and why it is useful](LINK TO PREVIOUS ARTICLE).

Admittedly, Cloud Init has more utility to a cloud provider provisioning machines for many different clients.  For a homelab run by a single sysadmin, much of what Cloud Init solves might be a little superfluous.  However, getting it setup and learning how it works is a great way to learn more about this cloud technology, and it does still have utility for configuring your devices on first boot.

Today we will learn about Cloud Init and its "NoCloud" datasource, designed to allow the use of Cloud Init outside of a traditional cloud provider setting.  To do that we will install Cloud Init on a client device and set up a container running a web service to respond to the client's requests. We will investigate what the client is requesting from the web service and modify the web service container to serve a basic, static Cloud Init service.

## Set up Cloud Init on an existing system

Cloud Init probably provides the most utility on first boot of a new system, querying for configuration data and making those changes to customize the system as directed. It can be included in a disk image for Raspberry Pis and single-board computers, or added to images used to provision virtual machines. ([Learn more about customizing disk images for your Raspberry Pi homelab.](https://opensource.com/article/20/5/disk-image-raspberry-pi))  For testing, however, it is easy to install and run Cloud Init on an existing system, or install a new system and then setup Cloud Init for testing.

As a major service used by most cloud providers, Cloud Init is supported on most Linux distributions. For this example, I will be using Fedora 31 Server for the Raspberry Pi, but this can be done on Raspbian, Ubuntu, CentOS, and most other distributions the same way.

### Install and enable the cloud-init services

On a system that you would like to be a Cloud Init client, install the cloud-init package with `dnf` (if using Fedora):

```shell
# Install the cloud-init package
dnf install -y cloud-init
```

Cloud Init is actually four different services (at least with Systemd), each in charge of retrieving config data and performing configuration changes during a different part of the boot process, allowing for greater flexibility in what can be done. While it is unlikely you will intereact with these services directly much, it is useful to know what they are in the event you need to troubleshoot something:

* cloud-init-local.service
* cloud-init.service
* cloud-config.service
* cloud-final.service

Enable all four services:

```shell
# Enable the four cloud-init services
systemctl enable cloud-init-local.service
systemctl enable cloud-init.service
systemctl enable cloud-config.service
systemctl enable cloud-final.service
```

### Configure the datasource to query

Once the service is enabled, configure the datasource from which the client will query the config data. There are a [large number of datasource types](https://cloudinit.readthedocs.io/en/latest/topics/datasources.html), most configured for specific cloud providers.  We will be using the "NoCloud" datasource, which as mentioned is designed for using Cloud Init without a cloud provider.

NoCloud allows configuration information to be included a number of ways: as key/value pairs in kernel parameters, using a CD (or virtual CD, in the case of virtual machines) mounted at startup, a file included on the filesystem, or, in the case we will use today, via HTTP from a URL we provide (the "NoCloud Net" option).

The datasource configuration itself can be provided via the kernel parameter or by setting it in the cloud-init configuration file, `/etc/cloud/cloud.cfg`. The configuration file works very well for setting up cloud-init with customized disk images, or for testing on existing hosts.

Cloud Init will also merge configuration data from any `*.cfg` files found in `/etc/cloud/cloud.cfg.d/`, so to keep things cleaner, we will configure the datasource in `/etc/cloud/cloud.cfg.d/10_datasource.cfg`. Cloud Init can be told to read from an HTTP data source with the `seedfrom` key, using the syntax:

`seedfrom: http://ip_address:port/`

The IP address and port are those of the web service we will create further down.  In my case, I used the IP of my laptop, and port 8080. This can also be a DNS name.

Create the `/etc/cloud/cloud.cfg.d/10_datasource.cfg` file:

```txt
# Add the datasource:
# /etc/cloud/cloud.cfg.d/10_datasource.cfg

# NOTE THE TRAILING SLASH HERE!
datasource:
  NoCloud:
    seedfrom: http://ip_address:port/
```

That's it for the client setup.  Now, when it is rebooted, the client will attempt to retrieve configuration data from the URL you entered in the `seedfrom` key and make any configuration changes that might be necessary.

Next, we need to setup a web server to listen for client requests so we can figure out what needs to be served.

## Setup a webserver to investigate client requests

We can create and run a webserver quickly with [Podman](https://podman.io/) or other container orchestration tools (like Docker or Kubernetes). This example will use Podman, but the same commands can be used with Docker.

To get started, we will use the `Fedora:31` container image, and create a Containerfile (for Docker this would be a Dockerfile) that installs and configures Nginx. From that Containerfile we can build a custom image and run it on the host we want to act as the cloud-init service.

Create a Containerfile with the following contents:

```txt
FROM fedora:31

ENV NGINX_CONF_DIR "/etc/nginx/default.d"
ENV NGINX_LOG_DIR "/var/log/nginx"
ENV NGINX_CONF "/etc/nginx/nginx.conf"
ENV WWW_DIR "/usr/share/nginx/html"

# Install Nginx and clear the yum cache
RUN dnf install -y nginx \
      && dnf clean all \
      && rm -rf /var/cache/yum

# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout ${NGINX_LOG_DIR}/access.log \
    && ln -sf /dev/stderr ${NGINX_LOG_DIR}/error.log

# Listen on port 8080, so root privileges are not required for podman
RUN sed -i -E 's/(listen)([[:space:]]*)(\[\:\:\]\:)?80;$/\1\2\38080 default_server;/' $NGINX_CONF
EXPOSE 8080

# Allow Nginx PID to be managed by non-root user
RUN sed -i '/user nginx;/d' $NGINX_CONF
RUN sed -i 's/pid \/run\/nginx.pid;/pid \/tmp\/nginx.pid;/' $NGINX_CONF

# Run as an unprivileged user
USER 1001

CMD ["nginx", "-g", "daemon off;"]
```

The most important part of the Containerfile above is the section changing how the logs are stored (writing to STDOUT rather than a file), so we can see requests coming into the server in the container logs. A few other changes are included so we can run the container with Podman without root privileges, and the processes in the container can run without root as well.

This first pass at the webserver does not serve any cloud-init data; we will just use this to see what the cloud-init client is requesting from it.

With the Containerfile created, use Podman to build and run a webserver image:

```shell
# Build the container image
$ podman build -f Containerfile -t cloud-init:01 .

# Create a container from the new image, and run it
# It will listen on port 8080
$ podman run --rm -p 8080:8080 -it cloud-init:01
```

This will run the container, leaving your terminal attached and with a pseudo-TTY. It will appear that nothing is happening at first, but requests to port 8080 of the host machine will be routed to the Nginx server inside the container, and a log message will appear in the terminal window. This can be tested with `curl` from the host machine:

```shell
# Use curl to send an HTTP request to the Nginx container
$ curl http://localhost:8080
```

After running that curl command, you should see a log message similar to this appear in the terminal window:

```txt
127.0.0.1 - - [09/May/2020:19:25:10 +0000] "GET / HTTP/1.1" 200 5564 "-" "curl/7.66.0" "-"
```

Now comes the fun part: reboot the cloud-init client and watch the Nginx logs to see what cloud-init requests from the webserver when the client boots up!

As the client finishes its boot process, you should see log messages similar to the following:

```txt
2020/05/09 22:44:28 [error] 2#0: *4 open() "/usr/share/nginx/html/meta-data" failed (2: No such file or directory), client: 127.0.0.1, server: _, request: "GET /meta-data HTTP/1.1", host: "instance-data:8080"
127.0.0.1 - - [09/May/2020:22:44:28 +0000] "GET /meta-data HTTP/1.1" 404 3650 "-" "Cloud-Init/17.1" "-"
```

Note: use CTRL-C to stop the running container.

You can see the request is for the `/meta-data` path - ie: `http://ip_address_of_the_webserver:8080/meta-data`. This is just a GET request - cloud-init is not POSTing (sending) any data to the webserver. It is just blindly requesting the files from the datasource URL, so it is up to the datasource to identify what host is asking.  In this simple example, we are just sending generic data to any client, but a larger homelab will need a more sophisticated service.

This is cloud-init requesting the [instance meta-data](https://cloudinit.readthedocs.io/en/latest/topics/instancedata.html#what-is-instance-data) information. This file can include a lot of information about the instance itself, such as the instance id, the hostname to assign the instance, the cloud id - even networking information.

We will create a basic meta-data file with an instance id, and hostname for the host, and try serving that to the cloud-init client.

First, create a meta-data file that can be copied into the container image:

```txt
instance-id: iid-local01
local-hostname: raspberry
hostname: raspberry
```

The `instance-id` can be anything. However, if you subsequently change the `instance-id` after cloud-init runs and the file is served to the client, it will trigger cloud-init to re-run.  You can use this mechanism to update instance configuration if you like, but be aware that it works that way.

The `local-hostname` and `hostname` keys are just that; they set the hostname information for the client when cloud-init runs.

Add the following line to the Containerfile to copy the meta-data file into the new image:

```txt
# Copy the meta-data file into the image for Nginx to serve it
COPY meta-data ${WWW_DIR}/meta-data
```

Now rebuild the image (use a new tag for easy troubleshooting) with the meta-data file, and create and run a new container with Podman:

```shell
# Build a new image named cloud-init:02
podman build -f Containerfile -t cloud-init:02 .

# Run a new container with this new meta-data file
podman run --rm -p 8080:8080 -it cloud-init:02
```

With the new container running, reboot your cloud-init client and watch the Nginx logs again:

```txt
127.0.0.1 - - [09/May/2020:22:54:32 +0000] "GET /meta-data HTTP/1.1" 200 63 "-" "Cloud-Init/17.1" "-"
2020/05/09 22:54:32 [error] 2#0: *2 open() "/usr/share/nginx/html/user-data" failed (2: No such file or directory), client: 127.0.0.1, server: _, request: "GET /user-data HTTP/1.1", host: "instance-data:8080"
127.0.0.1 - - [09/May/2020:22:54:32 +0000] "GET /user-data HTTP/1.1" 404 3650 "-" "Cloud-Init/17.1" "-"
```

You see this time the `/meta-data` path was served to the client.  Success!

However, the client is looking for a second file, at the `/user-data` path. This file contains configuration data provided by the instance owner, as opposed to data from the cloud provider. For a homelab, both of these are you.

There are a [large number of user-data modules](https://cloudinit.readthedocs.io/en/latest/topics/modules.html) you can use to configure your instance.  For this example, we will just use the `write_files` module to create some test files on the client and verify that cloud-init is working.

Create a user-data file with the following content:

```yaml
#cloud-config

# Create two files with example content using the write_files module
write_files:
 - content: |
     "Does cloud-init work?"
   owner: root:root
   permissions: '0644'
   path: /srv/foo
 - content: |
    "IT SURE DOES!"
   owner: root:root
   permissions: '0644'
   path: /srv/bar
```

In addition to a YAML file using the user-data modules provided by cloud init, you could also make this an executable script to be run by cloud-init.

After creating the user-data file, add the following line to the Containerfile to copy it into the image when the image is rebuilt:

```txt
# Copy the user-data file into the container image
COPY user-data ${WWW_DIR}/user-data
```

Rebuild the image and create and run a new container, this time with the user-data information:

```shell
# Build a new image named cloud-init:03
podman build -f Containerfile -t cloud-init:03 .

# Run a new container with this new user-data file
podman run --rm -p 8080:8080 -it cloud-init:03
```

Now, reboot your cloud-init client, and watch the Nginx logs on the webserver:

```txt
127.0.0.1 - - [09/May/2020:23:01:51 +0000] "GET /meta-data HTTP/1.1" 200 63 "-" "Cloud-Init/17.1" "-"
127.0.0.1 - - [09/May/2020:23:01:51 +0000] "GET /user-data HTTP/1.1" 200 298 "-" "Cloud-Init/17.1" "-
```

Success! This time both the meta-data and user-data files were served to the cloud-init client.

## Validate that cloud-init ran

From the logs above, we know that cloud-init ran on the client host, and requested the meta-data and user-data files, but did it do anything with them?  We can validate that cloud-init wrote the files we added in the user-data file, in the "write_files" section.

On your cloud-init client, check the contents of the "/srv/foo" and "/srv/bar" files:

```shell
# cd /srv/ && ls
bar foo
# cat foo
"Does cloud-init work?"
# cat bar
"IT SURE DOES!"
```

Success! The files were written and have the content we expected.

As mentioned above, there are plenty of other modules that can be used to configure the host.  For example, the user-data file can be configured to add packages with `apt`, copy SSH authorized_keys, create users and groups, configure and run configuration management tools, and many other things. In practice, for my own Private Cloud at Home, I use it to copy my authorized_keys, create a local user and group, and setup sudo permissions.

## Conclusion

In this article, we learned what Cloud Init is and why it would be useful in a homelab, especially a lab focusing on cloud technologies. You also configured a Cloud Init client, and used a webserver to investigate what the client requested from the Cloud Init datasource.  Finally, we added meta-data and user-data to the webserver to create a simple Cloud Init datasource.

This simple service may not be super useful for homelab, but this exercise showed how Cloud Init works. With this info, you can move on to creating a dynamic service that can configure each host with custom data, making a private cloud at home more similar to the cloud services provided by the major cloud providers.

With a slightly more complicated datasource, adding new physical (or virtual) machines to your private cloud at home can be as simple as plugging them in and turning them on.
