## Introduction

This document describes the configuration process required to enable Infinibox
to be used as an iSCSI or Fibre Channel Cinder Block Storage backend in
Red Hat OpenStack distributions.

Before using this document, please make sure:

-  Your Red Hat OpenStack Platform Overcloud has been correctly
   deployed through the Undercloud Director

-  Your Infinibox should be available in the cloud management
   network or routed to the cloud management network with the Infinibox iSCSI
   ports correctly configured. On the Infinibox you must
   also create a storage pool that will be used as the cinder backend

-  The Infinibox management IP (and iSCSI port IPs if applicable) must have
   connectivity from the controller and compute nodes

When Red Hat OpenSstack Platform is deployed through the Director, all
major Overcloud settings must be defined and orchestrated through the
Director as well. This will ensure that the settings persist through any
Overcloud updates.

## Configure Infinibox as a Cinder Backend

Choose the correct subdirectory base on your Red Hat OpenStack Platform Overcloud's version.
Copy the YAML file from subdirectory into the following locations in your Undercloud:

`cinder-infinidat-config.yaml` into `~stack/templates/`

Use the `Dockerfile` in the subdirectory to create an Infinidat specific Cinder Volume container image:
```
$ docker build . -t "openstack-cinder-volume-infinidat:latest"
```
This newly created image can then be pushed to a registry that has been configured
as the sources of images to be used by the RHOSP deployment.

Red Hat Certified versions of these containers can also be used. These can be found
in the Red Hat Container Catalog. See https://access.redhat.com/containers/#/search/infinidat

Edit the overcloud container images environment file (usually
`overcloud_images.yaml`, created when using the
`openstack overcloud container image prepare` command) and change the
appropriate parameter to use the custom container image.

Edit `~/templates/cinder-infinidat-config.yaml` and populate it with your specific
Infinibox data. Specifically, you must replace `infinidat-openstack-cert` with your infinibox's
customized data in the form of `infinibox-{pool name}`. For example if you pool name is `cinder-pool`,
replace `infinidat-openstack-cert` with `infinibox-cinder-pool`.

## Deploying the Configured Backend

To deploy the backend configured above, first, log in as the
stack user to the Undercloud. Then deploy the backend (defined in the
edited `~/templates/cinder-infinidat-config.yaml`) by running the
`openstack overcloud deploy` with the required switches for your
deployment version together with an additonal templates file defined
by `–e ~/templates/cinder-infinidat-config.yaml`

If you passed any extra environment files when you created the Overcloud
you must pass them again here using the –e option to avoid making
undesired changes to the Overcloud.

## Test the Configured Backend

After deploying the backend, test whether you can successfully create
volumes on it. Doing so will require loading the necessary environment
variables first. These variables are defined in `/home/stack/overcloudrc`
by default.

To load these variables, run the following command as the stack user:
```
$ source /home/stack/overcloudrc
```
You should now be logged into the Controller node. From there you can
create a *volume type*, which can be used to specify the back end you
want to use (in this case the newly-defined backend). This is required
in an OpenStack deployment where you have other backends enabled.

To create a volume type with description, run:
```
$ cinder type-create --description Infinidat-iscsi infinibox-cinder-pool
```
Now, associate that new volume type with one of the pre-defined back-ends. We can set or unset extra_spec for a volume type. i.e:
```
$ cinder type-key infinibox-cinder-pool set volume_backend_name=infinidat-openstack-cert
```
You should now be able to create a 2GB volume on your newly defined
backend by invoking its volume type. To do this run:
```
$ cinder create –volume-type infinibox-cinder-pool -display-name cinder-volume 2
```

