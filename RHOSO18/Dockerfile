FROM registry.redhat.io/rhoso/openstack-cinder-volume-rhel9:18.0

MAINTAINER Infinidat Host Team <adeiter@infinidat.com>

ARG VERSION=18.0
ARG RELEASE=1
ARG LICENSES=/licenses
ARG CINDER_LICENSE=https://opendev.org/openstack/cinder/raw/branch/master/LICENSE
ARG INFINIDAT_LICENSE=https://raw.githubusercontent.com/Infinidat/infinisdk/external/LICENSE
ARG PLUGIN_LICENSE=$LICENSES/licensing.txt

LABEL name="rhoso18/openstack-cinder-volume-infinidat-plugin"         \
      maintainer="adeiter@infinidat.com"                              \
      vendor="Infinidat"                                              \
      version="$VERSION"                                              \
      release="$RELEASE"                                              \
      summary="RHOSO $VERSION cinder-volume Infinidat Plugin"         \
      description="RHOSO $VERSION cinder-volume Infinidat Plugin"

USER root

RUN dnf install                                                       \
        --assumeyes                                                   \
        --releasever="9.4"                                            \
        dnf-plugins-core                                           && \
    dnf copr enable                                                   \
        --assumeyes                                                   \
        deiter/infinisdk                                           && \
    dnf install                                                       \
        --assumeyes                                                   \
        --releasever="9.4"                                            \
        python3-infinisdk                                          && \
    dnf update                                                        \
        --assumeyes                                                   \
        --releasever="9.4"                                            \
        --enablerepo="rhel-9-for-x86_64-appstream-aus-rpms"           \
        --enablerepo="rhel-9-for-x86_64-appstream-eus-rpms"           \
        --enablerepo="rhel-9-for-x86_64-baseos-aus-rpms"              \
        --enablerepo="rhel-9-for-x86_64-baseos-eus-rpms"              \
        --enablerepo="fast-datapath-for-rhel-9-x86_64-rpms"           \
        --enablerepo="rhceph-6-tools-for-rhel-9-x86_64-rpms"          \
        --enablerepo="rhel-9-for-x86_64-highavailability-eus-rpms"    \
        --enablerepo="rhel-9-for-x86_64-resilientstorage-eus-rpms" && \
    dnf upgrade-minimal                                               \
        --assumeyes                                                   \
        --security                                                    \
        --releasever="9.4"                                            \
        --enablerepo="rhoso-18.0-*"                                   \
        --enablerepo="rhoso-tools-18-*"                            && \
    dnf --enablerepo="*" clean all                                 && \
    install --verbose                                                 \
            --directory                                               \
            --mode 0755                                               \
            "$LICENSES"                                            && \
    curl --location                                                   \
         --silent                                                     \
         --show-error                                                 \
         "$CINDER_LICENSE"                                            \
         "$INFINIDAT_LICENSE"                                       | \
    tee "$PLUGIN_LICENSE"

USER cinder
