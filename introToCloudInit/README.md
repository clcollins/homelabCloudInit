# Introduction to Cloud Init for your Homelab

[Cloud-init](https://cloudinit.readthedocs.io/) is a standard - it would not be a stretch to say *the* standard - used by cloud providers to provide initialization and configuration data to cloud instances. It is most often used on the first boot of a new instance to automate network setup, account creation and ssh key installation: anything required to bring a new system online so it is accessible by the user.

In a previous article, [Modify a disk image to create a Raspberry Pi-based homelab](LINK TO THIS ARTICLE), I showed how to customize the operating system image for single board computers like the Raspberry Pi to accomplish a similar goal.  With Cloud Init, there is no need to add custom data to the image. Once it is enabled in your images, your virtual machines, physical servers, even tiny Raspberry Pis, can all behave like cloud instances in your own Private Cloud at Home. New machines can just be plugged in and turned on to automatically become part of your homelab.

To be completely honest, Cloud Init is not designed with homelabs in mind. As just mentioned, you can easily modify the disk image for a given set of systems to enable SSH access and configure them after first boot. Cloud Init is designed for large-scale cloud providers who need to accommodate many customers, maintain a small set of images, and provide a mechanism for the customers to access instances without customizing an image for each of them. A homelab with a single administrator does not face the same challenges.

Cloud Init is not without merit in the homelab though. One of the goals of our Private Cloud at Home project is education, and setting up Cloud Init for your homelab is a great way to gain experience with a technology used heavily by cloud providers, large and small. Cloud Init is an alternative to other initial configuration options, too. Rather than customizing each image, ISO, et cetera for every device in your homelab and face tedious updates when you want to make changes, you can just enable Cloud Init. This reduces technical debt - and is there anything worse than *personal* technical debt? Finally, using Cloud Init in your homelab allows your private cloud instances to behave the same as any public cloud instances you have or may have in the future - a true [hybrid-cloud](https://www.redhat.com/en/topics/cloud-computing/what-is-hybrid-cloud).

## An Overview of Cloud Init

When an instance configured for Cloud Init boots up and the Cloud Init service (actually, four services in systemd implementations, to handle dependencies during the boot process) starts, it checks its configuration for a [DataSource](https://cloudinit.readthedocs.io/en/latest/topics/datasources.html) to determine what type of cloud it is running in. Each of the major cloud providers has a datasource configuration, telling the instance where and how to retrieve configuration information. The instance then uses the datasource information to retrieve configuration information provided by the cloud provider, such as networking information and instance identification information, and configuration data provided by the customer, such as authorized keys to be copied, user accounts to be created, and many other possible tasks.

After retrieving the data, Cloud Init then configures the instance: setting up networking, copying the authorized keys, et cetera, and finally completes the boot process. It is at that point accessible to the remote user, ready for further configuration with tools like [Ansible](https://www.ansible.com/) or [Puppet](https://puppet.com/), or to receive a workload and begin its assigned tasks.

## Configuration Data

As mentioned above, the configuration data used by Cloud Init comes from two potential sources, the cloud provider and the instance user. In the case of your homelab, you fill both roles: providing networking and instance information as the cloud provider, and configuration information as the user.

### Cloud Provider meta-data file

In your cloud provider role, your homelab datasource will offer your private cloud instances a `meta-data` file. The [meta-data](https://cloudinit.readthedocs.io/en/latest/topics/instancedata.html#) file can contain information such as the instance-id, cloud type, or python version (Cloud Init is written in and uses Python), or a public SSH key to be assigned to the host.  The meta-data file may also contain networking information if not using DHCP (or the other mechanisms Cloud Init supports such as a config file in the image, or kernel parameters).

### User-provided user-data file

The real meat of Cloud Init's value is in the `user-data` file.  Provided by the user to the cloud provider and included in the datasource, the [user-data](https://cloudinit.readthedocs.io/en/latest/topics/format.html) file is what turns an instance from a generic machine into a member of the user's fleet.  The user-data file can come in the form of an executable script, working the same as the script would in normal circumstances, or as a [cloud config](https://cloudinit.readthedocs.io/en/latest/topics/examples.html) YAML file, which makes use of [Cloud Init's modules](https://cloudinit.readthedocs.io/en/latest/topics/modules.html) to perform configuration tasks.

## DataSource

The datasource is a service provided by the cloud provider that offers the meta-data and user-data files to the instances. Instance images or ISOs are configured to tell the instance what datasource is being used.

In the case of Amazon AWS, a [link-local](https://en.wikipedia.org/wiki/Link-local_address) is provided that will respond to HTTP requests from an instance with the instance's custom data. Other cloud providers have their own mechanisms as well.  Luckily for our Private Cloud at Home project, there are also "NoCloud" data sources.

The [NoCloud](https://cloudinit.readthedocs.io/en/latest/topics/datasources/nocloud.html) datasources allow configuration information to be provided via the kernel command like as key/value pairs, or as user-data and meta-data files provided as mounted ISO filesystems. These are useful for virtual machines, especially paired with automation to create the virtual machines themselves.

There is also a `NoCloudNet` datasource that behaves similar to the AWS EC2 datasource, providing an IP or DNS name from which to retrieve user-data and meta-data via HTTP.  This is most helpful for the physical machines in your homelab such as Raspberry Pis, NUCs or surplus server equipment.  While NoCloud could work too, it requires more manual attention; an anti-pattern for cloud instances.

## Cloud Init for the homelab

I hope this has given you an idea of what Cloud Init is, and how it may be helpful used in your homelab. It is an incredible tool already embraced by major cloud providers, and using it at home can be both educational and fun, and help to automate the addition of new physical or virtual servers to your lab. Future articles will detail how to create both simple static and more complex dynamic Cloud Init services and guide you in incorporating it into you Private Cloud at Home.
