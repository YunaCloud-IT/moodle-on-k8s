FROM php:8.1-apache

# LAYER 1: System Dependencies & PHP Extensions
RUN apt-get update && apt-get install -y \
    libpng-dev \
    libjpeg-dev \
    libxml2-dev \
    libicu-dev \
    libzip-dev \
    libpq-dev \
    unzip \
    git \
    --no-install-recommends \
    && rm -rf /var/lib/apt/lists/* \
    && docker-php-ext-configure gd --with-jpeg \
    && docker-php-ext-install -j$(nproc) pgsql pdo_pgsql gd intl soap zip opcache exif

# LAYER 2: Application Setup, Security & Non-Root Configuration
RUN set -ex; \
    # ---------------------------------------------------------
    # A. Download & Verify Moodle (Immutable)
    # ---------------------------------------------------------
    MOODLE_URL="https://download.moodle.org/download.php/direct/stable403/moodle-latest-403.tgz"; \
    \
    curl -L "$MOODLE_URL" -o moodle.tgz; \
    \
    # Verify we actually downloaded a file, not an HTML error page
    if [ $(wc -c < moodle.tgz) -lt 1000000 ]; then \
        echo "Error: moodle.tgz is too small. Likely a 404 or download failed."; \
        cat moodle.tgz; \
        exit 1; \
    fi; \
    \
    # Extract
    tar -xzf moodle.tgz -C /var/www/html --strip-components=1; \
    rm moodle.tgz; \
    \
    # ---------------------------------------------------------
    # B. Create Non-Root User 'moodle'
    # ---------------------------------------------------------
    groupadd -g 1001 moodle; \
    useradd -u 1001 -g moodle -m -s /bin/bash moodle; \
    \
    # ---------------------------------------------------------
    # C. Configure Apache for Port 8080 (Non-Root Binding)
    # ---------------------------------------------------------
    # Change Listen port in ports.conf and default site config
    sed -i 's/80/8080/g' /etc/apache2/sites-available/000-default.conf /etc/apache2/ports.conf; \
    # Update Apache envvars to run as the moodle user (optional but safer)
    sed -i 's/export APACHE_RUN_USER=www-data/export APACHE_RUN_USER=moodle/g' /etc/apache2/envvars; \
    sed -i 's/export APACHE_RUN_GROUP=www-data/export APACHE_RUN_GROUP=moodle/g' /etc/apache2/envvars; \
    \
    # ---------------------------------------------------------
    # D. Hardened PHP Configuration
    # ---------------------------------------------------------
    { \
        echo 'file_uploads = On'; \
        echo 'memory_limit = 512M'; \
        echo 'upload_max_filesize = 100M'; \
        echo 'post_max_size = 100M'; \
        echo 'max_execution_time = 600'; \
        echo 'max_input_vars = 5000'; \
        echo 'expose_php = Off'; \
        echo 'opcache.enable = 1'; \
        echo 'opcache.memory_consumption = 128'; \
        echo 'opcache.max_accelerated_files = 10000'; \
        echo 'opcache.revalidate_freq = 60'; \
    } > /usr/local/etc/php/conf.d/moodle-hardened.ini; \
    \

    # ---------------------------------------------------------
    # F. Install Prometheus Metrics Plugin
    # ---------------------------------------------------------
    PROMETHEUS_PLUGIN_URL="https://github.com/catalyst/moodle-local_prometheus/archive/refs/heads/master.zip"; \
    curl -L "$PROMETHEUS_PLUGIN_URL" -o prometheus.zip; \
    unzip prometheus.zip; \
    mv moodle-local_prometheus-master /var/www/html/local/prometheus; \
    rm prometheus.zip; \

    # ---------------------------------------------------------
    # E. Permissions & Directory Setup
    # ---------------------------------------------------------
    mkdir -p /var/www/moodledata; \
    # Create Apache directories that might be missing and grant ownership
    mkdir -p /var/run/apache2 /var/lock/apache2 /var/log/apache2; \
    \
    # Transfer ownership to 'moodle' user
    chown -R moodle:moodle /var/www/moodledata; \
    chown -R moodle:moodle /var/www/html; \
    chown -R moodle:moodle /var/run/apache2 /var/lock/apache2 /var/log/apache2; \
    \
    # Cleanup permissions (Secure web root)
    chmod 755 /var/www/html; \
    chmod 770 /var/www/moodledata

# Switch to non-root user
USER moodle

# Expose the new non-privileged port
EXPOSE 8080

# The base image's CMD ["apache2-foreground"] will now run as 'moodle'
