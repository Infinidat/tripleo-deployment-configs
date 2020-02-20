FROM registry.access.redhat.com/rhosp13/openstack-cinder-volume
MAINTAINER Infinidat Host Team <arnony@infinidat.com>

###Required Labels
LABEL name="rhosp13/openstack-cinder-volume-infinidat-plugin" \
      maintainer="arnony@infinidat.com" \
      vendor="Infinidat" \
      version="1.0" \
      release="1" \
      summary="Red Hat OpenStack Platform 13.0 cinder-volume Infinidat Plugin" \
      description="Red Hat OpenStack Platform 13.0 cinder-volume Infinidat Plugin"

USER root

###Adding package
RUN curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
RUN python get-pip.py

RUN pip install --no-cache-dir -U setuptools
COPY driver-requirements.txt /
RUN pip install -r driver-requirements.txt

###fix infinidat cinder driver bug
RUN sed -i -e 's/create_child/create_snapshot/g' /usr/lib/python2.7/site-packages/cinder/volume/drivers/infinidat.py

# Add required license as text file in Liceses directory (GPL, MIT, APACHE, Partner End User Agreement, etc)
RUN mkdir /licenses
COPY licensing.txt /licenses

USER cinder
