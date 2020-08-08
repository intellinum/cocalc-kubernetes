FROM ubuntu:18.04

ENV DEBIAN_FRONTEND=noninteractive LANG=C.UTF-8 LC_ALL=C.UTF-8

LABEL maintainer="Intellinum Analytics, Inc. <yuan@intellinum.co>"

# So we can source (http://goo.gl/oBPi5G), which we will do in build.sh.
RUN rm /bin/sh && ln -s /bin/bash /bin/sh

# Useful code to get stuff done -- move up to the top when stable.
RUN \
     apt-get update \
  && apt-get install -y  \
       gnupg2 \
       openssh-server \
       vim \
       apt-utils \
       wget \
       curl \
       software-properties-common \
       texlive \
       texlive-latex-extra \
       texlive-extra-utils \
       texlive-xetex \
       texlive-luatex \
       texlive-bibtex-extra \
       liblog-log4perl-perl \
       tmux \
       flex \
       bison \
       libreadline-dev \
       htop \
       screen \
       pandoc \
       aspell \
       poppler-utils \
       net-tools \
       python \
       python-pip \
       python3-pip \
       make \
       g++ \
       psmisc \
       haproxy \
       nginx \
       rsync \
       tidy \
       bup \
       inetutils-ping \
       lynx \
       telnet \
       emacs \
       subversion \
       ssh \
       m4 \
       latexmk \
       libpq5 \
       libpq-dev \
       build-essential \
       automake \
       gfortran \
       dpkg-dev \
       libssl-dev \
       imagemagick \
       libcairo2-dev \
       libcurl4-openssl-dev \
       libxml2-dev \
       graphviz \
       smem \
       octave \
       python3-yaml \
       python3-matplotlib \
       python3-jupyter* \
       python-matplotlib* \
       python-ipywidgets \
       python-ipywidgets-doc \
       python3-ipywidgets \
       jupyter \
       locales \
       locales-all \
       clang-format \
       golang \
       r-cran-formatr \
  && apt-get clean && apt-get -y autoremove

# Install prerequisites ubuntu packages for building the project nodejs server code.
RUN \
     apt-get update && apt-get upgrade -y \
  && apt-get install -y gnupg2 curl wget python3 python python-pip git make g++ dpkg-dev \
                        flex bison libreadline-dev libz-dev libssl-dev libpq5 libpq-dev vim telnet build-essential \
  && apt-get clean && apt-get -y autoremove


## Xpra backend support -- we have to use the debs from xpra.org,
## Since the official distro packages are ancient.
RUN \
     apt-get update \
  && DEBIAN_FRONTEND=noninteractive apt-get install -y xvfb websockify curl \
  && curl https://xpra.org/gpg.asc | apt-key add - \
  && echo "deb http://xpra.org/ bionic main" > /etc/apt/sources.list.d/xpra.list \
  && apt-get update \
  && DEBIAN_FRONTEND=noninteractive apt-get install -y xpra

## X11 apps to make x11 support useful.
## Will move this up in Dockerfile once stable.
RUN \
     apt-get update \
  && DEBIAN_FRONTEND=noninteractive apt-get install -y x11-apps dbus-x11 gnome-terminal \
     vim-gtk lyx libreoffice inkscape gimp chromium-browser texstudio evince mesa-utils \
     xdotool xclip x11-xkb-utils

# Jupyter widgets support
RUN pip install --no-cache-dir ipyleaflet
RUN pip3 install --no-cache-dir ipyleaflet

# octave kernel
RUN pip install octave_kernel

# python code formatting
RUN pip install yapf

# Build and install Sage -- see https://github.com/sagemath/docker-images
# The sage build uses sudo to switch to sage user.
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y sudo
COPY scripts/ /tmp/scripts
RUN chmod -R +x /tmp/scripts

RUN    adduser --quiet --shell /bin/bash --gecos "Sage user,101,," --disabled-password sage \
    && chown -R sage:sage /home/sage/

# make source checkout target, then run the install script
# see https://github.com/docker/docker/issues/9547 for the sync
# Sage can't be built as root, for reasons...
# Here -E inherits the environment from root, however it's important to
# include -H to set HOME=/home/sage, otherwise DOT_SAGE will not be set
# correctly and the build will fail!
RUN    mkdir -p /usr/local/sage \
    && chown -R sage:sage /usr/local/sage \
    && sudo -H -E -u sage /tmp/scripts/install_sage.sh /usr/local/ master \
    && sync

RUN /tmp/scripts/post_install_sage.sh /usr/local/ && rm -rf /tmp/* && sync

# Install SageTex
RUN \
     sudo -H -E -u sage sage -p sagetex \
  && cp -rv /usr/local/sage/local/share/texmf/tex/latex/sagetex/ /usr/share/texmf/tex/latex/ \
  && texhash

# Install R Jupyter Kernel package into R itself (so R kernel works).
# I can't get this to work ?!
# RUN echo "install.packages(c('repr', 'IRdisplay', 'evaluate', 'crayon', 'pbdZMQ', 'httr', 'uuid', 'digest'), repos='http://cran.us.r-project.org'); devtools::install_github('IRkernel/IRkernel')" | sage -R --no-save
# RUN echo "install.packages(c('repr', 'IRdisplay', 'evaluate', 'crayon', 'pbdZMQ', 'httr', 'uuid', 'digest'), repos='http://cran.us.r-project.org'); devtools::install_github('IRkernel/IRkernel')" | R --no-save

# Commit to checkout and build.
ARG branch=dev_1.0.2
# Get the source code of cocalc, putting it at /cocalc
RUN umask 022 && git clone https://github.com/intellinum/cocalc.git \
  && cd /cocalc && git pull && git fetch origin && git checkout ${branch:-HEAD}
  
# tini: https://github.com/krallin/tini
# This is also included in Docker itself via the `--init` flag
COPY tini /cocalc/bin/tini
RUN chmod +x /cocalc/bin/tini

# We install our own Node 10, which is independent of the node setup of the host image
# (i.e. we can change the software stack at will without interfering with running the project)
RUN \
     umask 022 \
  && cd /cocalc \
  && mkdir /cocalc/nvm \
  && curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | NVM_DIR=/cocalc/nvm bash \
  && source /cocalc/nvm/nvm.sh \
  && nvm install --latest-npm --no-progress 10 \
  && npm install -g coffeescript

# Get our custom build script.
COPY kubernetes_build.py /cocalc/kubernetes_build.py


# Run the build script once, so subsequent builds will be fast.
RUN umask 022 && source /cocalc/nvm/nvm.sh && /cocalc/kubernetes_build.py

# Delete some stupid big stuff from the /cocalc build; this won't be needed
# if we better refactor the smc-webapp code.  For now, we just do it.
RUN \
     cd /cocalc/src/smc-webapp/node_modules \
  && rm -rf three jquery-ui nyc react-draggable mathjax

# Create the project user's home account and home directory.
# TODO deal with .bashrc (/home/user is a volume)
COPY init /cocalc/init/

# This is our own all-in-one config file. Should be readable by the "user"
COPY kucalc-start-sshd.sh /cocalc/kucalc-start-sshd.sh

# Copy script over to start sage server.
COPY smc-sage-server /cocalc/bin/smc-sage-server

# Install support code for forking sage server
RUN umask 022 \
  && sage -pip install /cocalc/src/smc_sagews/

# The official Jupyter kernel with sage-9.0 (at least) is completely broken, due
# to calling the wrong script.  No clue what that's about. This works though:
COPY kernels/sagemath /usr/local/share/jupyter/kernels/sagemath

## Latest verison of JupyterLab and Jupyter Notebook classic
RUN umask 022 \
  && pip3 install jupyterlab jupyter

RUN umask 022 \
  && echo "install.packages('IRkernel'); IRkernel::installspec(user = FALSE)" | R --no-save

# Install the bash kernel
RUN umask 022 \
  && pip3 install bash_kernel \
  && python3 -m bash_kernel.install

# We do not use uid around 1000 since that's what the privileged
# accounts on the host are likely to use.
# ATTN some changes here (like, the user id number) should correspond to smc-build:smc-ansible/compute-k8s.yaml
RUN \
     umask 022 \
  && mkdir /home/user \
  && chown 2001:2001 -R /home/user \
  && /usr/sbin/groupadd --gid=2001 -o user \
  && /usr/sbin/useradd  --home-dir=/home/user --gid=2001 --uid=2001 --shell=/bin/bash user \
  && mkdir -p /var/run/sshd \
  && mkdir -p /usr/local/lib/python2.7/dist-packages/ \
  && echo "/cocalc/lib/python2.7/site-packages" > /usr/local/lib/python2.7/dist-packages/cocalc.pth \
  && chmod ugo=r /usr/local/lib/python2.7/dist-packages/cocalc.pth

RUN umask 022 \
  && echo "install.packages('rmarkdown');" | R --no-save

RUN umask 022 \
  && echo "install.packages('knitr');" | R --no-save
  
RUN umask 022 \
  && echo "install.packages('sparklyr');" | R --no-save
  
RUN umask 022 \
  && echo "install.packages('devtools');" | R --no-save

RUN umask 022 \
  && echo "install.packages('dplyr');" | R --no-save
  
RUN umask 022 \
  && echo "install.packages('DBI');" | R --no-save
  
# Disable ssh
RUN rm -f /usr/bin/ssh

# Install Mini-conda
RUN wget --quiet https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O /tmp/miniconda.sh && \
    /bin/bash /tmp/miniconda.sh -b -p /opt/conda && \
    rm /tmp/miniconda.sh

# Install custom conda env
RUN /opt/conda/bin/conda create -n spark2_py37 -y python=3.7 numpy ipykernel pandas
RUN /opt/conda/bin/conda install --name spark2_py37 -y pyspark==2.4.4 && \
/opt/conda/bin/conda install --name spark2_py37 -y scikit-learn && \
/opt/conda/bin/conda install --name spark2_py37 -y seaborn && \
/opt/conda/bin/conda install --name spark2_py37 -y scipy && \
/opt/conda/bin/conda install --name spark2_py37 -y matplotlib && \
/opt/conda/bin/conda install --name spark2_py37 -y sparkmagic && \
/opt/conda/bin/conda install --name spark2_py37 -y hdfs3 && \
echo 'y'| /opt/conda/envs/spark2_py37/bin/pip install statsmodels \
echo 'y'| /opt/conda/envs/spark2_py37/bin/pip install mlflow==1.10.0

RUN chown -R user:user /opt/conda

# Install kernel
COPY kernels/spark2_py37 /usr/local/share/jupyter/kernels/spark2_py37

# Install Sparkmagic
RUN pip3 install sparkmagic
# RUN /opt/conda/bin/conda install -y -c conda-forge nodejs
# RUN jupyter labextension install "@jupyter-widgets/jupyterlab-manager"
RUN jupyter nbextension enable --py --sys-prefix widgetsnbextension 
RUN pip3 install numpy --upgrade
RUN jupyter serverextension enable --py sparkmagic

RUN mkdir -p /opt/sparkmagic
RUN chown user:user /opt/sparkmagic
COPY sparkmagic/config.json /opt/sparkmagic/config.json

# =========Switch to being the non-privileged user.=========
USER user
WORKDIR /home/user

EXPOSE 2222 6000 6001

ENTRYPOINT ["/cocalc/bin/tini", "--"]
CMD ["sh", "-c", "env -i /cocalc/init/init.sh $COCALC_PROJECT_ID"]

