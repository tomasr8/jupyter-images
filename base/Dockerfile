# EXPERIMENTAL: SWAN Base Image using uv instead of conda/mamba
# 
# This is a prototype to evaluate potential size/speed improvements.
# Key differences from the conda-based image:
#   - Uses uv for Python and package management
#   - No conda/mamba overhead
#   - Potentially faster builds (10-100x faster package installation)
#   - Smaller footprint (no conda package cache structure)
#
# Trade-offs:
#   - Less isolation than conda environments
#   - Can't install non-Python system packages via package manager
#   - Scientific packages may be larger due to static linking in wheels
#     (but SWAN gets these from CVMFS anyway)

FROM gitlab-registry.cern.ch/linuxsupport/alma9-base:20240801-1

LABEL maintainer="swan-admins@cern.ch"
ARG NB_USER="jovyan"
ARG NB_UID="1000"
ARG NB_GID="100"
ARG PYTHON_VERSION=3.11

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# DNF configuration
RUN echo "skip_missing_names_on_install=False" >> /etc/dnf/dnf.conf

USER root

# =============================================================================
# System packages (same as original)
# =============================================================================
RUN dnf update -y && \
    dnf install -y 'dnf-command(config-manager)' && \
    dnf config-manager --set-enabled crb && \
    dnf install -y epel-release && \
    dnf install -y \
        bzip2 \
        sudo \
        tini \
        wget \
        curl \
        pandoc && \
    dnf clean all && \
    rm -rf /var/cache/dnf /var/log/dnf* /var/log/yum.*

# =============================================================================
# Environment configuration
# =============================================================================
ENV UV_HOME=/opt/uv \
    SHELL=/bin/bash \
    NB_USER="${NB_USER}" \
    NB_UID=${NB_UID} \
    NB_GID=${NB_GID} \
    PYTHONIOENCODING=UTF-8

# Virtual environment location (replaces CONDA_DIR)
ENV VIRTUAL_ENV=/opt/venv \
    HOME="/home/${NB_USER}"

# Add virtual environment and uv to PATH
ENV PATH="${VIRTUAL_ENV}/bin:${UV_HOME}:${PATH}"

# Copy fix-permissions script
COPY fix-permissions /usr/local/bin/fix-permissions
RUN chmod a+rx /usr/local/bin/fix-permissions

# =============================================================================
# Create user
# =============================================================================
RUN sed -i 's/^#force_color_prompt=yes/force_color_prompt=yes/' /etc/skel/.bashrc && \
    # Add virtual environment activation to bashrc
    echo 'if [ -d "${VIRTUAL_ENV}" ]; then source "${VIRTUAL_ENV}/bin/activate"; fi' >> /etc/skel/.bashrc

RUN echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers && \
    sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers && \
    useradd --no-log-init --create-home --shell /bin/bash --uid "${NB_UID}" --no-user-group "${NB_USER}" && \
    chmod g+w /etc/passwd && \
    fix-permissions "/home/${NB_USER}"

# =============================================================================
# Install uv
# =============================================================================
# uv is a single static binary (~10MB) - much smaller than mamba
RUN curl -LsSf https://astral.sh/uv/install.sh | \
    env UV_INSTALL_DIR="${UV_HOME}" INSTALLER_NO_MODIFY_PATH=1 sh && \
    chmod +x ${UV_HOME}/uv && \
    # Verify installation
    ${UV_HOME}/uv --version && \
    rm -rf $HOME/.config


RUN mkdir ${VIRTUAL_ENV} && \
    fix-permissions "${VIRTUAL_ENV}"

# =============================================================================
# Create Python virtual environment and install base packages
# =============================================================================

USER ${NB_UID}

RUN mkdir "/home/${NB_USER}/work" && \
    fix-permissions "/home/${NB_USER}"

WORKDIR /tmp

# Create virtual environment with specific Python version
# uv can download and manage Python versions!
RUN uv venv ${VIRTUAL_ENV} --python ${PYTHON_VERSION} && \
    # Verify Python version
    ${VIRTUAL_ENV}/bin/python --version

# Install Jupyter ecosystem
# Note: uv is MUCH faster than pip/mamba here (10-100x)
RUN uv pip install --python ${VIRTUAL_ENV}/bin/python \
    'jupyterlab==4.0.7' \
    'notebook==6.5.6' \
    'jupyterhub==4.1.6' \
    'nbclassic==1.0.0' \
    'jupyter_core>=5.5.0'

# Generate Jupyter config
RUN ${VIRTUAL_ENV}/bin/jupyter server --generate-config

# Clean up (uv has less cruft than conda, but still good practice)
RUN uv cache clean && \
    rm -rf /tmp/* ~/.cache/pip

# =============================================================================
# Container configuration
# =============================================================================
ENTRYPOINT ["tini", "-g", "--"]

COPY run-hooks.sh start.sh /usr/local/bin/

USER root

RUN mkdir /usr/local/bin/start-notebook.d && \
    mkdir /usr/local/bin/before-notebook.d

ENV JUPYTER_PORT=8888
EXPOSE $JUPYTER_PORT

CMD ["start-notebook.sh"]

COPY start-notebook.sh start-singleuser.sh /usr/local/bin/
COPY jupyter_server_config.py docker_healthcheck.py /etc/jupyter/

RUN fix-permissions /etc/jupyter/

RUN chmod +x /usr/local/bin/start-notebook.sh && \
    chmod +x /usr/local/bin/start-singleuser.sh && \
    chmod +x /usr/local/bin/run-hooks.sh && \
    chmod +x /usr/local/bin/start.sh

HEALTHCHECK --interval=5s --timeout=3s --start-period=5s --retries=3 \
    CMD /etc/jupyter/docker_healthcheck.py || exit 1

USER ${NB_UID}
WORKDIR "${HOME}"

# =============================================================================
# SWAN customizations
# =============================================================================
RUN uv pip install --python ${VIRTUAL_ENV}/bin/python swanheader==2.0.1

RUN jupyter labextension disable "@jupyterlab/apputils-extension:announcements"

USER root

RUN sed -i \
        -e 's/^CREATE_HOME.*/CREATE_HOME 0/g' \
        -e 's/^UID_MAX.*/UID_MAX 100000/g' \
        /etc/login.defs && \
    sed -i 's/^CREATE_MAIL_SPOOL.*/CREATE_MAIL_SPOOL=no/g' /etc/default/useradd

ENV PYDEVD_DISABLE_FILE_VALIDATION=1
ENV PYTHONNOUSERSITE=1

# =============================================================================
# Compatibility layer
# =============================================================================
# Create CONDA_DIR symlink for scripts that expect it
# This helps with backward compatibility
RUN ln -s ${VIRTUAL_ENV} /opt/conda || true

USER ${NB_UID}
