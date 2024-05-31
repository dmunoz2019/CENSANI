ARG ODOO_TAG
FROM ubuntu:jammy

SHELL ["/bin/bash", "-xo", "pipefail", "-c"]

# Generate locale C.UTF-8 for postgres and general locale data
ENV LANG en_US.UTF-8

# Retrieve the target architecture to install the correct wkhtmltopdf package
ARG TARGETARCH

# Install some deps, lessc and less-plugin-clean-css, and wkhtmltopdf

RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        dirmngr \
        fonts-noto-cjk \
        gnupg \
        libssl-dev \
        node-less \
        npm \
        python3-magic \
        python3-num2words \
        python3-odf \
        python3-pdfminer \
        python3-pip \
        python3-phonenumbers \
        python3-pyldap \
        python3-qrcode \
        python3-renderpm \
        python3-setuptools \
        python3-slugify \
        python3-vobject \
        python3-watchdog \
        python3-xlrd \
        python3-xlwt \
        xz-utils && \
    if [ -z "${TARGETARCH}" ]; then \
        TARGETARCH="$(dpkg --print-architecture)"; \
    fi; \
    WKHTMLTOPDF_ARCH=${TARGETARCH} && \
    case ${TARGETARCH} in \
    "amd64") WKHTMLTOPDF_ARCH=amd64 && WKHTMLTOPDF_SHA=967390a759707337b46d1c02452e2bb6b2dc6d59  ;; \
    "arm64")  WKHTMLTOPDF_SHA=90f6e69896d51ef77339d3f3a20f8582bdf496cc  ;; \
    "ppc64le" | "ppc64el") WKHTMLTOPDF_ARCH=ppc64el && WKHTMLTOPDF_SHA=5312d7d34a25b321282929df82e3574319aed25c  ;; \
    esac \
    && curl -o wkhtmltox.deb -sSL https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.jammy_${WKHTMLTOPDF_ARCH}.deb \
    && echo ${WKHTMLTOPDF_SHA} wkhtmltox.deb | sha1sum -c - \
    && apt-get install -y --no-install-recommends ./wkhtmltox.deb \
    && rm -rf /var/lib/apt/lists/* wkhtmltox.deb

# install latest postgresql-client
RUN echo 'deb http://apt.postgresql.org/pub/repos/apt/ jammy-pgdg main' > /etc/apt/sources.list.d/pgdg.list \
    && GNUPGHOME="$(mktemp -d)" \
    && export GNUPGHOME \
    && repokey='B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8' \
    && gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "${repokey}" \
    && gpg --batch --armor --export "${repokey}" > /etc/apt/trusted.gpg.d/pgdg.gpg.asc \
    && gpgconf --kill all \
    && rm -rf "$GNUPGHOME" \
    && apt-get update  \
    && apt-get install --no-install-recommends -y postgresql-client \
    && rm -f /etc/apt/sources.list.d/pgdg.list \
    && rm -rf /var/lib/apt/lists/*

# Install rtlcss (on Debian buster)
RUN npm install -g rtlcss

# Install Odoo
ENV ODOO_VERSION 17.0
ARG ODOO_RELEASE=20240521

RUN curl -o odoo.deb -sSL http://nightly.odoo.com/${ODOO_VERSION}/nightly/deb/odoo_${ODOO_VERSION}.${ODOO_RELEASE}_all.deb && \
    ODOO_SHA=$(sha256sum odoo.deb | awk '{print $1}') && \
    echo "${ODOO_SHA} odoo.deb" | sha256sum -c - && \
    apt-get update && \
    apt-get -y install --no-install-recommends ./odoo.deb && \
    rm -rf /var/lib/apt/lists/* odoo.deb


# Copy entrypoint_2 script and Odoo configuration file
COPY ./odoo/entrypoint_2.sh /
RUN chmod +x /entrypoint_2.sh
COPY ./odoo/odoo.conf /etc/odoo/

# Set permissions and Mount /var/lib/odoo to allow restoring filestore and /mnt/extra-addons for users addons
RUN chown odoo /etc/odoo/odoo.conf \
    && mkdir -p /mnt/extra-addons \
    && chown -R odoo /mnt/extra-addons
VOLUME ["/var/lib/odoo", "/mnt/extra-addons"]

# Expose Odoo services
EXPOSE 8069 8071 8072

# Set the default config file
ENV ODOO_RC /etc/odoo/odoo.conf

COPY ./odoo/wait-for-psql.py /usr/local/bin/wait-for-psql.py

# Set default user when running the container
USER odoo

ENTRYPOINT ["/entrypoint_2.sh"]
CMD ["odoo"]

# Continuamos con el proceso normal
USER root

ARG ODOO_TAG
ARG ROOT_PATH
ARG LOG_PATH
ARG ENTERPRISE_USER
ARG ENTERPRISE_ACCESS_TOKEN
ARG ENTERPRISE_ADDONS
ARG GITHUB_USER
ARG GITHUB_ACCESS_TOKEN
ARG THIRD_PARTY_ADDONS
ARG ODOO_RC
ARG USE_REDIS
ARG USE_S3
ARG USE_SENTRY

ENV ODOO_TAG=${ODOO_TAG} \
    LOG_PATH=${LOG_PATH} \
    ENTERPRISE_USER=${ENTERPRISE_USER} \
    ENTERPRISE_ACCESS_TOKEN=${ENTERPRISE_ACCESS_TOKEN} \
    ENTERPRISE_ADDONS=${ENTERPRISE_ADDONS} \
    GITHUB_USER=${GITHUB_USER} \
    GITHUB_ACCESS_TOKEN=${GITHUB_ACCESS_TOKEN} \
    THIRD_PARTY_ADDONS=${THIRD_PARTY_ADDONS} \
    ODOO_RC=${ODOO_RC} \
    USE_REDIS=${USE_REDIS} \
    USE_S3=${USE_S3} \
    USE_SENTRY=${USE_SENTRY}


RUN touch ${LOG_PATH} && chown odoo:odoo ${LOG_PATH}


RUN apt-get update && apt-get install -y \
    git\
    zip \
    unzip \
    git \
    git-man \
    less \
    libcbor0.8 \
    libcurl3-gnutls \
    libedit2 \
    liberror-perl \
    libfido2-1 \
    libxmuu1 \
    openssh-client \
    patch \
    xauth \
    && rm -rf /var/lib/apt/lists/*


COPY --chown=odoo:odoo ./odoo/requirements.txt /tmp/requirements.txt

RUN python3 -m pip install -r /tmp/requirements.txt && \
    rm /tmp/requirements.txt

COPY --chown=odoo:odoo ./odoo/clone-addons.sh /
COPY --chown=odoo:odoo ./odoo/third-party-addons.txt /
RUN /clone-addons.sh

COPY --chown=odoo:odoo ./.env /
COPY --chown=odoo:odoo ./odoo/odoo.conf /
COPY --chown=odoo:odoo ./odoo/odoorc.sh /

RUN /odoorc.sh && chown odoo:odoo ${ODOO_RC}

USER odoo