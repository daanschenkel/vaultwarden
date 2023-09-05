# syntax=docker/dockerfile:1

# This file was generated using a Jinja2 template.
# Please make your changes in `Dockerfile.j2` and then `make` the individual Dockerfiles.
# Using multistage build:
# 	https://docs.docker.com/develop/develop-images/multistage-build/
# 	https://whitfin.io/speeding-up-rust-docker-builds/

########################## BUILD IMAGE  ##########################
FROM docker.io/library/rust:1.72.0-bookworm as build

# Get both version of the front-end
RUN wget -c https://github.com/Timshel/oidc_web_builds/releases/latest/download/oidc_button_web_vault.tar.gz -O - \
    | tar -xz \
    ; mv web-vault web-vault_button
RUN wget -c https://github.com/Timshel/oidc_web_builds/releases/latest/download/oidc_override_web_vault.tar.gz -O - \
    | tar -xz \
    ; mv web-vault web-vault_override

# Build time options to avoid dpkg warnings and help with reproducible builds.
ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8 \
    TZ=UTC \
    TERM=xterm-256color \
    CARGO_HOME="/root/.cargo" \
    REGISTRIES_CRATES_IO_PROTOCOL=sparse \
    USER="root"

# Create CARGO_HOME folder and don't download rust docs
RUN mkdir -pv "${CARGO_HOME}" \
    && rustup set profile minimal

# Install build dependencies for the armhf architecture
RUN dpkg --add-architecture armhf \
    && apt-get update \
    && apt-get install -y \
        --no-install-recommends \
        gcc-arm-linux-gnueabihf \
        libc6-dev:armhf \
        linux-libc-dev:armhf \
        libmariadb-dev:armhf \
        libmariadb-dev-compat:armhf \
        libmariadb3:armhf \
        libpq-dev:armhf \
        libpq5:armhf \
        libssl-dev:armhf \
    #
    # Make sure cargo has the right target config
    && echo '[target.armv7-unknown-linux-gnueabihf]' >> "${CARGO_HOME}/config" \
    && echo 'linker = "arm-linux-gnueabihf-gcc"' >> "${CARGO_HOME}/config" \
    && echo 'rustflags = ["-L/usr/lib/arm-linux-gnueabihf"]' >> "${CARGO_HOME}/config"

# Set arm specific environment values
ENV CC_armv7_unknown_linux_gnueabihf="/usr/bin/arm-linux-gnueabihf-gcc" \
    CROSS_COMPILE="1" \
    OPENSSL_INCLUDE_DIR="/usr/include/arm-linux-gnueabihf" \
    OPENSSL_LIB_DIR="/usr/lib/arm-linux-gnueabihf"

# Creates a dummy project used to grab dependencies
RUN USER=root cargo new --bin /app
WORKDIR /app

# Copies over *only* your manifests and build files
COPY ./Cargo.* ./
COPY ./rust-toolchain.toml ./rust-toolchain.toml
COPY ./build.rs ./build.rs

RUN rustup target add armv7-unknown-linux-gnueabihf

# Configure the DB ARG as late as possible to not invalidate the cached layers above
ARG DB=sqlite,mysql,postgresql

# Builds your dependencies and removes the
# dummy project, except the target folder
# This folder contains the compiled dependencies
RUN cargo build --features ${DB} --release --target=armv7-unknown-linux-gnueabihf \
    && find . -not -path "./target*" -delete

# Copies the complete project
# To avoid copying unneeded files, use .dockerignore
COPY . .

# Make sure that we actually build the project
RUN touch src/main.rs

# Builds again, this time it'll just be
# your actual source files being built
RUN cargo build --features ${DB} --release --target=armv7-unknown-linux-gnueabihf

######################## RUNTIME IMAGE  ########################
# Create a new stage with a minimal image
# because we already have a binary built
FROM docker.io/balenalib/armv7hf-debian:bookworm

ENV ROCKET_PROFILE="release" \
    ROCKET_ADDRESS=0.0.0.0 \
    ROCKET_PORT=80

RUN [ "cross-build-start" ]

# Create data folder and Install needed libraries
RUN mkdir /data \
    && apt-get update && apt-get install -y \
    --no-install-recommends \
    ca-certificates \
    curl \
    libmariadb-dev-compat \
    libpq5 \
    openssl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN [ "cross-build-end" ]

VOLUME /data
EXPOSE 80
EXPOSE 3012

# Copies the files from the context (Rocket.toml file and web-vault)
# and the binary from the "build" stage to the current stage
WORKDIR /
COPY --from=build /web-vault_button ./web-vault_button
COPY --from=build /web-vault_override ./web-vault_override
COPY --from=build /app/target/armv7-unknown-linux-gnueabihf/release/vaultwarden .

COPY docker/healthcheck.sh /healthcheck.sh
COPY docker/start.sh /start.sh

HEALTHCHECK --interval=60s --timeout=10s CMD ["/healthcheck.sh"]

CMD ["/start.sh"]
