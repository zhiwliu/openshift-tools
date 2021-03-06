FROM openshifttools/oso-centos7-ops-base:latest
#     ___ ___ _  _ ___ ___    _ _____ ___ ___         
#    / __| __| \| | __| _ \  /_\_   _| __|   \        
#   | (_ | _|| .` | _||   / / _ \| | | _|| |) |       
#    \___|___|_|\_|___|_|_\/_/_\_\_|_|___|___/_ _____ 
#   |   \ / _ \  | \| |/ _ \_   _| | __|   \_ _|_   _|
#   | |) | (_) | | .` | (_) || |   | _|| |) | |  | |  
#   |___/ \___/  |_|\_|\___/ |_|   |___|___/___| |_|  
# 


# Pause indefinitely if asked to do so.
RUN test "$OO_PAUSE_ON_BUILD" = "true" && while true ; do sleep 10 ; done || :

# PCP
##################
# install pcp-collector and it's dependencies, clean the cache.
RUN yum install -y pcp pcp-conf pcp-collector xz && yum clean all
# Run in the container as root - avoids PCP_USER mismatches
RUN sed -i -e 's/PCP_USER=.*$/PCP_USER=root/' -e 's/PCP_GROUP=.*$/PCP_GROUP=root/' /etc/pcp.conf

# Disable service advertising - no avahi daemon in the container
# (dodges warnings from pmcd attempting to connect during startup)
RUN . /etc/pcp.conf && echo "-A" >> $PCP_PMCDOPTIONS_PATH

# denote this as a container environment, for rc scripts
ENV PCP_CONTAINER_IMAGE pcp-collector
ENV NAME pcp-collector
ENV IMAGE pcp-collector
ENV PATH /usr/share/pcp/lib:/usr/libexec/pcp/bin:$PATH

# script to watch health of pmcd
ADD check-pmcd-status.sh /usr/local/bin/check-pmcd-status.sh
##################

RUN echo -e "\n\nalias oca='KUBECONFIG=/tmp/admin.kubeconfig oc '" >> /root/.bashrc
RUN echo "alias oadma='KUBECONFIG=/tmp/admin.kubeconfig oadm '" >> /root/.bashrc

# Need to manually install the RPMs, since yum will get confused
# when resolving dependencies for /usr/bin/oc and /usr/bin/oadm
# in python-openshift-tools-monitoring-openshift.rpm using the
# maxamillion/origin-next copr repository below
RUN cd /etc/yum.repos.d && curl -O https://copr.fedorainfracloud.org/coprs/maxamillion/origin-next/repo/epel-7/maxamillion-origin-next-epel-7.repo
RUN yum install -y origin origin-clients

RUN yum clean metadata && \
    yum install -y python-pip pcp pcp-conf pcp-testsuite \
        python-requests \
        python-openshift-tools \
        python-openshift-tools-monitoring-pcp \
        python-openshift-tools-monitoring-docker \
        python-openshift-tools-monitoring-zagg \
        python-openshift-tools-monitoring-openshift \
        python-openshift-tools-ansible \
        python-openshift-tools-web \
        openshift-tools-scripts \
        openshift-tools-scripts-cloud-aws \
        openshift-tools-scripts-monitoring-pcp \
        openshift-tools-scripts-monitoring-docker \
        openshift-tools-scripts-monitoring-zagg-client \
        openshift-tools-scripts-monitoring-aws \
        openshift-tools-scripts-monitoring-openshift \
        openshift-tools-scripts-monitoring-autoheal \
        pcp-manager pcp-webapi python-pcp \
        openvswitch \
        docker-python && \
    yum -y update && \
    yum clean all

ADD urllib3-connectionpool-patch /root/
RUN yum install -y patch && yum clean all && cd /usr/lib/python2.7/site-packages/ && patch -p1 < /root/urllib3-connectionpool-patch

# Ansible startup configuration playbook
ADD root /root

# FIXME: These are vendor libs that need to be packaged and installed via RPM.
ADD vendor/prometheus_client /usr/lib/python2.7/site-packages/prometheus_client/

# Create ops-runner.log file with proper permissions
RUN touch /var/log/ops-runner.log && chmod 664 /var/log/ops-runner.log

# Setup the AWS credentials file so that we can populate it on startup.
RUN mkdir -p /root/.aws && \
    touch /root/.aws/credentials && \
    chmod g+rw /root/.aws/credentials

# Add the start script and tell the container to run it by default
ADD start.sh /usr/local/bin/
CMD /usr/local/bin/start.sh
