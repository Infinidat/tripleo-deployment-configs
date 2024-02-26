# Infinidat InfiniBox Storage Installation Guide for RHOSP 17.1

## Documentation

 - [Installing and Managing Red Hat OpenStack Platform with Director](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/17.1/html-single/installing_and_managing_red_hat_openstack_platform_with_director/index)
 - [Deploying a custom Block Storage Back End](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/17.1/html-single/deploying_a_custom_block_storage_back_end/index)
 - [INFINIDAT InfiniBox Block Storage Driver](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/infinidat-volume-driver.html)

## Overview

This page provides detailed steps on how to enable the containerized Infinidat Cinder driver for Red Hat OpenStack Platform 17.1.
It also contains steps to deploy and configure Infinidat InfiniBox backends for Red Hat OpenStack Platform 17.1.

The Infinidat Cinder container image contains the following additional packages:
- `python3-api-object-schema`
- `python3-arrow`
- `python3-capacity`
- `python3-click`
- `python3-colorama`
- `python3-confetti`
- `python3-flux`
- `python3-gossip`
- `python3-infi-dtypes-iqn`
- `python3-infi-dtypes-wwn`
- `python3-infinisdk`
- `python3-logbook`
- `python3-mitba`
- `python3-munch`
- `python3-pact`
- `python3-sentinels`
- `python3-storage-interfaces`
- `python3-urlobject`
- `python3-vintage`
- `python3-waiting`

## Prerequisites

* Red Hat OpenStack Platform 17.1 with Red Hat Enterprise Linux 9.2.
* Infinidat InfiniBox storage 4.0 or above.

## Steps

### 1. Prepare an environment file for the Infinidat Cinder backend in a cinder-volume container

#### 1.1. Environment file for cinder-volume container

To use Infinidat InfiniBox as a block storage backend, a `cinder-volume` container must be deployed.

##### Procedure

Generate a default environment file that prepares images using your Satellite server as a source.
Refer to the Red Hat OpenStack 17.1 deployment guide, chapter [6.15. Preparing a Satellite server for container images](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/17.1/html-single/installing_and_managing_red_hat_openstack_platform_with_director/index#proc_preparing-a-satellite-server-for-container-images_preparing-for-director-installation), step 9.

Edit the [`containers-prepare-parameter.yaml`](containers-prepare-parameter.yaml) file. Add an exclude parameter to the strategy for the main Red Hat OpenStack Platform 17.1 Cinder container image:

```
parameter_defaults:
  ContainerImagePrepare:
    - push_destination: true
      excludes:
        - cinder-volume
      set:
        namespace: registry.redhat.io/rhosp-rhel9
        name_prefix: openstack-
        name_suffix: ''
        tag: 17.1
      tag_from_label: "{version}-{release}"
```

Refer to the Infinidat's sample [`containers-prepare-parameter.yaml`](containers-prepare-parameter.yaml) file in the [repository](https://github.com/Infinidat/tripleo-deployment-configs).

Add a new strategy to the `ContainerImagePrepare` parameter that includes the replacement container image for the Infinidat InfiniBox Cinder plugin:

```
parameter_defaults:
  ContainerImagePrepare:
  - push_destination: true
    excludes:
    - cinder-volume
    set:
      name_prefix: openstack-
      name_suffix: ''
      namespace: registry.redhat.io/rhosp-rhel9
      rhel_containers: false
      tag: '17.1'
    tag_from_label: '{version}-{release}'
  - push_destination: false
    includes:
    - cinder-volume
    set:
      namespace: registry.connect.redhat.com/infinidat
      name_prefix: openstack-
      name_suffix: -infinidat-plugin-rhosp-17-1
      tag: 'latest'
```
Refer to the Infinidat's sample [`containers-prepare-parameter.yaml`](containers-prepare-parameter.yaml) file in our repository.

Configure authentication for the Red Hat registries in the `ContainerImageRegistryCredentials` parameter. Refer to Installing and managing Red Hat OpenStack Platform with director, chapter [6.7. Obtaining container images from private registries](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/17.1/html-single/installing_and_managing_red_hat_openstack_platform_with_director/index#ref_obtaining-container-images-from-private-registries_preparing-for-director-installation).

Use the `containers-prepare-parameter.yaml` file with all deployment commands, such as `openstack overcloud deploy`:

```
openstack overcloud deploy --templates \
    ...
    -e containers-prepare-parameter.yaml \
    ...
```

When the director deploys the overcloud, the overcloud uses the Infinidat Cinder container image instead of the standard Cinder container image.

#### 1.2. Environment file for the Cinder backend

The Infinidat InfiniBox environment file for Red Hat OpenStack Platform contains settings for each backend you might want to define.

Create the environment file [`cinder-infinidat-config.yaml`](cinder-infinidat-config.yaml) with the following parameters and other backend details:

```
parameter_defaults:
  # Disable all other types of Cinder backends
  CinderEnableIscsiBackend: false
  CinderEnableRbdBackend: false
  CinderEnableNfsBackend: false
  NovaEnableRbdBackend: false

  # Specify defailt volume type for multibackend deployments
  CinderDefaultVolumeType: infinidat-iscsi
  CinderRpcResponseTimeout: 180

  # Enable and configure multipath
  NovaLibvirtVolumeUseMultipath: true
  MultipathdEnable: true
  MultipathdCustomConfigFile: /home/stack/multipath.conf

  # Extra backends configuration
  ControllerExtraConfig:
    cinder::config::cinder_config:
      # iSCSI backend
      # InfiniBox cinder driver
      infinidat-iscsi/volume_driver:
        value: cinder.volume.drivers.infinidat.InfiniboxVolumeDriver
      # Name for the backend.
      infinidat-iscsi/volume_backend_name:
        value: infinidat-iscsi
      # Management address and credentials
      infinidat-iscsi/san_ip:
        value: infinibox
      infinidat-iscsi/san_login:
        value: user
      infinidat-iscsi/san_password:
        value: password
      # Enable SSL support
      infinidat-iscsi/driver_use_ssl:
        value: true
      # Supress SSL warnings
      infinidat-iscsi/suppress_requests_ssl_warnings:
        value: true
      # Configure driver specific options
      infinidat-iscsi/iscsi-pool:
        value: iscsi-pool
      infinidat-iscsi/infinidat_storage_protocol:
        value: iscsi
      infinidat-iscsi/infinidat_iscsi_netspaces:
        value: network-space
      # Enable thin provisionning
      infinidat-iscsi/san_thin_provision:
        value: true
      # Attach volumes using multipath
      infinidat-iscsi/use_multipath_for_image_xfer:
        value: true
      # FC backend
      # InfiniBox cinder driver
      infinidat-fc/volume_driver:
        value: cinder.volume.drivers.infinidat.InfiniboxVolumeDriver
      # Name for the backend.
      infinidat-fc/volume_backend_name:
        value: infinidat-fc
      # Management address and credentials
      infinidat-fc/san_ip:
        value: infinibox
      infinidat-fc/san_login:
        value: user
      infinidat-fc/san_password:
        value: password
      # Enable SSL support
      infinidat-fc/driver_use_ssl:
        value: true
      # Supress SSL warnings
      infinidat-fc/suppress_requests_ssl_warnings:
        value: true
      # Configure driver specific options
      infinidat-fc/infinidat_pool_name:
        value: fc-pool
      infinidat-fc/infinidat_storage_protocol:
        value: fc
      # Enable thin provisionning
      infinidat-fc/san_thin_provision:
        value: true
      # Attach volumes using multipath
      infinidat-fc/use_multipath_for_image_xfer:
        value: true
    # Specify all the backends you want to enable
    cinder_user_enabled_backends:
    - infinidat-iscsi
    - infinidat-fc
```

Refer to the Infinidat's sample [`cinder-infinidat-config.yaml`](cinder-infinidat-config.yaml) file in our repository.

#### Additional help

For further details of the Infinidat InfiniBox storage Cinder driver configuration, refer to OpenStack documentation, chapter [INFINIDAT InfiniBox Block Storage driver](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/infinidat-volume-driver.html).

> Note: Infinidat recommends that you use an Infinidat-specific [`multipath.conf`](multipath.conf) instead of generic one.

> Note: Red Hat OpenStack Platform 17.1 supports configuring only a limited set of options for multipath.conf.    

> Note: For further details, refer to [Red Hat OpenStack Platform 17.1 Configuring persistent storage, chapter 2.12.1.1. Multipath heat template parameters](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/17.1/html-single/configuring_persistent_storage/index#ref_multipath-heat-template-parameters_configuring-cinder).

> Note: Refer to [Red Hat OpenStack Platform 17.1 Configuring persistent storage, chapter 2.12 Multipath configuration](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/17.1/html-single/configuring_persistent_storage/index#con-multipath-configuration_configuring-cinder) to specify and deploy a custom [`multipath.conf`](multipath.conf).

> Note: To view all the options available in the [`multipath.conf`](multipath.conf), refer to [Linux Man Page For multipath.conf(5)](https://manpages.org/multipathconf/5).

### 2. Deploy the overcloud and configured backends

After creating the [`cinder-infinidat-config.yaml`](cinder-infinidat-conf) environment file with appropriate backends, deploy the backend configuration by running the `openstack overcloud deploy` command using the templates option:

```
openstack overcloud deploy --templates \
    ...
    -e /home/stack/cinder-infinidat-config.yaml \
    ...
```

The order of the environment files (.yaml) is important becasue the parameters and resources defined in subsequent environment files take precedence:

```
openstack overcloud deploy --templates \
    ...
    -e /home/stack/cinder-infinidat-config.yaml \
    -e /home/stack/containers-prepare-parameter.yaml \
    ...
    --stack overcloud
```

### 3. Verify the configured changes

3.1. Check the volume services status:
```
(overcloud) [stack@rhosp-cert-director ~]$ openstack volume service list -c Binary -c Host -c Status
+------------------+---------------------------+---------+
| Binary           | Host                      | Status  |
+------------------+---------------------------+---------+
| cinder-scheduler | rhosp-cert-controller     | enabled |
| cinder-backup    | rhosp-cert-controller     | enabled |
| cinder-volume    | hostgroup@infinidat-iscsi | enabled |
| cinder-volume    | hostgroup@infinidat-fc    | enabled |
+------------------+---------------------------+---------+
```

3.2. SSH to the controller node from the undercloud, and check the process for the `cinder-volume` container:
```
[root@rhosp-cert-controller ~]# podman ps | grep cinder-volume
7c326bc69b72 rhosp-cert-director.ctlplane.localdomain:8787/rhosp-rhel9/openstack-cinder-volume-infinidat-plugin-rhosp-17-1:17.1 36 hours ago Up 36 hours (healthy) openstack-cinder-volume-podman-0
```

3.3. Verify that the backend details are visible in `/etc/cinder/cinder.conf` in the `cinder-volume` container:

```
[root@rhosp-cert-controller ~]# podman exec -it openstack-cinder-volume-podman-0 cat /etc/cinder/cinder.conf
...
[infinidat-iscsi]
driver_use_ssl=True
infinidat_iscsi_netspaces=network-space
infinidat_pool_name=iscsi-pool
infinidat_storage_protocol=iscsi
san_ip=infinibox
san_login=user
san_password=password
san_thin_provision=True
suppress_requests_ssl_warnings=True
use_multipath_for_image_xfer=True
volume_backend_name=infinidat-iscsi
volume_driver=cinder.volume.drivers.infinidat.InfiniboxVolumeDriver

[infinidat-fc]
driver_use_ssl=True
infinidat_pool_name=fc-pool
infinidat_storage_protocol=fc
san_ip=infinibox
san_login=user
san_password=password
san_thin_provision=True
suppress_requests_ssl_warnings=True
use_multipath_for_image_xfer=True
volume_backend_name=infinidat-fc
volume_driver=cinder.volume.drivers.infinidat.InfiniboxVolumeDriver
```
