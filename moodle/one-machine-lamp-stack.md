**Final Result:**
![](assets/Pasted%20image%2020260617214415.png)

| Software      | Version         | Port         |
| ------------- | --------------- | ------------ |
| OS            | Ubuntu 24.04    | UFW: 80, 443 |
| Database      | MariaDB 12.3    | 3306         |
| Front         | Apche2 2.4.58   | 80, 443      |
| Back          | PHP/PHP-FPM 8.3 | Socket       |


> Make sure your server has access to the internet.

> Reference guide: https://docs.moodle.org/502/en/Step-by-step_Installation_Guide_for_Ubuntu

> Active moodle branches https://github.com/moodle/moodle/branches/active

> Moodle website: https://moodle.org/

> Tuning OS and services will be in a separate file.

---
# **First switch to `root` user:**

```bash
sudo su - root
```

---
# Setup Credentials And Variables:

> Every coming step depends on these variables.

> These variable are temporary and will be gone once you close the terminal.

> Change the values of these variables according to your situation and needs.

```bash

MOODLE_DATA_FOLDER=/var/moodle/moodledata
MOODLE_CODE_PATH=/var/www/html
MOODLE_CODE_FOLDER=${MOODLE_CODE_PATH}/moodle
COMPOSER_CACHE_FOLDER=/var/moodle/.cache/composer

MOODLE_VERSION_TAG=v5.2.1
PHP_VERSION=8.3


# Apache2:
MOODLE_APACHE_CONFIG_FILE=my-moodle.conf
APACHE_LOGS_FOLDER=/var/log/apache2
MOODLE_APACHE_ROOT_DOCUMENT=${MOODLE_CODE_FOLDER}/public
# PROTOCOL=https://
PROTOCOL=http://
# SERVER_ADDRESS=e-learning.mans.edu.eg
SERVER_ADDRESS=$(hostname -I | awk '{print $1}')
WEBSITE_PORT=80
WEBSITE_SECURE_PORT=443
APACHE_SSL_PRIVATE_KEY=/etc/ssl/private/private_key.key
APACHE_SSL_PUBLIC_KEY=/etc/ssl/certs/public_key.crt
APACHE_SSL_CA_FULL_CHAIN=/etc/ssl/certs/CA_fullChain.crt


LANGUAGE=en


# Site Name:
SITE_LONG_NAME="Mansourah University FCI."
SITE_SHORT_NAME="Mans. F.C.I."


# Admin:
MOODLE_ADMIN_USER_NAME=admin
MOODLE_ADMIN_PASSWORD=$(openssl rand -base64 32 | tr '+/=\\' '-_')
MOODLE_ADMIN_EMAIL=admin@example.com


# Database:
# MOODLE_DATABASE_USER=www-data
MOODLE_DATABASE_USER=moodle_user
MOODLE_DATABASE_USER_PASSWORD=$(openssl rand -base64 32 | tr '+/=\\' '-_')
MOODLE_DATABASE_NAME=moodle_db
MOODLE_DATABASE_TYPE=mariadb
MOODLE_DATABASE_SOCKET=/run/mysqld/mysqld.sock
MOODLE_DATABASE_HOST=localhost
MOODLE_DATABASE_PORT=3306
DATABASE_PREFEX=mdl_


# Print Passwords:
echo -e "\033[31mNote: Please Save This Password.\033[0m" # red text
echo -e "\033[34mMoodle Admin Password Is:\033[0m" # blue text
echo -e "\033[32m${MOODLE_ADMIN_PASSWORD}\033[0m" # green text
echo -e "\033[34mDatabase Password Is:\033[0m" # blue text
echo -e "\033[32m${MOODLE_DATABASE_USER_PASSWORD}\033[0m" # green text

```

---
---
# Configure Machine IP:

> **NOTE**: You can skip this step but next time you reboot the machine or after a while it might change the IP and then you have to modify the IP in `${MOODLE_APACHE_CONFIG_FILE}` and `config.php`.

```bash
vim /etc/netplan/*.yaml
```

> Default file is usually: `/etc/netplan/50-cloud-init.yaml`

> **IMPORTANT**:  Change Interface Name, EXP: `enp0s3`, `ens18`, and every parameter according to your network.

**Automatic:**
```bash
network:
  version: 2
  ethernets:
    enp0s3: # Interface Name
      dhcp4: yes # Automatic
```

**Static:**
```bash
network:
  version: 2
  ethernets:
    ens18: # Interface Name -> enp0s3, ens18
      dhcp4: no # Static
      addresses: [192.168.1.120/24] # IP/Subnet
      dhcp6: no # disable IPV6
      link-local: [] # disable IPV6
      routes:
      - to: "default"
        via: "192.168.1.1" # Gateway
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1] # DNS Servers
```

> Save and exit.

**Apply:**
```bash
netplan apply
```

> Now Open a new ssh session with the new IP.

---
---
# Database Installation:

[Install MariaDB Official Guide]([https://mariadb.org/download/?t=repo-config&d=24.04+%22noble%22&v=12.3&r_m=liquidtelecom](https://mariadb.org/download/?t=repo-config&d=24.04+%22noble%22&v=12.3&r_m=liquidtelecom))

[MariaDB Master Slave Replication](https://mariadb.com/docs/server/ha-and-performance/standard-replication/setting-up-replication)

---
#### Import the MariaDB repository key on your Ubuntu system:

```bash
apt install apt-transport-https curl -y
mkdir -p /etc/apt/keyrings
curl -o /etc/apt/keyrings/mariadb-keyring.asc 'https://mariadb.org/mariadb_release_signing_key.pgp'
```

#### Create `MariaDB 12.3` repository for `apt` package manager:

```bash
cat >> /etc/apt/sources.list.d/mariadb.sources << EOF
# MariaDB 12.3 repository list - created 2026-06-05 20:39 UTC
# https://mariadb.org/download/
X-Repolib-Name: MariaDB
Types: deb
# deb.mariadb.org is a dynamic mirror if your preferred mirror goes offline. See https://mariadb.org/mirrorbits/ for details.
# URIs: https://deb.mariadb.org/12.3/ubuntu
URIs: https://mariadb.mirror.liquidtelecom.com/repo/12.3/ubuntu
Suites: noble
Components: main main/debug
Signed-By: /etc/apt/keyrings/mariadb-keyring.asc
EOF
```

**Install the database:**
```bash
apt update
DEBIAN_FRONTEND=noninteractive apt install mariadb-server mariadb-client -y
```

> `DEBIAN_FRONTEND=noninteractive` makes you avoid this question:

![](assets/Pasted%20image%2020260614224912.png)

**Check status:**
```bash
systemctl status --no-pager mariadb
```

> Should be `active (running)`.

**Check version:**
```bash
mariadb --version
```

**Output:**
```bash
mariadb from 12.3.2-MariaDB, client 15.2 for debian-linux-gnu (x86_64) using  EditLine wrapper
```

---
#### Secure MariaDB:

```bash
mariadb-secure-installation
```

| #   | Question                              | Answer                   | Explanation                                                                                                                                                                             |
| --- | ------------------------------------- | ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Current password for root             | _Press Enter_ (if fresh) | If root user has no password(default) press enter.                                                                                                                                      |
| 2   | Switch to unix_socket authentication? | `y`                      | `root` user can only connect to the database server locally using `unix_socket` instead of a password, if `n` that means root can connect remotely with a password which is not secure. |
| 3   | Change root password?                 | `y`                      | In case you answer step `2` and `5` with `n` this changes the password for the root user for the database server only both locally and remotely.                                        |
| 4   | Remove anonymous users?               | `y`                      | Fresh MariaDB installs and creates an anonymous user that lets anyone log in without credentials.                                                                                       |
| 5   | Disallow root login remotely?         | `y`                      | Never let root login remotely.                                                                                                                                                          |
| 6   | Remove test database?                 | `y`                      | Remove unnecessary default database.                                                                                                                                                    |
| 7   | Reload privilege tables now?          | `y`                      | Changes above take effect immediately without restarting MariaDB.                                                                                                                       |

---
#### Create the `${MOODLE_DATABASE_USER}` and `${MOODLE_DATABASE_NAME}` moodle app will use:

> By default if you use `localhost` mariadb uses local unix socket, but if you use `127.0.0.1` mariadb uses TCP internal network.

**Create a new database with the `utf8mb4` character set and `utf8mb4_unicode_ci` collation:**
```bash
mariadb -e "CREATE DATABASE ${MOODLE_DATABASE_NAME} DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

**Show databases:**
```bash
mariadb -e "show databases;"
```

**Output:**
```bash
+--------------------+
| Database           |
+--------------------+
| information_schema |
| moodle_db          |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

**Create `${MOODLE_DATABASE_USER}` for moodle:**
```bash
mariadb -e "CREATE USER '${MOODLE_DATABASE_USER}'@'${MOODLE_DATABASE_HOST}' IDENTIFIED BY '${MOODLE_DATABASE_USER_PASSWORD}';"
```

**Show `${MOODLE_DATABASE_USER}` in database:**
```bash
mariadb -e "SELECT User, Host FROM mysql.user;"
```

**Output:**
```bash
+-------------+-----------+
| User        | Host      |
+-------------+-----------+
| mariadb.sys | localhost |
| moodle_user | localhost |
| mysql       | localhost |
| root        | localhost |
+-------------+-----------+
```


**Grant privileges on the database to `${MOODLE_DATABASE_USER}` to allow access locally:**
```bash
mariadb -e "GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, CREATE TEMPORARY TABLES, DROP, INDEX, ALTER ON ${MOODLE_DATABASE_NAME}.* TO '${MOODLE_DATABASE_USER}'@'${MOODLE_DATABASE_HOST}';"
```

**Male it take effect:**
```bash
mariadb -e "FLUSH PRIVILEGES;"
```

**Check that `${MOODLE_DATABASE_USER}` can access `${MOODLE_DATABASE_NAME}` from `${MOODLE_DATABASE_HOST}`:**
```bash
sudo mariadb -e "SELECT User, Host, Db FROM mysql.db ORDER BY User;"
```

```bash
+-------------+-----------+-----------+
| User        | Host      | Db        |
+-------------+-----------+-----------+
| moodle_user | localhost | moodle_db |
+-------------+-----------+-----------+
```

**Test connection for `root` user:**
```bash
mariadb -u root -p --socket=/run/mysqld/mysqld.sock
```

> Just press enter without password.

**Output:**
```bash
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 22
Server version: 12.3.2-MariaDB-ubu2404 mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> exit
Bye
```

**Test connection for `${MOODLE_DATABASE_USER}` user:**
```bash
mariadb -u ${MOODLE_DATABASE_USER} --password=${MOODLE_DATABASE_USER_PASSWORD}  --socket=/run/mysqld/mysqld.sock
```

**Output:**
```bash
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 23
Server version: 12.3.2-MariaDB-ubu2404 mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> exit
Bye
```

---
---
# Install Necessary Packages:

**Install required packages:**
```bash
apt install -y php${PHP_VERSION}-fpm php${PHP_VERSION}-cli php${PHP_VERSION}-curl php${PHP_VERSION}-zip php${PHP_VERSION}-gd php${PHP_VERSION}-xml php${PHP_VERSION}-intl  php${PHP_VERSION}-mbstring php${PHP_VERSION}-xmlrpc php${PHP_VERSION}-soap php${PHP_VERSION}-bcmath php${PHP_VERSION}-exif php${PHP_VERSION}-ldap php${PHP_VERSION}-mysql ufw vim graphviz aspell git clamav ghostscript composer libapache2-mod-fcgid apache2
```

> Will take sometime.

> `php*`: PHP packages needed by moodle.

> `graphviz`: For Moodle quiz/report diagrams.

> `aspell`: Used by applications to check spelling in various languages.

> `clamav`: Open-source antivirus engine. Used to scan uploaded files, email attachments, and storage for malware.

> `ghostscript`: Processes PostScript (PS) and PDF files. Can convert PDFs, generate thumbnails, merge PDFs, and render pages as images.

> `composer`: Dependency/package manager for PHP. Similar to npm for Node.js or pip for Python. Used to install PHP libraries defined in composer.json.

---
---
# Clone Moodle From Github:

```bash
git clone --branch ${MOODLE_VERSION_TAG} --single-branch --depth 1 https://github.com/moodle/moodle.git ${MOODLE_CODE_FOLDER}
```

> Clone a specific moodle version to `${MOODLE_CODE_FOLDER}`.

**Output:**
```bash
Cloning into '/var/www/html/moodle'...
remote: Enumerating objects: 71381, done.
remote: Counting objects: 100% (71381/71381), done.
remote: Compressing objects: 100% (34700/34700), done.
remote: Total 71381 (delta 33530), reused 68342 (delta 33019), pack-reused 0 (from 0)
Receiving objects: 100% (71381/71381), 94.72 MiB | 1.73 MiB/s, done.
Resolving deltas: 100% (33530/33530), done.
Note: switching to '63e16b757ca8fee05b672a27c23ee27cc8f9fabb'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -c with the switch command. Example:

  git switch -c <new-branch-name>

Or undo this operation with:

  git switch -

Turn off this advice by setting config variable advice.detachedHead to false

Updating files: 100% (61552/61552), done.
```

**Create a backup so that you wont have to download it again:**
```bash
cp -R ${MOODLE_CODE_FOLDER} ${MOODLE_CODE_FOLDER}-backup
```

**Take a look:**
```bash
ls -al ${MOODLE_CODE_PATH}
```

**Output:**
```bash
total 28
drwxr-xr-x  4 root root  4096 Jun 17 18:17 .
drwxr-xr-x  3 root root  4096 Jun 17 18:14 ..
-rw-r--r--  1 root root 10671 Jun 17 18:15 index.html
drwxr-xr-x 12 root root  4096 Jun 17 18:16 moodle
drwxr-xr-x 12 root root  4096 Jun 17 18:17 moodle-backup
```

---
---
# Create necessary Directories:

```bash
mkdir -p  ${MOODLE_DATA_FOLDER} ${COMPOSER_CACHE_FOLDER}
```

---
---
# Change Permissions and Ownership:

```bash

# Set Permissions:
find ${MOODLE_CODE_FOLDER} -type d -exec chmod 755 {} \;
find ${MOODLE_CODE_FOLDER} -type f -exec chmod 644 {} \;

find ${MOODLE_DATA_FOLDER} -type d -exec chmod 700 {} \;
find ${MOODLE_DATA_FOLDER} -type f -exec chmod 600 {} \;

chmod -R 750 ${COMPOSER_CACHE_FOLDER}

# Change ownership to www-data user and group:
chown -R www-data:www-data ${MOODLE_CODE_FOLDER} ${MOODLE_DATA_FOLDER} ${COMPOSER_CACHE_FOLDER}

```

> Will take sometime.

---
---
# PHP Configs:

> **Logs**: `/var/log/php${PHP_VERSION}-fpm.log`, `/var/log/php${PHP_VERSION}-fpm.log.1`

> Any Change should be applied to both `/etc/php/${PHP_VERSION}/fpm/php.ini` and `/etc/php/${PHP_VERSION}/cli/php.ini`.

---
#### Setup cache directory for compressor:

```bash
cd ${MOODLE_CODE_FOLDER}
sudo -u www-data COMPOSER_CACHE_DIR=${COMPOSER_CACHE_FOLDER} composer install --no-dev --classmap-authoritative
```

**Output looks something like this:**
```bash
...
  - Installing psr/http-server-handler (1.0.2): Extracting archive
  - Installing psr/http-server-middleware (1.0.2): Extracting archive
  - Installing nikic/fast-route (v1.3.0): Extracting archive
  - Installing slim/slim (4.13.0): Extracting archive
  - Installing spatie/php-cloneable (1.0.2): Extracting archive
Generating optimized autoload files
17 packages you are using are looking for funding.
Use the `composer fund` command to find out more!
```

#### Make sure `php${PHP_VERSION}-fpm` is working: (Optional)

```bash
sudo systemctl status --no-pager  php${PHP_VERSION}-fpm
```

> Should be `active (running)`.

**Enable and start if needed:**
```bash
sudo systemctl enable --now php${PHP_VERSION}-fpm
```

#### Recommended minimum php configs for moodle:
```bash
MAX_INPUT_VARS="10000" # 5000 minimum
POST_MAX="1024M"
UPLOAD_MAX="1024M"
FILE_UPLOADS="On" # On or Off
PHP_INI_FPM="/etc/php/${PHP_VERSION}/fpm/php.ini"
PHP_INI_CLI="/etc/php/${PHP_VERSION}/cli/php.ini"

# For /etc/php/${PHP_VERSION}/fpm/php.ini
sudo sed -i "s/^;*max_input_vars = .*/max_input_vars = ${MAX_INPUT_VARS}/" "$PHP_INI_FPM"
sudo sed -i "s/^post_max_size = .*/post_max_size = ${POST_MAX}/" "$PHP_INI_FPM"
sudo sed -i "s/^file_uploads = .*/file_uploads = ${FILE_UPLOADS}/" "$PHP_INI_FPM"
sudo sed -i "s/^upload_max_filesize = .*/upload_max_filesize = ${UPLOAD_MAX}/" "$PHP_INI_FPM"


# For /etc/php/${PHP_VERSION}/cli/php.ini
sudo sed -i "s/^;*max_input_vars = .*/max_input_vars = ${MAX_INPUT_VARS}/" "$PHP_INI_CLI"
sudo sed -i "s/^post_max_size = .*/post_max_size = ${POST_MAX}/" "$PHP_INI_CLI"
sudo sed -i "s/^file_uploads = .*/file_uploads = ${FILE_UPLOADS}/" "$PHP_INI_CLI"
sudo sed -i "s/^upload_max_filesize = .*/upload_max_filesize = ${UPLOAD_MAX}/" "$PHP_INI_CLI"

systemctl restart php${PHP_VERSION}-fpm.service
```

> **Important**: Edit variables in both `/etc/php/${PHP_VERSION}/fpm/php.ini` and `/etc/php/${PHP_VERSION}/cli/php.ini`.

> `max_input_vars`: number of fields in a quiz or grades sheet ...., 5000 for small ones, 10000 for large, 20000 for massive question banks and other things that need many fields.

> If `max_input_vars` is smaller than needed you will experience truncation while uploading a sheet with many fields for example.

> `upload_max_filesize`, `post_max_size` controls the upper size limit of files that can be uploaded to moodle.

> `upload_max_filesize` if = 0 that means upload is disabled from web GUI.

> `post_max_size` if = 0 that means upload is disabled.

> `file_uploads = Off` or `On` disable or enable file upload at all.

or 
#### Manually edit php variables: `upload_max_filesize`, `post_max_size`, `file_upload`, `max_input_vars` in the following files:

```bash
vim /etc/php/${PHP_VERSION}/fpm/php.ini
```

```bash
vim /etc/php/${PHP_VERSION}/cli/php.ini
```

---
#### Enable `cron.php`: (IMPORTANT)

> Call the cron.php in the moodle admin directory to run every minute. 

```bash
echo "* * * * * /usr/bin/php ${MOODLE_CODE_FOLDER}/admin/cli/cron.php >/dev/null" | sudo crontab -u www-data -
```

**Show cron jobs for www-data user:**
```bash
sudo -u www-data crontab -l
```

**Output:**
```bash
* * * * * /usr/bin/php /var/www/html/moodle/admin/cli/cron.php >/dev/null
```

> Runs every minute.

---
#### Configure moodle:

```bash
cd ${MOODLE_CODE_FOLDER}

chmod -R 0777 ${MOODLE_CODE_FOLDER}

sudo -u www-data /usr/bin/php${PHP_VERSION} ${MOODLE_CODE_FOLDER}/admin/cli/install.php \
--non-interactive \
--allow-unstable \
--lang="${LANGUAGE}" \
--wwwroot="${PROTOCOL}${SERVER_ADDRESS}" \
--dataroot="${MOODLE_DATA_FOLDER}" \
--dbtype="${MOODLE_DATABASE_TYPE}" \
--dbhost="${MOODLE_DATABASE_HOST}" \
--dbname="${MOODLE_DATABASE_NAME}" \
--dbuser="${MOODLE_DATABASE_USER}" \
--dbpass=${MOODLE_DATABASE_USER_PASSWORD} \
--dbsocket=${MOODLE_DATABASE_SOCKET} \
--dbport=${MOODLE_DATABASE_PORT} \
--fullname="${SITE_LONG_NAME}" \
--shortname="${SITE_SHORT_NAME}" \
--adminuser="${MOODLE_ADMIN_USER_NAME}" \
--adminpass="${MOODLE_ADMIN_PASSWORD}" \
--adminemail="${MOODLE_ADMIN_EMAIL}" \
--summary="" \
--agree-license
```

**See all options:**
```bash
sudo -u www-data /usr/bin/php ${MOODLE_CODE_FOLDER}/admin/cli/install.php --help
```

> If your are using a `load balancer` or `NAT` change `--wwwroot` value to match the address or the IP that you will be typing in the web-browser, EXP: `https://e-learning.masn.edu.eg`, `http://e-learning.masn.edu.eg`, `https://46.58.71.31`, `http://46.58.71.31`. 

> If you use a custom port to access moodle server add it to the `--wwwroot`, EXP: `https://e-learning.masn.edu.eg:9999`, `http://e-learning.masn.edu.eg:9999`, `https://46.58.71.31:9999`, `http://46.58.71.31:9999`.

**Output looks something like this:**
```bash
...
-->factor_totp
++ Success (0.03 seconds) ++
-->factor_webauthn
++ Success (0.03 seconds) ++
-->upgrade_noncore()
++ Success (0.88 seconds) ++
Installation completed successfully.
```

**Take a look at the configs:**
```bash
cat ${MOODLE_CODE_FOLDER}/config.php
```

**Output:**
```bash
<?php  // Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = 'localhost';
$CFG->dbname    = 'moodle_db';
$CFG->dbuser    = 'moodle_user';
$CFG->dbpass    = 'a15My-lz1wHBXNiVadBIcc7IyujlMQR0gehb3FWuH9U_';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => 3306,
  'dbsocket' => '/run/mysqld/mysqld.sock',
  'dbcollation' => 'utf8mb4_unicode_ci',
);

$CFG->wwwroot   = 'http://192.168.1.120';
$CFG->dataroot  = '/var/moodle/moodledata';
$CFG->admin     = 'admin';

$CFG->directorypermissions = 02777;

require_once(__DIR__ . '/lib/setup.php');

// There is no php closing tag in this file,
// it is intentional because it prevents trailing whitespace problems!
```

---
---
# Change Permissions and Ownership:

```bash

# Set Permissions:
find ${MOODLE_CODE_FOLDER} -type d -exec chmod 755 {} \;
find ${MOODLE_CODE_FOLDER} -type f -exec chmod 644 {} \;

find ${MOODLE_DATA_FOLDER} -type d -exec chmod 700 {} \;
find ${MOODLE_DATA_FOLDER} -type f -exec chmod 600 {} \;

chmod -R 750 ${COMPOSER_CACHE_FOLDER}

# Change ownership to www-data user and group:
chown -R www-data:www-data ${MOODLE_CODE_FOLDER} ${MOODLE_DATA_FOLDER} ${COMPOSER_CACHE_FOLDER}

```

> Will take sometime.

---
---
# Apache Configs:

#### Make sure `apache2` is working: (Optional)

```bash
systemctl status --no-pager apache2
```

> Should be `active (running)`.

**Enable and start apache if needed:**
```bash
systemctl enable --now apache2
```

---
#### Enable required `apache2` modules for `PHP-FPM` and `rewriting`:

```bash
a2enmod proxy_fcgi setenvif rewrite
systemctl restart apache2
```

**Output:**
```bash
Considering dependency proxy for proxy_fcgi:
Enabling module proxy.
Enabling module proxy_fcgi.
Module setenvif already enabled
Enabling module rewrite.
To activate the new configuration, you need to run:
  systemctl restart apache2
```

---
## Create site configuration file in `/etc/apache2/sites-available`:

### (Option_1) No SSL/TLS -> Not secure (just for testing):

#### Make sure ssl mod is disabled:

```bash
a2dismod ssl
```

**Output:**
```bash
Module ssl already disabled
```

#### Create the actual site configuration file:

> Comment `ServerAlias` if you are going to use IP.

**Add the following:**
```bash
tee /etc/apache2/sites-available/${MOODLE_APACHE_CONFIG_FILE} > /dev/null <<EOF
<VirtualHost *:${WEBSITE_PORT}>
    # Domain name without http://
    # EXP: Domain Name: mans.edu.eg or IP: 192.168.1.120
    ServerName ${SERVER_ADDRESS}
    ServerAlias www.${SERVER_ADDRESS}

    DocumentRoot ${MOODLE_APACHE_ROOT_DOCUMENT}
    
    <Directory /var/www/html/moodle>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
        DirectoryIndex index.php index.html

        # Enable fallback routing for URLs not matching files/directories
        FallbackResource /r.php
    </Directory>

    # PHP-FPM ${PHP_VERSION} FastCGI handler via socket
    <FilesMatch "\.php$">
        SetHandler "proxy:unix:/run/php/php${PHP_VERSION}-fpm.sock|fcgi://localhost/"
    </FilesMatch>

    # Log files
    ErrorLog ${APACHE_LOGS_FOLDER}/error.log
    CustomLog ${APACHE_LOGS_FOLDER}/access.log combined
    
    # all relevant timeouts 
    Timeout 300
    ProxyTimeout 300

    # FastCGI/PHP-FPM: 
    FcgidIOTimeout 300
    FcgidConnectTimeout 300
    FcgidBusyTimeout 300
</VirtualHost>
EOF
```


**Take a look at the `${MOODLE_APACHE_CONFIG_FILE`} file:**
```bash
cat /etc/apache2/sites-available/${MOODLE_APACHE_CONFIG_FILE}
```

**Output:**
```bash
<VirtualHost *:80>
    # Domain name without http://
    # EXP: Domain Name: mans.edu.eg or IP: 192.168.1.120
    ServerName 192.168.1.120
    ServerAlias www.192.168.1.120

    DocumentRoot /var/www/html/moodle/public
    
    <Directory /var/www/html/moodle>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
        DirectoryIndex index.php index.html

        # Enable fallback routing for URLs not matching files/directories
        FallbackResource /r.php
    </Directory>

    # PHP-FPM 8.3 FastCGI handler via socket
    <FilesMatch "\.php$">
        SetHandler "proxy:unix:/run/php/php8.3-fpm.sock|fcgi://localhost/"
    </FilesMatch>

    # Log files
    ErrorLog /var/log/apache2/error.log
    CustomLog /var/log/apache2/access.log combined
    
    # all relevant timeouts 
    Timeout 300
    ProxyTimeout 300

    # FastCGI/PHP-FPM: 
    FcgidIOTimeout 300
    FcgidConnectTimeout 300
    FcgidBusyTimeout 300
</VirtualHost>
```
### **End of: (Option_1)**

---
### (Option_2) With SSL/TLS enabled:

[Add Apache SSL Guide Link](https://httpd.apache.org/docs/current/ssl/ssl_howto.html)
#### Enable apache2 over ssl: (IMPORTANT)

```bash
a2enmod ssl
```

#### Create the actual site configuration file:

> Comment `ServerAlias` if you are going to use IP.

> HTTP port:`80` requests are redirected to HTTPS port:`443`.

**Add the following:**
```bash
tee /etc/apache2/sites-available/${MOODLE_APACHE_CONFIG_FILE} > /dev/null <<EOF
<VirtualHost *:${WEBSITE_PORT}>
    # Domain name without http://
    # EXP: mans.edu.eg
    ServerName ${SERVER_ADDRESS}
    Redirect / https://${SERVER_ADDRESS}
</VirtualHost>

<VirtualHost *:${WEBSITE_SECURE_PORT}>
    # Domain name without http://
    # EXP: Domain Name: mans.edu.eg or IP: 192.168.1.120
    ServerName ${SERVER_ADDRESS}
    ServerAlias www.${SERVER_ADDRESS}

    #DocumentRoot  ${MOODLE_APACHE_ROOT_DOCUMENT}

    <Directory /var/www/html/moodle>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
        DirectoryIndex index.php index.html

        # Enable fallback routing for URLs not matching files/directories
        FallbackResource /r.php
    </Directory>

    # PHP-FPM ${PHP_VERSION} FastCGI handler via socket
    <FilesMatch "\.php$">
        SetHandler "proxy:unix:/run/php/php${PHP_VERSION}- fpm.sock|fcgi://localhost/"
    </FilesMatch>

    # Log files
    ErrorLog ${APACHE_LOGS_FOLDER}/error.log
    CustomLog ${APACHE_LOGS_FOLDER}/access.log combined
    
    # all relevant timeouts 
    Timeout 300
    ProxyTimeout 300

    # FastCGI/PHP-FPM: 
    FcgidIOTimeout 300
    FcgidConnectTimeout 300
    FcgidBusyTimeout 300
    
    # IMPORTANT: Make sure you have these files in these locations.
    SSLCertificateKeyFile ${APACHE_SSL_PRIVATE_KEY}
    SSLCertificateFile ${APACHE_SSL_PUBLIC_KEY}
    SSLCertificateChainFile ${APACHE_SSL_CA_FULL_CHAIN}
</VirtualHost>
EOF
```

**Take a look at the `${MOODLE_APACHE_CONFIG_FILE`} file:**
```bash
cat /etc/apache2/sites-available/${MOODLE_APACHE_CONFIG_FILE}
```
### **End of: (Option_2)**

---
#### Make `${MOODLE_APACHE_CONFIG_FILE}` as default website for apache and disable `000-default.conf`:

```bash
# Enable the new Moodle site and disable the default site if desired
a2ensite ${MOODLE_APACHE_CONFIG_FILE}
a2dissite 000-default.conf
# Restart Apache to apply all changes
systemctl restart apache2
```

#### Remove default page: (Optional)

```bash
rm /var/www/html/index.html
```

---
#### Check that configurations are correct: (Optional)

```bash
sudo apache2ctl configtest
```

**Output:**
```output
Syntax OK
```

---
---
# Test the website:

> EXP: http://192.168.1.120

> In your browser type the url in `--wwwroot="${PROTOCOL}${SERVER_ADDRESS}"`, EXP: `https://e-learning.masn.edu.eg`, `http://e-learning.masn.edu.eg`, `https://46.58.71.31`, `http://46.58.71.31`, `https://e-learning.masn.edu.eg:9999`, `http://e-learning.masn.edu.eg:9999`, `https://46.58.71.31:9999`, `http://46.58.71.31:9999`. 

> For some reason login doesn't work from the first time.

![](assets/Pasted%20image%2020260617214415.png)


![](assets/Pasted%20image%2020260617112340.png)


![](assets/Pasted%20image%2020260617214835.png)

---
---
# **Machine Firewall Configs:**

#### Less secure:

```bash
ufw allow 80 comment "port for http"
ufw allow 22 comment "port for ssh"
```

**Output:**
```bash
ufw allow 22 comment "port for ssh"
Rules updated
Rules updated (v6)
Rules updated
Rules updated (v6)
```
#### **More secure:**

```bash
ufw allow from <IP> to any port 80 comment "port for http"
```

```bash
ufw allow from <IP> to any port 443 comment "port for https"
```

```bash
ufw allow from <YOUR_IP> to any port <SSH_PORT> comment "port for ssh"
```

> Replace `<IP>` with the IP of the machine you are using to access moodle if you want to enable the firewall, or the IP of the load balancer if you are using one.

> Replace `<YOUR_IP>` with the IP of the machine you are using to ssh into the server.

> Replace `<SSH_PORT>`.

---
#### **Enable firewall:**

```bash
ufw enable
```

> Type `y`.

**Output:**
```bash
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```

---
#### Show rules numbered:

```bash
ufw status numbered
```

```bash
Status: active

     To                     Action      From
     --                     ------      ----
[ 1] 80                     ALLOW IN    Anywhere               # port for http
[ 2] 22                     ALLOW IN    Anywhere               # port for ssh
[ 3] 80 (v6)                ALLOW IN    Anywhere (v6)          # port for http
[ 4] 22 (v6)                ALLOW IN    Anywhere (v6)          # port for ssh
```

---
#### Delete a rule: (Optional)

```bash
ufw delete <NUMBER>
```

---
