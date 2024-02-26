FROM registry.redhat.io/rhosp-rhel8/openstack-cinder-volume:16.2.4-10

MAINTAINER Infinidat Host Team <kfred@infinidat.com>

# Required Labels
LABEL name="rhosp16/openstack-cinder-volume-infinidat-plugin" \
      maintainer="kfred@infinidat.com" \
      vendor="Infinidat" \
      version="1.0" \
      release="15" \
      summary="Red Hat OpenStack Platform 16.2 cinder-volume Infinidat Plugin" \
      description="Red Hat OpenStack Platform 16.2 cinder-volume Infinidat Plugin"

USER root

# Installing driver dependencies
RUN dnf copr enable -y deiter/infinisdk && \
    dnf install -y python3-infinisdk

# These RPMs (or slightly updated versions) will ship with our next OSP-16.2.5 maintenance release.
COPY openstack-cinder-15.6.1-2.20230117204940.9f9194d.el8osttrunk.noarch.rpm \
     python3-cinder-15.6.1-2.20230117204940.9f9194d.el8osttrunk.noarch.rpm .

# Install new cinder RPMs
RUN dnf localinstall -y \
        openstack-cinder-15.6.1-2.20230117204940.9f9194d.el8osttrunk.noarch.rpm \
        python3-cinder-15.6.1-2.20230117204940.9f9194d.el8osttrunk.noarch.rpm && \
    rm -vf openstack-cinder-15.6.1-2.20230117204940.9f9194d.el8osttrunk.noarch.rpm \
           python3-cinder-15.6.1-2.20230117204940.9f9194d.el8osttrunk.noarch.rpm && \
    dnf clean all

# Add required license as text file in Liceses directory (GPL, MIT, APACHE, Partner End User Agreement, etc)
RUN mkdir /licenses
COPY licensing.txt /licenses

USER cinder
