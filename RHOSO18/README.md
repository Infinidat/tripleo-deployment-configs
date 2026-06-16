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
- Fibre Channel or iSCSI connectivity between the OpenShift nodes running `cinder-volume` and InfiniBox storage.
- EDPM compute nodes running Nova compute services for VM workloads.

This document provides parallel configuration examples for both supported storage protocols:

- `infinidat-fc` for Fibre Channel back end configuration.
- `infinidat-iscsi` for iSCSI back end configuration.

For Fibre Channel deployments, ensure that zoning is completed between:

- OpenShift nodes that host `cinder-volume` pods and the InfiniBox FC storage ports.
- EDPM compute nodes and the InfiniBox FC storage ports.

For iSCSI deployments, ensure that network connectivity is available between:

- OpenShift nodes that host `cinder-volume` pods and the InfiniBox iSCSI network spaces.
- EDPM compute nodes and the InfiniBox iSCSI network spaces.

## Prerequisites

- A working RHOSO 18 deployment.
- An OpenShift cluster with worker nodes suitable for running RHOSO storage services.
- EDPM compute nodes deployed and registered with Nova.
- InfiniBox storage systems reachable from the RHOSO environment.
- Cinder volume types planned for the Infinidat back ends.
- A certified or certification-candidate Infinidat `cinder-volume` container image for RHOSO 18.
- Access to the registry that stores the Infinidat `cinder-volume` image.
- An OpenShift user with permissions to update RHOSO custom resources and, if multipath is required on OpenShift nodes, to create `MachineConfig` resources.

For Fibre Channel deployments:

- Fibre Channel zoning must be completed between OpenShift nodes, EDPM compute nodes, and InfiniBox FC storage ports.
- The required Fibre Channel host packages and services must be available on EDPM compute nodes.
- `multipathd` must be configured for Infinidat devices where multipath is required.

For iSCSI deployments:

- iSCSI network connectivity must be available between OpenShift nodes, EDPM compute nodes, and InfiniBox iSCSI network spaces.
- The required iSCSI host packages and services must be available on EDPM compute nodes.
- `multipathd` must be configured for Infinidat devices where multipath is required.

## Infinidat Cinder volume image

The Infinidat Cinder volume image must be built from the corresponding Red Hat RHOSO 18 `openstack-cinder-volume` image. The Infinidat image tag should match the RHOSO base image version that it is built from.

Example image name:

```text
registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
```

The Infinidat Cinder container image contains the additional Python packages required by the Infinidat driver.

The package list depends on the image build, but commonly includes the Infinidat SDK and its dependencies, for example:

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
      infinidat-fc: registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
      infinidat-iscsi: registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
```

Apply the custom resource or patch the existing one:

```bash
oc apply -f openstackversion-infinidat.yaml
```

Verify that the `cinder-volume` pods use the Infinidat image:

```bash
for pod in cinder-volume-infinidat-fc-0 cinder-volume-infinidat-iscsi-0; do
  echo "===== ${pod} ====="
  oc get pod -n openstack "${pod}" \
    -o jsonpath='{range .spec.containers[*]}{.name}{" => "}{.image}{"\n"}{end}'
done
```

Expected output:

```text
===== cinder-volume-infinidat-fc-0 =====
cinder-volume => registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
probe => registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
===== cinder-volume-infinidat-iscsi-0 =====
cinder-volume => registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
probe => registry.example.com/openstack-cinder-volume-infinidat-plugin:18.0.0
```

## Create Secret files

Each Infinidat Cinder back end requires a unique Kubernetes Secret that contains the storage credentials for that backend.

The examples below create separate Secrets for the Fibre Channel and iSCSI back ends:

- `cinder-volume-infinidat-fc-secrets`
- `cinder-volume-infinidat-iscsi-secrets`

### Fibre Channel Secret

Create a file named `cinder-volume-infinidat-fc-secrets.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cinder-volume-infinidat-fc-secrets
  namespace: openstack
  labels:
    service: cinder
    component: cinder-volume
type: Opaque
stringData:
  cinder-volume-infinidat-fc-secrets: |
    [infinidat-fc]
    san_ip = <infinibox-management-address-for-fc-backend>
    san_login = <infinibox-username>
    san_password = <infinibox-password>
```

Apply the Secret:

```bash
oc apply -f cinder-volume-infinidat-fc-secrets.yaml
```

### iSCSI Secret

Create a file named `cinder-volume-infinidat-iscsi-secrets.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cinder-volume-infinidat-iscsi-secrets
  namespace: openstack
  labels:
    service: cinder
    component: cinder-volume
type: Opaque
stringData:
  cinder-volume-infinidat-iscsi-secrets: |
    [infinidat-iscsi]
    san_ip = <infinibox-management-address-for-iscsi-backend>
    san_login = <infinibox-username>
    san_password = <infinibox-password>
```

Apply the Secret:

```bash
oc apply -f cinder-volume-infinidat-iscsi-secrets.yaml
```

Verify that the Secrets were created:

```bash
oc get secret -n openstack cinder-volume-infinidat-fc-secrets
oc get secret -n openstack cinder-volume-infinidat-iscsi-secrets
```

## Configure Cinder back ends

Open your `OpenStackControlPlane` CR file, for example `openstack_control_plane-infinidat.yaml`, and add the Infinidat Cinder volume backend configuration.

Cinder back ends are configured in the `OpenStackControlPlane` custom resource under `spec.cinder.template.cinderVolumes`.

The examples below show complete backend configuration for both Fibre Channel and iSCSI. Replace pool names, network attachments, and other values with the values for your environment.

### Example 1: Fibre Channel backend configuration

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
        infinidat-fc:
          replicas: 1
          networkAttachments:
            - storage
            - storageMgmt
          customServiceConfigSecrets:
            - cinder-volume-infinidat-fc-secrets
          customServiceConfig: |
            [DEFAULT]
            enabled_backends = infinidat-fc

            [infinidat-fc]
            volume_backend_name = infinidat-fc
            volume_driver = cinder.volume.drivers.infinidat.InfiniboxVolumeDriver
            infinidat_pool_name = <fc-pool-name>
            infinidat_storage_protocol = FC
            san_thin_provision = true
            driver_use_ssl = true
            suppress_requests_ssl_warnings = true
            use_multipath_for_image_xfer = true
            enforce_multipath_for_image_xfer = false
```

### Example 2: iSCSI backend configuration

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
        infinidat-iscsi:
          replicas: 1
          networkAttachments:
            - storage
            - storageMgmt
          customServiceConfigSecrets:
            - cinder-volume-infinidat-iscsi-secrets
          customServiceConfig: |
            [DEFAULT]
            enabled_backends = infinidat-iscsi

            [infinidat-iscsi]
            volume_backend_name = infinidat-iscsi
            volume_driver = cinder.volume.drivers.infinidat.InfiniboxVolumeDriver
            infinidat_pool_name = <iscsi-pool-name>
            infinidat_storage_protocol = iSCSI
            san_thin_provision = true
            driver_use_ssl = true
            suppress_requests_ssl_warnings = true
            use_multipath_for_image_xfer = true
            enforce_multipath_for_image_xfer = false
```

Apply the updated `OpenStackControlPlane` configuration:

```bash
oc apply -f openstack_control_plane-infinidat.yaml
```

Wait for the Cinder resources to reconcile:

```bash
oc get cinder -n openstack
oc get cindervolume -n openstack
oc get pods -n openstack | grep cinder-volume
```

## Configure EDPM compute nodes

Nova compute services run on EDPM data plane nodes, not as part of the OpenShift control plane.

Ensure that each EDPM compute node has:

- Fibre Channel zoning to InfiniBox storage ports, for FC deployments.
- Network connectivity to InfiniBox iSCSI network spaces, for iSCSI deployments.
- A working `multipath` configuration for Infinidat devices.
- The required host packages and services for the selected volume attachment protocol.
- Connectivity to the RHOSO control-plane endpoints required by Nova and Cinder.

Verify the Nova compute service:

```bash
openstack compute service list --service nova-compute
```

The service state must be `up`.

## Create volume types

Create Cinder volume types for the Infinidat back ends and map each type to the corresponding `volume_backend_name`.

### Fibre Channel volume type

```bash
openstack volume type create infinidat-fc
openstack volume type set --property volume_backend_name=infinidat-fc infinidat-fc
```

### iSCSI volume type

```bash
openstack volume type create infinidat-iscsi
openstack volume type set --property volume_backend_name=infinidat-iscsi infinidat-iscsi
```

Verify the volume types:

```bash
openstack volume type list
openstack volume type show infinidat-fc
openstack volume type show infinidat-iscsi
```

## Verify Cinder services and pools

Verify that Cinder services are running:

```bash
openstack volume service list
```

Expected output includes `cinder-volume` services for each Infinidat backend:

```text
cinder-volume    cinder-volume-infinidat-fc-0@infinidat-fc          nova    enabled    up
cinder-volume    cinder-volume-infinidat-iscsi-0@infinidat-iscsi    nova    enabled    up
```

Verify the backend pools:

```bash
openstack volume backend pool list --long
```

Expected output includes pools similar to:

```text
cinder-volume-infinidat-fc-0@infinidat-fc#infinidat-fc          FC
cinder-volume-infinidat-iscsi-0@infinidat-iscsi#infinidat-iscsi iSCSI
```

## Create and verify volumes

Create a test volume on each Infinidat backend to verify that the volume types and backend mappings work correctly.

### Create a Fibre Channel volume

```bash
openstack volume create \
  --size 1 \
  --type infinidat-fc \
  infinidat-fc-test-volume
```

Verify that the volume was created successfully:

```bash
openstack volume show infinidat-fc-test-volume \
  -c id \
  -c name \
  -c status \
  -c type \
  -c size
```

Expected result:

```text
status = available
type   = infinidat-fc
```

### Create an iSCSI volume

```bash
openstack volume create \
  --size 1 \
  --type infinidat-iscsi \
  infinidat-iscsi-test-volume
```

Verify that the volume was created successfully:

```bash
openstack volume show infinidat-iscsi-test-volume \
  -c id \
  -c name \
  -c status \
  -c type \
  -c size
```

Expected result:

```text
status = available
type   = infinidat-iscsi
```

List both test volumes:

```bash
openstack volume list --name infinidat
```

Optional cleanup:

```bash
openstack volume delete infinidat-fc-test-volume
openstack volume delete infinidat-iscsi-test-volume
```
