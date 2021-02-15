# rhos16.1 build
This is a Readme for RHOSP version 16.1 which describes how to build an image for deploying the Infinidat-Cinder-Volume plugin driver during an overcloud deployment.
1. Download the pacakage python3-mock-3.0.5-1.el8ost.noarch.rpm and save it in the same directory where Dockerfile is.
2. Next is to build the image, tag it and upload to undercloud for deployment.
	2.1 Download the repository from https://github.com/Infinidat/tripleo-deployment-configs/tree/master/RHOSP16.1
	2.2 Change location into the downloaded directory.
	2.3 Build the image using the command:
	'podman build --tag "openstack-cinder-volume-infinidat:16.1" --file Dockerfile .'
	2.4 Push the image into container registry which is used for overcloud deployment, this is done using tripleo command:
	'sudo openstack tripleo container image push --local undercloud.ctlplane.localdomain:8787/library/openstack-cinder-volume-infinidat:16.1'
	2.5 Verify that the new image was uploaded using 'sudo podman images' or 'sudo openstack tripleo container image list'
3. Deployment: The deployment of overcloud should be done with your environment files. 
	In order to include the openstack-cinder-volume-infinidat image instead of the default cinder-volume from Redhat registry, one needs to edit the containers-prepare-parameter.yaml file.
	3.1 Under push_destination: add an exclude for cinder-volume and another section of includes: with a name_suffix: -infinidat. See an example.
	3.2 Include the containers-prepare-parameter.yaml you have edited in your deployment environment files.
	3.3 Add the file cinder-infinidat-config.yaml to your deployment, this file will have the parameters of your Infinibox Storage Array.
4. Testing
	4.1 One should test if the deployment of openstack-cinder-volume container was successful.
	4.2 Source the rc file created during deployment, and create a new volume type. See instructions in https://github.com/Infinidat/tripleo-deployment-configs/blob/master/README.md 
	4.3 Create a new volume using the newly created volume type. 
