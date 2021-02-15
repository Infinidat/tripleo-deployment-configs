FROM registry.redhat.io/rhosp-rhel8/openstack-cinder-volume:16.1

MAINTAINER Infinidat Host Team <kfred@infinidat.com>

###Required Labels
LABEL name="rhosp16/openstack-cinder-volume-infinidat" \
      maintainer="Infinidat Storage, kfred@infinidat.com" \
      vendor="Infinidat" \
      version="1.0" \
      release="4" \
      summary="Red Hat OpenStack Platform 16.1 cinder-volume Infinidat Plugin" \
      description="Red Hat OpenStack Platform 16.1 cinder-volume Infinidat Plugin"

USER root

###Adding package
RUN curl https://bootstrap.pypa.io/3.4/get-pip.py -o get-pip.py
RUN python get-pip.py

RUN pip install --upgrade pip
COPY driver-requirements.txt /
RUN pip install -r driver-requirements.txt

###Adding mock rpm 
COPY python3-mock-3.0.5-1.el8ost.noarch.rpm /tmp/
RUN dnf install /tmp/python3-mock-3.0.5-1.el8ost.noarch.rpm -y 

# Add required license as text file in Liceses directory (GPL, MIT, APACHE, Partner End User Agreement, etc)
RUN mkdir /licenses
COPY licensing.txt /licenses

# switch the container back to the default user
USER cinder
