## Dockerfile for constructing the base Open Education Effort (OPE)
## container.  This container contains everything required for authoring OPE courses
## Well is should anyway ;-)

ARG FROM_REG
ARG FROM_IMAGE
ARG FROM_TAG

FROM ${FROM_REG}${FROM_IMAGE}${FROM_TAG} AS stage-one

LABEL maintainer="Open Education <opeffort@gmail.com>"

# ARGS are consumed and reset after FROM
# so any arguments you want to use below must be stated below FROM

ARG ADDITIONAL_DISTRO_PACKAGES
ARG PYTHON_PREREQ_VERSIONS
ARG PYTHON_INSTALL_PACKAGES
ARG GDB_BUILD_SRC
ARG UNMIN

# Fix: https://github.com/hadolint/hadolint/wiki/DL4006
# Fix: https://github.com/koalaman/shellcheck/wiki/SC3014
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Add a "USER root" statement followed by RUN statements to install system packages using apt-get,
# change file permissions, etc.

# install linux packages that we require for systems classes
USER root

RUN dpkg --add-architecture i386 && \
    apt-get -y update --fix-missing && \
    apt-get -y install ${ADDITIONAL_DISTRO_PACKAGES} && \
    apt-get clean && rm -rf /var/lib/apt/lists/* 

# Now that distro packages are installed lets install the python packages we want
# This was inspired by
# https://github.com/jupyter/docker-stacks/blob/b186ce5fea6aa9af23fb74167dca52908cb28d71/scipy-notebook/Dockerfile

USER ${NB_UID}

# sometimes there are problems with the existing version of installed python packages
# these need to be installed to the prerequisite versions before we can install the rest of the
# packages. Install Python 3 packages
RUN mamba install --quiet --yes ${PYTHON_PREREQ_VERSIONS}  && \
    mamba install --quiet --yes ${PYTHON_INSTALL_PACKAGES} && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}" && \
    mamba clean -afy

USER root

# get and build gdb form source so that we have a current version >10 that support more advanced tui functionality
RUN if [[ -n "${GDB_BUILD_SRC}" ]] ; then \
      cd /tmp && \
      wget http://ftp.gnu.org/gnu/gdb/${GDB_BUILD_SRC}.tar.gz && \
      tar -zxf ${GDB_BUILD_SRC}.tar.gz && \
      cd ${GDB_BUILD_SRC} && \
      ./configure --prefix /usr/local --enable-tui=yes && \
      make -j 4 && make install && \
      cd /tmp && \
      rm -rf ${GDB_BUILD_SRC} && rm ${GDB_BUILD_SRC}.tar.gz ; \
    fi

# adding our custom OPE version of RISE for lab -- remove once main RISE is fixed
RUN  wget -O - https://dl.yarnpkg.com/debian/pubkey.gpg 2> /dev/null | sudo apt-key add - && \
     echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list && \
     sudo apt-get update -y && \
     sudo apt-get install yarn -y && \
     mkdir /home/RISE && \
     chown jovyan /home/RISE && \
     fix-permissions /home/RISE && \
     apt-get clean && rm -rf /var/lib/apt/lists/*

USER $NB_USER
# last work by Yiqin has been merged into Fcollnval's fork that seems to be the
# main place the work for lab support is going switched over to using his fork
RUN  git clone https://github.com/fcollonval/RISE.git -b ft/jlab3  /home/RISE  && \
     cd /home/RISE && \
     yarn install && \
     yarn run build && \
     pip install -e . 

## Dockerfile for constructing the base Open Education Effort (OPE)
## container.  This container contains everything required for authoring OPE courses
## Well is should anyway ;-)

ARG FROM_REG
ARG FROM_IMAGE
ARG FROM_TAG

FROM ${FROM_REG}${FROM_IMAGE}${FROM_TAG} AS stage-two
#COPY --from=stage-one /opt/conda/lib/python3.10/site-packages/ /opt/conda/lib/python3.10/site-packages/ 
#COPY --from=stage-one /opt/conda/lib/ /opt/conda/lib/
COPY --from=stage-one /opt/conda /opt/conda
COPY --from=stage-one /usr/local/bin /usr/local/bin
COPY --from=stage-one /home/RISE /home/RISE

LABEL maintainer="Open Education <opeffort@gmail.com>"

ARG JUPYTER_ENABLE_EXTENSIONS
ARG JUPYTER_DISABLE_EXTENSIONS
ARG UNMIN

# Fix: https://github.com/hadolint/hadolint/wiki/DL4006
# Fix: https://github.com/koalaman/shellcheck/wiki/SC3014
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER ${NB_UID}

RUN for ext in ${JUPYTER_ENABLE_EXTENSIONS} ; do \
   jupyter nbextension enable "$ext" ; \
   done

# disable extensions -- disabling core extensions can be really useful for customizing
#                       jupyterlab user experience
RUN for ext in ${JUPYTER_DISABLE_EXTENSIONS} ; do \
   jupyter labextension disable "$ext" ; \
   done

# Import matplotlib the first time to build the font cache.
ENV XDG_CACHE_HOME="/home/${NB_USER}/.cache/"

RUN MPLBACKEND=Agg python -c "import matplotlib.pyplot" && \
    fix-permissions "/home/${NB_USER}"

# copy overrides.json
COPY settings ${CONDA_DIR}/share/jupyter/lab/settings

USER root

# we want the container to feel more like a fully fledged system so we are pulling the trigger and unminimizing it
RUN [[ $UNMIN == "yes" ]] &&  yes | unminimize || true


# Add ARM development environtment
COPY aarch64vm /home/aarch64vm

# final bits of cleanup
RUN touch /home/${NB_USER}/.hushlogin && \
# use a short prompt to improve default behaviour in presentations
    echo "export PS1='\$ '" >> /home/${NB_USER}/.bashrc && \
# work around bug when term is xterm and emacs runs in xterm.js -- causes escape characters in file
    echo "export TERM=linux" >> /home/${NB_USER}/.bashrc && \
# finally remove default working directory from joyvan home
    rmdir /home/${NB_USER}/work

# one more fix permissions for good measure ;-)
RUN  fix-permissions "${CONDA_DIR}" && \
     fix-permissions "/home/${NB_USER}" && \
     fix-permissions "/home/RISE" && \
     fix-permissions /home/aarch64vm

# as per the nbstripout readme we setup nbstripout be always be used for the joyvan user for all repos
RUN nbstripout --install --system 

# Static Customize for OPE USER ID choices
# To avoid problems with start.sh logic we do not modify user name
# FIXME: Add support for swinging home directory if you want to point to a mounted volume
ARG DEFAULT_NB_UID=${NB_UID}
ARG OPE_UID
ARG OPE_GID
ARG OPE_GROUP
ARG CHOWN_HOME=yes
ARG CHOWN_HOME_OPTS="-R"
ARG CHOWN_EXTRA_OPTS='-R'
ARG CHOWN_EXTRA="/home/aarch64vm /home/RISE ${CONDA_DIR}"
ENV OPE_UID=${OPE_UID}
ARG NB_UID=${OPE_UID}
ENV NB_UID=${NB_UID}
ENV OPE_GID=${OPE_GID}
ARG NB_GID=${OPE_GID}
ENV NB_GID=${NB_GID}
ENV OPE_GROUP=${OPE_GROUP}
ARG NB_GROUP=${OPE_GROUP}
ENV NB_GROUP=${NB_GROUP}

# # use built in startup script to setup new user home
RUN /usr/local/bin/start.sh true; \
    echo "hardcoding $NB_USER to uid=$NB_UID and group name $NB_GROUP with gid=$NB_GID shell to /bin/bash" ; \
    usermod -s /bin/bash $NB_USER ; \
    [[ -w /etc/passwd ]] && echo "Removing write access to /etc/passwd" && chmod go-w /etc/passwd

# Done customization

# jupyter-stack contains logic to run custom start hook scripts from
# two locations -- /usr/local/bin/start-notebook.d and
#                 /usr/local/bin/before-notebook.d
# and scripts in these directoreis are run automatically
# an opportunity to set things up based on dynamic facts such as user name
COPY start-notebook.d /usr/local/bin/start-notebook.d

ENV USER=$NB_USER

USER ${NB_UID}

