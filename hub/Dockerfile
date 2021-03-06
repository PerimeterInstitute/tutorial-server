FROM ubuntu:18.04
USER root

ENV DEBIAN_FRONTEND=noninteractive
RUN apt update && \
  apt install -y --no-install-recommends apt-utils && \
  apt install -y --no-install-recommends \
  python3 python3-pip openssh-client \
  libssl1.0-dev nodejs-dev node-gyp npm \
  gnupg policycoreutils python3-pam imagemagick curl vim wget \
  apt-transport-https ca-certificates git less \
  nfs-server iputils-ping emacs-nox debconf-utils \
  slapd ldap-utils ldapscripts libnss-ldap libpam-ldap patch rsyslog \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

# Make python3 the default
RUN update-alternatives --install /usr/bin/python python /usr/bin/python3 1 && \
    update-alternatives --install /usr/bin/pip pip /usr/bin/pip3 1

# Jupyterhub is a little picky about matching specific versions between
# the jupyterhub process and the notebook servers.
RUN pip install --no-cache-dir setuptools && \
    pip install --no-cache-dir \
    oauthenticator==0.9.0 \
    jupyter==1.0.0 \
    jupyterhub==1.0.0 \
    tornado==5.1.1 \
    python-oauth2==1.1.1 \
    jupyterhub-dummyauthenticator \
    notebook==6.0.2 \
    kubernetes==9.0.1

# For jupyterhub
RUN npm install -g configurable-http-proxy

# Jupyterhub spawner that starts notebook servers on Kubernetes pods
RUN git clone https://github.com/jupyterhub/kubespawner.git && \
    cd kubespawner && \
    python setup.py install

# Add "gcloud" program for interacting with Google Cloud Platform --
# for launching notebook servers in Google Kubernetes Engine
RUN echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list && \
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key --keyring /usr/share/keyrings/cloud.google.gpg add - && \
    apt update && apt install -y google-cloud-sdk kubectl

# Initialize LDAP server setup; we reset the password in run.sh
RUN echo "slapd slapd/domain string hub" | debconf-set-selections && \
    echo "slapd slapd/purge_database boolean true" | debconf-set-selections && \
    echo "slapd slapd/move_old_database boolean true" | debconf-set-selections && \
    echo "slapd shared/organization string PI" | debconf-set-selections && \
    echo "slapd slapd/allow_ldap_v2 boolean false" | debconf-set-selections && \
    echo "slapd slapd/no_configuration boolean false" | debconf-set-selections && \
    echo "slapd slapd/backend string MDB" | debconf-set-selections && \
    echo "slapd slapd/password1 password XXX" | debconf-set-selections && \
    echo "slapd slapd/password2 password XXX" | debconf-set-selections && \
    echo "libpam-runtime  libpam-runtime/profiles multiselect     unix, ldap" | debconf-set-selections && \
    dpkg-reconfigure slapd libpam-runtime

# Copy service account credentials!
# We instead assume the key is in /nfs/sys/svc.json
# and authorize in run.sh
#COPY research-technologies-testbed-a786f12c5112.json /svc.json

## Convenience: ssh direct to root@hub with key...
RUN apt install -y --no-install-recommends openssh-server
RUN echo "Port 2222" >> /etc/ssh/sshd_config
RUN mkdir -p /root/.ssh && chmod 700 /root/.ssh
# Create your own key with ssh-keygen -f id_root_hub ...
COPY id_root_hub.pub /root/.ssh/authorized_keys

# append
RUN echo "BASE dc=hub\nURI ldap://hub\n" >> /etc/ldap/ldap.conf
COPY nsswitch.conf /etc
COPY etcldap.conf /etc/ldap.conf
COPY ldapscripts.conf /etc/ldapscripts
COPY add_content.ldif /tmp

RUN mkdir -p /usr/local/lib/python3.6/dist-packages/cyolauthenticator
COPY __init__.py /usr/local/lib/python3.6/dist-packages/cyolauthenticator/__init__.py
COPY cyolauthenticator.py /usr/local/lib/python3.6/dist-packages/cyolauthenticator/cyolauthenticator.py
#RUN pip install cyolauthenticator

# would need gcc etc!
#RUN apt install --no-install-recommends libcurl4-gnutls-dev
#RUN pip install pycurl

RUN pip install --no-cache-dir wheel && \
    pip install jupyterhub-idle-culler

# HACK -- install Julia on the Jupyterhub node so that we can run "julia"
# to compile the "IJulia" package in the user's home dir.
RUN wget -nv https://julialang-s3.julialang.org/bin/linux/x64/1.5/julia-1.5.2-linux-x86_64.tar.gz && \
     tar xvz -C /usr/local --strip-components 1 -f julia-1.5.2-linux-x86_64.tar.gz && \
     rm julia-1.5.2-linux-x86_64.tar.gz

# Edit the message in the h1 tags to customize your tutorial
COPY login.html error.html /usr/local/share/jupyterhub/templates/
# Replace logo.png with whatever logo you wish to use to brand the server
COPY pi-logo.png /usr/local/share/jupyterhub/static/images/logo.png

COPY run.sh run-log.sh inituser.sh jup-config.py jup-config-template.py /
RUN chmod 755 /*.sh
COPY codeword.txt /usr/enable_mkuser
RUN echo "/nfs/home 10.0.0.0/8(rw,async,no_subtree_check,root_squash)" > /etc/exports

## Error JSONifying event causes spawn to fail
COPY jupyterhub.patch /
RUN cat jupyterhub.patch | (cd /usr/local/lib/python3.6/dist-packages && patch -p 1)

COPY nfs-pv.yaml nfs-pvc.yaml /

# This list of EXPOSEd ports is not complete (and not required when running in Google Compute Engine)
EXPOSE 80
EXPOSE 8081
EXPOSE 443
EXPOSE 2222
# per https://github.com/GoogleCloudPlatform/nfs-server-docker/blob/master/1/debian9/1.3/Dockerfile
EXPOSE 2049/tcp
EXPOSE 20048/tcp

CMD bash run-log.sh

