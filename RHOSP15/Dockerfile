FROM registry.redhat.io/rhosp15-rhel8/openstack-cinder-base
MAINTAINER Infinidat Host Team <arnony@infinidat.com>

###Required Labels
LABEL name="rhosp15/openstack-cinder-volume-infinidat-plugin" \
      maintainer="arnony@infinidat.com" \
      vendor="Infinidat" \
      version="1.0" \
      release="1" \
      summary="Red Hat OpenStack Platform 15.0 cinder-volume Infinidat Plugin" \
      description="Red Hat OpenStack Platform 15.0 cinder-volume Infinidat Plugin"

USER root

###Adding package
# Enable a repo to install a package
RUN curl https://bootstrap.pypa.io/3.4/get-pip.py -o get-pip.py
RUN python get-pip.py

RUN pip install --upgrade pip
RUN pip install --no-cache-dir -U setuptools
COPY driver-requirements.txt /
RUN pip install -r driver-requirements.txt

# Add required license as text file in Liceses directory (GPL, MIT, APACHE, Partner End User Agreement, etc)
RUN mkdir /licenses
COPY licensing.txt /licenses

USER cinder
