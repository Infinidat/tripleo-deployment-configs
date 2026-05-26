# Infinidat InfiniBox Storage Installation Guide for Red Hat OpenStack Services on OpenShift 18

## Documentation

- [Red Hat OpenStack Services on OpenShift 18.0 - Integrating partner content](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html-single/integrating_partner_content/index)
- [Red Hat OpenStack Services on OpenShift 18.0 - Configuring persistent storage](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html-single/configuring_persistent_storage/index)
- [INFINIDAT InfiniBox Block Storage Driver](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/infinidat-volume-driver.html)

## Overview

This document describes how to integrate and configure the Infinidat InfiniBox Cinder volume driver container image in a Red Hat OpenStack Services on OpenShift 18.0 deployment.

RHOSO 18 is different from director-based Red Hat OpenStack Platform deployments. RHOSP 15, 16, and 17 used TripleO environment files and Heat parameters to customize the `cinder-volume` container image and configure Cinder back ends. RHOSO 18 uses OpenShift custom resources instead:

- `OpenStackVersion` is used to override service container images, including per-backend `cinder-volume` images.
- `OpenStackControlPlane` is used to configure the Cinder service and its back ends.
- `OpenStackDataPlaneNodeSet` and `OpenStackDataPlaneDeployment` are used for EDPM compute nodes.
- OpenShift `MachineConfig` is used to configure node-level services such as `multipathd` on OpenShift nodes that host Cinder services.

The Infinidat Cinder volume image is based on the Red Hat RHOSO 18 `openstack-cinder-volume` image and adds the Python dependencies required by the Infinidat Cinder driver.

## Supported deployment model

This guide covers an RHOSO 18 deployment with:

- RHOSO 18 control plane deployed on Red Hat OpenShift Container Platform.
- Cinder API, scheduler, backup, and volume services running as OpenShift workloads.
- One or more Infinidat Cinder volume back ends.
- Fibre Channel connectivity between the OpenShift nodes running `cinder-volume` and InfiniBox storage.
- EDPM compute nodes running Nova compute services for VM workloads.

The same deployment model can be extended to iSCSI where supported by the driver and by the environment, but the examples in this document focus on Fibre Channel.

## Prerequisites

- A working RHOSO 18 deployment.
- An OpenShift cluster with worker nodes suitable for running RHOSO storage services.
- EDPM compute nodes deployed and registered with Nova.
- InfiniBox storage systems reachable from the RHOSO environment.
- Fibre Channel zoning completed between:
  - OpenShift nodes that host `cinder-volume` pods and the InfiniBox storage ports.
  - EDPM compute nodes and the InfiniBox storage ports.
- Cinder volume types planned for the Infinidat back ends.
- A certified or certification-candidate Infinidat `cinder-volume` container image for RHOSO 18.
- Access to the registry that stores the Infinidat `cinder-volume` image.
- An OpenShift user with permissions to update RHOSO custom resources and, if multipath is required on OpenShift nodes, to create `MachineConfig` resources.

## Infinidat Cinder volume image

The Infinidat Cinder volume image must be built from the corresponding Red Hat RHOSO 18 `openstack-cinder-volume` image. The Infinidat image tag should match the RHOSO base image version that it is built from.

Example image name:

```text
registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
```

The Infinidat Cinder container image contains the additional Python packages required by the Infinidat driver. The package list depends on the image build, but commonly includes the Infinidat SDK and its dependencies, for example:

- `python3-infinisdk`
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
- `python3-logbook`
- `python3-mitba`
- `python3-munch`
- `python3-pact`
- `python3-sentinels`
- `python3-storage-interfaces`
- `python3-urlobject`
- `python3-vintage`
- `python3-waiting`

## Configure the Infinidat image in OpenStackVersion

RHOSO 18 supports per-backend `cinder-volume` images. Configure the Infinidat image in the `OpenStackVersion` custom resource by mapping each Infinidat backend name to the Infinidat `cinder-volume` image.

Example:

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackVersion
metadata:
  name: openstack
  namespace: openstack
spec:
  customContainerImages:
    cinderVolumeImages:
      infinidat-1: registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
      infinidat-2: registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
```

Apply the custom resource or patch the existing one:

```bash
oc apply -f openstackversion-infinidat.yaml
```

Verify that the `cinder-volume` pods use the Infinidat image:

```bash
for pod in cinder-volume-infinidat-1-0 cinder-volume-infinidat-2-0; do
  echo "===== ${pod} ====="
  oc get pod -n openstack "${pod}" \
    -o jsonpath='{range .spec.containers[*]}{.name}{" => "}{.image}{"\n"}{end}'
done
```

Expected output:

```text
===== cinder-volume-infinidat-1-0 =====
cinder-volume => registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
probe => registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
===== cinder-volume-infinidat-2-0 =====
cinder-volume => registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
probe => registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
```

## Configure Cinder back ends

Cinder back ends are configured in the `OpenStackControlPlane` custom resource under `spec.cinder.template.cinderVolumes`.

The following example configures two Infinidat Fibre Channel back ends. Replace the storage addresses, credentials, pool names, and other values with the values for your environment.

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: openstack
  namespace: openstack
spec:
  cinder:
    template:
      cinderVolumes:
        infinidat-1:
          customServiceConfig: |
            [DEFAULT]
            enabled_backends = infinidat-1

            [infinidat-1]
            volume_backend_name = infinidat-1
            volume_driver = cinder.volume.drivers.infinidat.InfiniboxVolumeDriver
            infinidat_pool_name = <pool-name-1>
            infinidat_storage_protocol = FC
            san_ip = <infinibox-management-address-1>
            san_login = <infinibox-username>
            san_password = <infinibox-password>
            san_thin_provision = true
            driver_use_ssl = true
            suppress_requests_ssl_warnings = true
            use_multipath_for_image_xfer = true
            enforce_multipath_for_image_xfer = false

        infinidat-2:
          customServiceConfig: |
            [DEFAULT]
            enabled_backends = infinidat-2

            [infinidat-2]
            volume_backend_name = infinidat-2
            volume_driver = cinder.volume.drivers.infinidat.InfiniboxVolumeDriver
            infinidat_pool_name = <pool-name-2>
            infinidat_storage_protocol = FC
            san_ip = <infinibox-management-address-2>
            san_login = <infinibox-username>
            san_password = <infinibox-password>
            san_thin_provision = true
            driver_use_ssl = true
            suppress_requests_ssl_warnings = true
            use_multipath_for_image_xfer = true
            enforce_multipath_for_image_xfer = false
```

Do not store production credentials in plain text manifests. Use the secret mechanism supported by your RHOSO deployment process and refer to the secret from the service configuration where applicable.

Apply the updated `OpenStackControlPlane` configuration:

```bash
oc apply -f openstackcontrolplane-infinidat.yaml
```

Wait for the Cinder resources to reconcile:

```bash
oc get cinder -n openstack
oc get cindervolume -n openstack
oc get pods -n openstack | grep cinder-volume
```

## Configure EDPM compute nodes

Nova compute services run on EDPM data plane nodes, not as part of the OpenShift control plane. Ensure that each EDPM compute node has:

- Fibre Channel zoning to InfiniBox storage ports.
- A working `multipath` configuration for Infinidat devices.
- The required host packages and services for FC volume attachment.
- Connectivity to the RHOSO control-plane endpoints required by Nova and Cinder.

Verify the Nova compute service:

```bash
openstack compute service list --service nova-compute
```

The service state must be `up`.

## Create volume types

Create Cinder volume types for the Infinidat back ends and map each type to the corresponding `volume_backend_name`.

```bash
openstack volume type create infinidat-1
openstack volume type set --property volume_backend_name=infinidat-1 infinidat-1

openstack volume type create infinidat-2
openstack volume type set --property volume_backend_name=infinidat-2 infinidat-2
```

Verify the volume types:

```bash
openstack volume type list
```

## Verify Cinder services and pools

Verify that Cinder services are running:

```bash
openstack volume service list
```

Expected output includes `cinder-volume` services for each Infinidat backend:

```text
cinder-volume    cinder-volume-infinidat-1-0@infinidat-1    nova    enabled    up
cinder-volume    cinder-volume-infinidat-2-0@infinidat-2    nova    enabled    up
```

Verify the backend pools:

```bash
openstack volume backend pool list --long
```

Expected output includes pools similar to:

```text
cinder-volume-infinidat-1-0@infinidat-1#infinidat-1    FC
cinder-volume-infinidat-2-0@infinidat-2#infinidat-2    FC
```