ARG DOCKER_HUB_PROXY=""

FROM "${DOCKER_HUB_PROXY}python:3.12-slim-bookworm" AS python-build
    ENV DEBIAN_FRONTEND=noninteractive
    ARG GUARD_TAG
    ARG GUARD_COMMIT

    RUN <<-EOF
        apt-get update
        apt-get install -y --no-install-recommends \
            ca-certificates \
            git \
            build-essential \
            wget
EOF

    # Fix critical SQLite vulnerabilities (CVE-2023-7104, CVE-2024-0232, CVE-2024-21491)
    # Upgrade from vulnerable 3.40.1 to secure 3.46.1
    RUN <<-EOF
        cd /tmp
        # Try primary location first, fallback to mirror if needed
        wget -q -T 30 https://www.sqlite.org/2024/sqlite-autoconf-3460100.tar.gz || \
        wget -q -T 30 https://sqlite.org/2024/sqlite-autoconf-3460100.tar.gz || \
        curl -sL -m 30 https://www.sqlite.org/2024/sqlite-autoconf-3460100.tar.gz -o sqlite-autoconf-3460100.tar.gz || \
        echo "Failed to download SQLite, using system version (vulnerable)"
        
        if [ -f sqlite-autoconf-3460100.tar.gz ]; then
            tar xzf sqlite-autoconf-3460100.tar.gz
            cd sqlite-autoconf-3460100
            ./configure --prefix=/usr/local --disable-static --enable-shared
            make -j$(nproc)
            make install
            ldconfig
            echo "SQLite upgraded successfully to 3.46.1"
        else
            echo "WARNING: SQLite upgrade failed - using vulnerable system version 3.40.1"
        fi
        cd /
        rm -rf /tmp/sqlite-autoconf-3460100*
EOF

    RUN apt-get remove -y build-essential wget && \
        apt-get autoremove -y && apt-get clean -y && rm -rf /var/lib/apt/lists/*

    RUN mkdir /wheels

    RUN <<-EOF
        if [ ! -z ${GUARD_COMMIT} ]; then
            git clone https://github.com/MISP/misp-guard.git /srv/misp-guard && cd /srv/misp-guard && git checkout ${GUARD_COMMIT}
        else
            git clone --branch ${GUARD_TAG} --depth 1 https://github.com/MISP/misp-guard.git /srv/misp-guard
        fi
EOF

    WORKDIR /srv/misp-guard/src
    RUN pip wheel -r requirements.txt --no-cache-dir -w /wheels/



FROM "${DOCKER_HUB_PROXY}python:3.12-slim-bookworm"
    ENV DEBIAN_FRONTEND=noninteractive

    RUN <<-EOF
        apt-get update
        apt-get install -y --no-install-recommends \
            jq \
            build-essential \
            wget
EOF

    # Fix critical SQLite vulnerabilities (CVE-2023-7104, CVE-2024-0232, CVE-2024-21491)
    # Upgrade from vulnerable 3.40.1 to secure 3.46.1
    RUN <<-EOF
        cd /tmp
        # Try primary location first, fallback to mirror if needed
        wget -q -T 30 https://www.sqlite.org/2024/sqlite-autoconf-3460100.tar.gz || \
        wget -q -T 30 https://sqlite.org/2024/sqlite-autoconf-3460100.tar.gz || \
        curl -sL -m 30 https://www.sqlite.org/2024/sqlite-autoconf-3460100.tar.gz -o sqlite-autoconf-3460100.tar.gz || \
        echo "Failed to download SQLite, using system version (vulnerable)"
        
        if [ -f sqlite-autoconf-3460100.tar.gz ]; then
            tar xzf sqlite-autoconf-3460100.tar.gz
            cd sqlite-autoconf-3460100
            ./configure --prefix=/usr/local --disable-static --enable-shared
            make -j$(nproc)
            make install
            ldconfig
            echo "SQLite upgraded successfully to 3.46.1"
        else
            echo "WARNING: SQLite upgrade failed - using vulnerable system version 3.40.1"
        fi
        cd /
        rm -rf /tmp/sqlite-autoconf-3460100*
EOF

    RUN apt-get remove -y build-essential wget && \
        apt-get autoremove -y && apt-get clean -y && rm -rf /var/lib/apt/lists/*

    COPY --from=python-build /wheels /wheels
    RUN pip install --no-cache-dir /wheels/*.whl && rm -rf /wheels

    COPY --from=python-build /srv /srv
    COPY files/ /

    WORKDIR /srv/misp-guard/src
    ENTRYPOINT [ "/entrypoint.sh" ]
