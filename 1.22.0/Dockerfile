# Stage 1: Build OpenSSL.
FROM debian:bullseye as openssl

ENV VERSION_OPENSSL=openssl-3.4.0 \
    SHA256_OPENSSL=e15dda82fe2fe8139dc2ac21a36d4ca01d5313c75f99f46c4e8a27709b7294bf \
    SOURCE_OPENSSL=https://www.openssl.org/source/ \
    OPGP_OPENSSL_1=EFC0A467D613CB83C7ED6D30D894E2CE8B3D79F5 \
    OPGP_OPENSSL_2=7953AC1FBC3DC8B3B292393ED5E9E43F7DF9EE8C \
    OPGP_OPENSSL_3=8657ABB260F056B1E5190839D9C4D26D0E604491 \
    OPGP_OPENSSL_4=B7C1C14360F353A36862E4D5231C84CDDCC69C45 \
    OPGP_OPENSSL_5=A21FAB74B0088AA361152586B8EF1A6BA9DA2D5C \
    OPGP_OPENSSL_6=C1F33DD8CE1D4CC613AF14DA9195C48241FBF7DD \
    OPGP_OPENSSL_7=E5E52560DD91C556DDBDA5D02064C53641C25E5D \
    OPGP_OPENSSL_8=BA5473A2B0587B07FB27CF2D216094DFD0CB81EF

WORKDIR /tmp/src

RUN set -ex && \
    build_deps="build-essential ca-certificates curl dirmngr gnupg libidn2-0-dev libssl-dev" && \
    DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y --no-install-recommends $build_deps && \
    curl -L $SOURCE_OPENSSL$VERSION_OPENSSL.tar.gz -o openssl.tar.gz && \
    echo "${SHA256_OPENSSL} ./openssl.tar.gz" | sha256sum -c - && \
    curl -L $SOURCE_OPENSSL$VERSION_OPENSSL.tar.gz.asc -o openssl.tar.gz.asc && \
    GNUPGHOME="$(mktemp -d)" && \
    export GNUPGHOME && \
    gpg --no-tty --keyserver keyserver.ubuntu.com --recv-keys "$OPGP_OPENSSL_1" "$OPGP_OPENSSL_2" "$OPGP_OPENSSL_3" "$OPGP_OPENSSL_4" "$OPGP_OPENSSL_5" "$OPGP_OPENSSL_6" "$OPGP_OPENSSL_7" "$OPGP_OPENSSL_8" && \
    gpg --batch --verify openssl.tar.gz.asc openssl.tar.gz && \
    tar xzf openssl.tar.gz && \
    cd $VERSION_OPENSSL && \
    ./Configure linux-x32 && \
    ./config --prefix=/opt/openssl --openssldir=/opt/openssl no-weak-ssl-ciphers no-ssl3 no-shared -DOPENSSL_NO_HEARTBEATS -fstack-protector-strong && \
    make depend && \
    nproc | xargs -I % make -j% && \
    make install_sw && \
    apt-get purge -y --auto-remove $build_deps && \
    rm -rf /tmp/* /var/tmp/* /var/lib/apt/lists/*

# Stage 2: Build Unbound
FROM debian:bullseye as unbound

ENV NAME=unbound \
    UNBOUND_VERSION=1.22.0 \
    UNBOUND_SHA256=c5dd1bdef5d5685b2cedb749158dd152c52d44f65529a34ac15cd88d4b1b3d43 \
    UNBOUND_DOWNLOAD_URL=https://nlnetlabs.nl/downloads/unbound/unbound-1.22.0.tar.gz

WORKDIR /tmp/src

COPY --from=openssl /opt/openssl /opt/openssl

RUN build_deps="curl gcc libc-dev libevent-dev libexpat1-dev libnghttp2-dev make" && \
    set -x && \
    DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y --no-install-recommends $build_deps bsdmainutils ca-certificates ldnsutils libevent-2.1-7 libexpat1 && \
    curl -sSL $UNBOUND_DOWNLOAD_URL -o unbound.tar.gz && \
    echo "${UNBOUND_SHA256} *unbound.tar.gz" | sha256sum -c - && \
    tar xzf unbound.tar.gz && \
    rm -f unbound.tar.gz && \
    cd unbound-1.22.0 && \
    groupadd _unbound && \
    useradd -g _unbound -s /etc -d /dev/null _unbound && \
    ./configure --disable-dependency-tracking --prefix=/opt/unbound --with-pthreads --with-username=_unbound --with-ssl=/opt/openssl --with-libevent --with-libnghttp2 --enable-tfo-server --enable-tfo-client --enable-event-api && \
    make install && \
    mv /opt/unbound/etc/unbound/unbound.conf /opt/unbound/etc/unbound/unbound.conf.example && \
    apt-get purge -y --auto-remove $build_deps && \
    rm -rf /opt/unbound/share/man /tmp/* /var/tmp/* /var/lib/apt/lists/*

# Stage 3: Final image with init commands baked in
FROM debian:bullseye

WORKDIR /tmp/src

# Copy OpenSSL and Unbound files from previous stage
COPY --from=unbound /opt /opt

# Install runtime dependencies and additional tools
RUN set -ex && \
    DEBIAN_FRONTEND=noninteractive apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y dnsutils logrotate ca-certificates ldnsutils libevent-2.1-7 libnghttp2-14 libexpat1 bsdmainutils && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Add your initialization commands here.
# These commands configure unbound, run unbound-control-setup to generate certificates,
# configure log rotation, etc.
RUN set -ex && \
    # Configure the unbound log file
    touch /opt/unbound/etc/unbound/unbound.log && \
    chown _unbound:_unbound /opt/unbound/etc/unbound/unbound.log && \
    chmod 644 /opt/unbound/etc/unbound/unbound.log && \
    \
    # Run unbound-control-setup to generate certificates if they are not present.
    # (Assuming unbound-control-setup generates unbound_control.key, unbound_control.pem,
    # unbound_server.key, and unbound_server.pem in /opt/unbound/etc/unbound/)
    if [ ! -f "/opt/unbound/etc/unbound/unbound_control.pem" ] || [ ! -f "/opt/unbound/etc/unbound/unbound_server.pem" ]; then \
      unbound-control-setup && \
      sleep 15; \
    else \
      echo "Certificates already exist, skipping unbound-control-setup."; \
    fi && \
    \
    # Configure logrotate for unbound logs (rotate daily, keep 7 copies)
    echo "/opt/unbound/etc/unbound/unbound.log { \
      daily \
      rotate 7 \
      copytruncate \
      compress \
      missingok \
      notifempty \
    }" > /etc/logrotate.d/unbound && \
    logrotate -f /etc/logrotate.d/unbound && \
    \
    # Optionally, add a cron job (this container won't run cron by default)
    echo "0 0 * * * /usr/sbin/logrotate /etc/logrotate.d/unbound" | crontab -

# Copy additional data (configuration files, scripts, etc.) from your build context.
# For example, if you have a startup script, copy it and make it executable.
COPY data/ / 

# Make sure your startup script is executable (if using one)
RUN chmod +x /unbound.sh

WORKDIR /opt/unbound/

ENV PATH /opt/unbound/sbin:"$PATH"

LABEL org.opencontainers.image.version=${UNBOUND_VERSION} \
      org.opencontainers.image.title="mvance/unbound-rpi" \
      org.opencontainers.image.description="a validating, recursive, and caching DNS resolver" \
      org.opencontainers.image.url="https://github.com/MatthewVance/unbound-docker-rpi" \
      org.opencontainers.image.vendor="Matthew Vance" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.source="https://github.com/MatthewVance/unbound-docker-rpi"

EXPOSE 53/tcp
EXPOSE 53/udp

HEALTHCHECK --interval=30s --timeout=30s --start-period=10s --retries=3 \
  CMD drill @127.0.0.1 cloudflare.com || exit 1

# Start the container using your startup script.
CMD ["/unbound.sh"]
