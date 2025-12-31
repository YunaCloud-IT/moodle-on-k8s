# ==============================================================================
# Moodle Production Dockerfile (Non-Root, Redis-Ready, High-Performance)
# ==============================================================================
FROM php:8.2-apache

# ------------------------------------------------------------------------------
# LAYER 1: System Dependencies & PHP Extensions
# ------------------------------------------------------------------------------
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpng-dev \
    libzip-dev \
    libicu-dev \
    libxml2-dev \
    libpq-dev \
    ghostscript \
    unzip \
    curl \
    git \
    autoconf \
    build-essential \
    \
    # 1. Install Standard Moodle Extensions
    && docker-php-ext-install -j$(nproc) \
        gd \
        intl \
        soap \
        zip \
        opcache \
        pgsql \
        pdo_pgsql \
    \
    # 2. Install Redis (Crucial for Production Caching)
    && pecl install redis \
    && docker-php-ext-enable redis \
    \
    # 3. Cleanup to keep image slim
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# ------------------------------------------------------------------------------
# LAYER 2: Configuration & Permission Setup (Runs as Root)
# ------------------------------------------------------------------------------
RUN set -ex; \
    # 1. Create Non-Root User
    groupadd -g 1001 moodle; \
    useradd -u 1001 -g moodle -m -s /bin/bash moodle; \
    \
    # 2. Configure Apache for Port 8080 (Non-Root Binding)
    sed -i 's/80/8080/g' /etc/apache2/sites-available/000-default.conf /etc/apache2/ports.conf; \
    sed -i 's/export APACHE_RUN_USER=www-data/export APACHE_RUN_USER=moodle/g' /etc/apache2/envvars; \
    sed -i 's/export APACHE_RUN_GROUP=www-data/export APACHE_RUN_GROUP=moodle/g' /etc/apache2/envvars; \
    echo "ServerName localhost" >> /etc/apache2/apache2.conf; \
    \
    # 3. Hardened PHP Configuration (Performance & Limits)
    { \
        echo 'file_uploads = On'; \
        echo 'memory_limit = 512M'; \
        echo 'upload_max_filesize = 100M'; \
        echo 'post_max_size = 100M'; \
        echo 'max_execution_time = 600'; \
        echo 'max_input_vars = 5000'; \
        echo 'expose_php = Off'; \
        # Opcache Tuning for Production
        echo 'opcache.enable = 1'; \
        echo 'opcache.memory_consumption = 256'; \
        echo 'opcache.max_accelerated_files = 20000'; \
        echo 'opcache.revalidate_freq = 60'; \
        echo 'opcache.validate_timestamps = 1'; \
        echo 'opcache.interned_strings_buffer = 16'; \
    } > /usr/local/etc/php/conf.d/moodle-hardened.ini; \
    \
    # 4. Prepare Directories
    mkdir -p /var/www/moodledata; \
    mkdir -p /var/run/apache2 /var/lock/apache2 /var/log/apache2; \
    chown -R moodle:moodle /var/www/moodledata; \
    chown -R moodle:moodle /var/www/html; \
    chown -R moodle:moodle /var/run/apache2 /var/lock/apache2 /var/log/apache2; \
    chmod 755 /var/www/html; \
    chmod 770 /var/www/moodledata

# ------------------------------------------------------------------------------
# LAYER 3: Application Code (Runs as 'moodle' User)
# ------------------------------------------------------------------------------
USER 1001

# A. Install Moodle Core
RUN set -ex; \
    MOODLE_URL="https://download.moodle.org/download.php/direct/stable403/moodle-latest-403.tgz"; \
    curl -L "$MOODLE_URL" -o moodle.tgz; \
    if [ $(wc -c < moodle.tgz) -lt 1000000 ]; then \
        echo "Error: moodle.tgz is too small."; exit 1; \
    fi; \
    tar -xzf moodle.tgz -C /var/www/html --strip-components=1; \
    rm moodle.tgz

# B. Install Prometheus Metrics Plugin
RUN set -ex; \
    mkdir -p /var/www/html/local/prometheus; \
    PROMETHEUS_PLUGIN_URL="https://github.com/Vidalia/moodle-local_prometheus/archive/HEAD.tar.gz"; \
    curl -fL "$PROMETHEUS_PLUGIN_URL" | tar -xz -C /var/www/html/local/prometheus --strip-components=1

# ------------------------------------------------------------------------------
# LAYER 4: Final Metadata
# ------------------------------------------------------------------------------
WORKDIR /var/www/html
EXPOSE 8080
CMD ["apache2-foreground"]
