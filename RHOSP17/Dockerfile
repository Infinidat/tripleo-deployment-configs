FROM registry.redhat.io/rhosp-rhel9/openstack-cinder-volume:17.1

MAINTAINER Infinidat Host Team <adeiter@infinidat.com>

ARG ARCH=x86_64
ARG RHEL_VERSION=9.2
ARG RHEL_MAJOR_VERSION=9
ARG RHOSP_VERSION=17.1
ARG RHEL_REPO=rhel-$RHEL_MAJOR_VERSION-for-$ARCH-*-eus-rpms
ARG RHOSP_REPO=openstack-$RHOSP_VERSION-for-rhel-$RHEL_MAJOR_VERSION-$ARCH-rpms
ARG LICENSES=/licenses
ARG CINDER_LICENSE=https://opendev.org/openstack/cinder/raw/branch/master/LICENSE
ARG INFINIDAT_LICENSE=https://raw.githubusercontent.com/Infinidat/infinisdk/external/LICENSE
ARG PLUGIN_LICENSE=$LICENSES/licensing.txt

LABEL name="rhosp17/openstack-cinder-volume-infinidat-plugin"                                \
      maintainer="adeiter@infinidat.com"                                                     \
      vendor="Infinidat"                                                                     \
      version="$RHOSP_VERSION"                                                               \
      release="2"                                                                            \
      summary="Red Hat OpenStack Platform $RHOSP_VERSION cinder-volume Infinidat Plugin"     \
      description="Red Hat OpenStack Platform $RHOSP_VERSION cinder-volume Infinidat Plugin"

USER root

RUN dnf upgrade-minimal                \
        --assumeyes                    \
        --security                     \
        --releasever="$RHEL_VERSION"   \
        --disablerepo="*"              \
        --enablerepo="$RHEL_REPO"      \
        --enablerepo="$RHOSP_REPO"  && \
    dnf install                        \
        --assumeyes                    \
        dnf-plugins-core            && \
    dnf copr enable                    \
        --assumeyes                    \
        deiter/infinisdk            && \
    dnf install                        \
        --assumeyes                    \
        --releasever="$RHEL_VERSION"   \
        --disablerepo="*"              \
        --enablerepo="*infinisdk*"     \
        --enablerepo="$RHEL_REPO"      \
        --enablerepo="$RHOSP_REPO"     \
        python3-infinisdk           && \
    dnf clean all                   && \
    install --verbose                  \
            --directory                \
            --mode 0755                \
            "$LICENSES"             && \
    curl --location                    \
         --silent                      \
         --show-error                  \
         "$CINDER_LICENSE"             \
         "$INFINIDAT_LICENSE" |        \
    tee "$PLUGIN_LICENSE"

USER cinder
