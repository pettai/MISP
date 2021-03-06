# INSTALLATION INSTRUCTIONS for RHEL 7.x and CentOS 7.x
-------------------------

### -1/ Installer and Manual install instructions

Make sure you are reading the parsed version of this Document. When in doubt [click here](https://misp.github.io/MISP/INSTALL.rhel7/).

!!! warning
    In the **future**, to install MISP on a fresh RHEL 7 install all you need to do is:

    ```bash
    # Please check the installer options first to make the best choice for your install
    curl -fsSL https://raw.githubusercontent.com/MISP/MISP/2.4/INSTALL/INSTALL.debian.sh | bash -s

    # This will install MISP Core and misp-modules (recommended)
    curl -fsSL https://raw.githubusercontent.com/MISP/MISP/2.4/INSTALL/INSTALL.debian.sh | bash -s -- -c -M
    ```
    **The above does NOT work yet**

## 0/ Overview and Assumptions

{!generic/rhelVScentos.md!}

!!! warning
    The core MISP team cannot verify if this guide is working or not. Please help us in keeping it up to date and accurate.
    Thus we also have difficulties in supporting RHEL issues but will do a best effort on a similar yet slightly different setup.

!!! notice
    Maintenance for CentOS 7 will end on: June 30th, 2024 [Source[0]](https://wiki.centos.org/About/Product) [Source[1]](https://linuxlifecycle.com/)
    CentOS 7.6-1810 [NetInstallURL](http://mirror.centos.org/centos/7.6.1810/os/x86_64/)

This document details the steps to install MISP on Red Hat Enterprise Linux 7.x (RHEL 7.x) and CentOS 7.x.
At time of this writing it was tested on versions 7.6 for both.
This is a joint RHEL/CentOS install guide. The authors tried to make it contextually evident what applies to which flavor.

The following assumptions with regard to this installation have been made.

### 0.1/ A valid support agreement allowing the system to register to the Red Hat Customer Portal and receive updates
### 0.2/ The ability to enable additional RPM repositories, specifically the EPEL and Software Collections (SCL) repos
### 0.3/ This system will have direct or proxy access to the Internet for updates. Or connected to a Red Hat Satellite Server
### 0.4/ This document will bootstrap a MISP instance running over HTTPS. A full test of all features have yet to be done. [The following GitHub issue](https://github.com/MISP/MISP/issues/4084) details some shortcomings.

{!generic/globalVariables.md!}

```bash
# <snippet-begin 0_RHEL_PHP_INI.sh>
# RHEL/CentOS Specific
RUN_PHP='/usr/bin/scl enable rh-php72'
RUN_PYTHON='/usr/bin/scl enable rh-python36'
SUDO_WWW='sudo -H -u apache'
WWW_USER='apache'

PHP_INI=/etc/opt/rh/rh-php72/php.ini
# <snippet-end 0_RHEL_PHP_INI.sh>
```

!!! note
		For fresh installs the following tips might be handy.<br />
		Allow ssh to pass the firewall on the CLI
		```bash
		firewall-cmd --zone=public --add-port=22/tcp --permanent
		firewall-cmd --reload
		```
		<br />
		To quickly make sure if NetworkManager handles your network interface on boot, check in the following location:
		```
		/etc/sysconfig/network-scripts/ifcfg-*
		```

# 1/ OS Install and additional repositories

## 1.1/ Complete a minimal RHEL/CentOS installation, configure IP address to connect automatically.

## 1.2/ Configure system hostname (if not done during install)
```bash
sudo hostnamectl set-hostname misp.local # Your choice, in a production environment, it's best to use a FQDN
```

## 1.3/ **[RHEL]** Register the system for updates with Red Hat Subscription Manager
```bash
# <snippet-begin 0_RHEL_register.sh>
sudo subscription-manager register --auto-attach # register your system to an account and attach to a current subscription
# <snippet-end 0_RHEL_register.sh>
```

## 1.4/ **[RHEL]** Enable the optional, extras and Software Collections (SCL) repos
```bash
# <snippet-begin 0_RHEL_SCL.sh>
sudo subscription-manager refresh 
sudo subscription-manager repos --enable rhel-7-server-optional-rpms
sudo subscription-manager repos --enable rhel-7-server-extras-rpms
sudo subscription-manager repos --enable rhel-server-rhscl-7-rpms
# <snippet-end 0_RHEL_SCL.sh>
```

## 1.4c/ **[CentOS]** Enable EPEL for additional dependencies
```bash
# <snippet-begin 0_CentOS_EPEL.sh>
# We need some packages from the Extra Packages for Enterprise Linux repository
sudo yum install epel-release -y

# Since MISP 2.4 PHP 5.5 is a minimal requirement, so we need a newer version than CentOS base provides
# Software Collections is a way do to this, see https://wiki.centos.org/AdditionalResources/Repositories/SCL
sudo yum install centos-release-scl -y
# <snippet-end 0_CentOS_EPEL.sh>
```

### 1.5a/ Install the deltarpm package to help reduce download size when installing updates (optional)
```bash
sudo yum install deltarpm -y
```

### 1.5.b/ Install vim (optional)
```bash
# Because (neo)vim is just so practical
sudo yum install neovim -y
```

### 1.5.c/ Install ntpdate (optional)
```bash
# In case you time is wrong, this will fix it.
sudo yum install ntpdate -y
sudo ntpdate pool.ntp.org
```

## 1.5/ Update the system and reboot
```bash
# <snippet-begin 0_yum-update.sh>
sudo yum update -y
# <snippet-end 0_yum-update.sh>
```

## 1.6/ **[RHEL]** Install the EPEL repo
```bash
# <snippet-begin 0_RHEL_EPEL.sh>
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm -y
# <snippet-end 0_RHEL_EPEL.sh>
```

# 2/ Dependencies

!!! note
    This guide installs PHP 7.2 from SCL

!!! warning
    [PHP 5.6 and 7.0 aren't supported since December 2018](https://secure.php.net/supported-versions.php). Please update accordingly. In the future only PHP7 will be supported.

## 2.01/ Install some base system dependencies
```bash
# <snippet-begin 0_yumInstallCoreDeps.sh>
yumInstallCoreDeps () {
  # Install the dependencies:
  sudo yum install gcc git zip rh-git218 \
                   httpd24 \
                   mod_ssl \
                   rh-redis32 \
                   rh-mariadb102 \
                   libxslt-devel zlib-devel ssdeep-devel -y

  # Enable and start redis
  sudo systemctl enable --now rh-redis32-redis.service

  # Install PHP 7.2 from SCL, see https://www.softwarecollections.org/en/scls/rhscl/rh-php72/
  sudo yum install rh-php72 rh-php72-php-fpm rh-php72-php-devel \
                   rh-php72-php-mysqlnd \
                   rh-php72-php-mbstring \
                   rh-php72-php-xml \
                   rh-php72-php-bcmath \
                   rh-php72-php-opcache \
                   rh-php72-php-gd -y

  # Install Python 3.6 from SCL, see
  # https://www.softwarecollections.org/en/scls/rhscl/rh-python36/
  sudo yum install rh-python36 -y

  sudo systemctl enable --now rh-php72-php-fpm.service
}
# <snippet-end 0_yumInstallCoreDeps.sh>
```

!!! notice
    $RUN_PHP makes php available for you if using rh-php72. e.g: sudo $RUN_PHP "pear list | grep Crypt_GPG"

```bash
# <snippet-begin 0_yumInstallHaveged.sh>
# GPG needs lots of entropy, haveged provides entropy
# /!\ Only do this if you're not running rngd to provide randomness and your kernel randomness is not sufficient.
sudo yum install haveged -y
sudo systemctl enable --now haveged.service
# <snippet-end 0_yumInstallHaveged.sh>
```

# 3/ MISP code
## 3.01/ Download MISP code using git in /var/www/ directory

```bash
# <snippet-begin 1_mispCoreInstall_RHEL.sh>
# Download MISP using git in the /var/www/ directory.
sudo mkdir $PATH_TO_MISP
sudo chown $WWW_USER:$WWW_USER $PATH_TO_MISP
cd /var/www
$SUDO_WWW git clone https://github.com/MISP/MISP.git
cd $PATH_TO_MISP
##$SUDO_WWW git checkout tags/$(git describe --tags `git rev-list --tags --max-count=1`)
# if the last shortcut doesn't work, specify the latest version manually
# example: git checkout tags/v2.4.XY
# the message regarding a "detached HEAD state" is expected behaviour
# (you only have to create a new branch, if you want to change stuff and do a pull request for example)

# Fetch submodules
$SUDO_WWW git submodule update --init --recursive
# Make git ignore filesystem permission differences for submodules
$SUDO_WWW git submodule foreach --recursive git config core.filemode false
# Make git ignore filesystem permission differences
$SUDO_WWW git config core.filemode false

# Install packaged pears
sudo $RUN_PHP "pear channel-update pear.php.net"
sudo $RUN_PHP "pear install ${PATH_TO_MISP}/INSTALL/dependencies/Console_CommandLine/package.xml"
sudo $RUN_PHP "pear install ${PATH_TO_MISP}/INSTALL/dependencies/Crypt_GPG/package.xml"

# Create a python3 virtualenv
$SUDO_WWW $RUN_PYTHON "virtualenv -p python3 $PATH_TO_MISP/venv"
sudo mkdir /usr/share/httpd/.cache
sudo chown $WWW_USER:$WWW_USER /usr/share/httpd/.cache
$SUDO_WWW $PATH_TO_MISP/venv/bin/pip install -U pip setuptools

cd $PATH_TO_MISP/app/files/scripts
$SUDO_WWW git clone https://github.com/CybOXProject/python-cybox.git
$SUDO_WWW git clone https://github.com/STIXProject/python-stix.git
$SUDO_WWW git clone --branch master --single-branch https://github.com/lief-project/LIEF.git lief
$SUDO_WWW git clone https://github.com/CybOXProject/mixbox.git

cd $PATH_TO_MISP/app/files/scripts/python-cybox
# If you umask is has been changed from the default, it is a good idea to reset it to 0022 before installing python modules
UMASK=$(umask)
umask 0022
cd $PATH_TO_MISP/app/files/scripts/python-stix
$SUDO_WWW $PATH_TO_MISP/venv/bin/pip install .

# install mixbox to accommodate the new STIX dependencies:
cd $PATH_TO_MISP/app/files/scripts/mixbox
$SUDO_WWW $PATH_TO_MISP/venv/bin/pip install .

# install STIX2.0 library to support STIX 2.0 export:
cd $PATH_TO_MISP/cti-python-stix2
$SUDO_WWW $PATH_TO_MISP/venv/bin/pip install .

# install maec
$SUDO_WWW $PATH_TO_MISP/venv/bin/pip install -U maec

# install zmq
$SUDO_WWW $PATH_TO_MISP/venv/bin/pip install -U zmq

# install redis
$SUDO_WWW $PATH_TO_MISP/venv/bin/pip install -U redis

# lief needs manual compilation
sudo yum install devtoolset-7 cmake3 cppcheck -y

# FIXME: This does not work!
cd $PATH_TO_MISP/app/files/scripts/lief
$SUDO_WWW mkdir build
cd build
$SUDO_WWW scl enable devtoolset-7 rh-python36 "bash -c 'cmake3 \
-DLIEF_PYTHON_API=on \
-DPYTHON_VERSION=3.6 \
-DPYTHON_EXECUTABLE=$PATH_TO_MISP/venv/bin/python \
-DLIEF_DOC=off \
-DCMAKE_BUILD_TYPE=Release \
..'"
$SUDO_WWW make -j3 pyLIEF

# The following adds a PYTHONPATH to where the pyLIEF module has been compiled
echo /var/www/MISP/app/files/scripts/lief/build/api/python |$SUDO_WWW tee /var/www/MISP/venv/lib/python3.6/site-packages/lief.pth

# install magic, pydeep
$SUDO_WWW $PATH_TO_MISP/venv/bin/pip install -U python-magic git+https://github.com/kbandla/pydeep.git plyara

# install PyMISP
cd $PATH_TO_MISP/PyMISP
$SUDO_WWW $PATH_TO_MISP/venv/bin/pip install -U .

# Enable python3 for php-fpm
echo 'source scl_source enable rh-python36' | sudo tee -a /etc/opt/rh/rh-php72/sysconfig/php-fpm
sudo sed -i.org -e 's/^;\(clear_env = no\)/\1/' /etc/opt/rh/rh-php72/php-fpm.d/www.conf
sudo systemctl restart rh-php72-php-fpm.service

umask $UMASK

# Enable dependencies detection in the diagnostics page
# This allows MISP to detect GnuPG, the Python modules' versions and to read the PHP settings.
# The LD_LIBRARY_PATH setting is needed for rh-git218 to work, one might think to install httpd24 and not just httpd ...
echo "env[PATH] = /opt/rh/rh-git218/root/usr/bin:/opt/rh/rh-redis32/root/usr/bin:/opt/rh/rh-python36/root/usr/bin:/opt/rh/rh-php72/root/usr/bin:/usr/local/bin:/usr/bin:/bin" |sudo tee -a /etc/opt/rh/rh-php72/php-fpm.d/www.conf
echo "env[LD_LIBRARY_PATH] = /opt/rh/httpd24/root/usr/lib64/" |sudo tee -a /etc/opt/rh/rh-php72/php-fpm.d/www.conf
sudo systemctl restart rh-php72-php-fpm.service
# <snippet-end 1_mispCoreInstall_RHEL.sh>
```

# 4/ CakePHP
## 4.01/ Install CakeResque along with its dependencies if you intend to use the built in background jobs

!!! notice
    CakePHP is now included as a submodule of MISP and has been fetch by a previous step.

```bash
# <snippet-begin 1_installCake_RHEL.sh>
installCake_RHEL ()
{
  sudo chown -R $WWW_USER:$WWW_USER $PATH_TO_MISP
  sudo mkdir /usr/share/httpd/.composer
  sudo chown $WWW_USER:$WWW_USER /usr/share/httpd/.composer
  cd $PATH_TO_MISP/app
  # Update composer.phar (optional)
  #$SUDO_WWW $RUN_PHP -- php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
  #$SUDO_WWW $RUN_PHP -- php -r "if (hash_file('SHA384', 'composer-setup.php') === '48e3236262b34d30969dca3c37281b3b4bbe3221bda826ac6a9a62d6444cdb0dcd0615698a5cbe587c3f0fe57a54d8f5') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
  #$SUDO_WWW $RUN_PHP "php composer-setup.php"
  #$SUDO_WWW $RUN_PHP -- php -r "unlink('composer-setup.php');"
  $SUDO_WWW $RUN_PHP "php composer.phar require kamisama/cake-resque:4.1.2"
  $SUDO_WWW $RUN_PHP "php composer.phar config vendor-dir Vendor"
  $SUDO_WWW $RUN_PHP "php composer.phar install"

  ## sudo yum install php-redis -y
  sudo scl enable rh-php72 'pecl channel-update pecl.php.net'
  sudo scl enable rh-php72 'yes no|pecl install redis'
  echo "extension=redis.so" |sudo tee /etc/opt/rh/rh-php72/php-fpm.d/redis.ini
  sudo ln -s /etc/opt/rh/rh-php72/php-fpm.d/redis.ini /etc/opt/rh/rh-php72/php.d/99-redis.ini
  sudo systemctl restart rh-php72-php-fpm.service

  # If you have not yet set a timezone in php.ini
  echo 'date.timezone = "Asia/Tokyo"' |sudo tee /etc/opt/rh/rh-php72/php-fpm.d/timezone.ini
  sudo ln -s ../php-fpm.d/timezone.ini /etc/opt/rh/rh-php72/php.d/99-timezone.ini

  # Recommended: Change some PHP settings in /etc/opt/rh/rh-php72/php.ini
  # max_execution_time = 300
  # memory_limit = 512M
  # upload_max_filesize = 50M
  # post_max_size = 50M
  for key in upload_max_filesize post_max_size max_execution_time max_input_time memory_limit
  do
      sudo sed -i "s/^\($key\).*/\1 = $(eval echo \${$key})/" $PHP_INI
  done
  sudo systemctl restart rh-php72-php-fpm.service

  # To use the scheduler worker for scheduled tasks, do the following:
  sudo cp -fa $PATH_TO_MISP/INSTALL/setup/config.php $PATH_TO_MISP/app/Plugin/CakeResque/Config/config.php
}
# <snippet-begin 1_installCake_RHEL.sh>
```

# 5/ Set file permissions
```bash
# <snippet-begin 2_permissions_RHEL.sh>
# Main function to fix permissions to something sane
permissions_RHEL () {
  sudo chown -R $WWW_USER:$WWW_USER $PATH_TO_MISP
  ## ? chown -R root:apache /var/www/MISP
  sudo find $PATH_TO_MISP -type d -exec chmod g=rx {} \;
  sudo chmod -R g+r,o= $PATH_TO_MISP
  ## **Note :** For updates through the web interface to work, apache must own the /var/www/MISP folder and its subfolders as shown above, which can lead to security issues. If you do not require updates through the web interface to work, you can use the following more restrictive permissions :
  sudo chmod -R 750 $PATH_TO_MISP
  sudo chmod -R g+xws $PATH_TO_MISP/app/tmp
  sudo chmod -R g+ws $PATH_TO_MISP/app/files
  sudo chmod -R g+ws $PATH_TO_MISP/app/files/scripts/tmp
  sudo chmod -R g+rw $PATH_TO_MISP/venv
  sudo chmod -R g+rw $PATH_TO_MISP/.git
  sudo chown $WWW_USER:$WWW_USER $PATH_TO_MISP/app/files
  sudo chown $WWW_USER:$WWW_USER $PATH_TO_MISP/app/files/terms
  sudo chown $WWW_USER:$WWW_USER $PATH_TO_MISP/app/files/scripts/tmp
  sudo chown $WWW_USER:$WWW_USER $PATH_TO_MISP/app/Plugin/CakeResque/tmp
  sudo chown -R $WWW_USER:$WWW_USER $PATH_TO_MISP/app/Config
  sudo chown -R $WWW_USER:$WWW_USER $PATH_TO_MISP/app/tmp
  sudo chown -R $WWW_USER:$WWW_USER $PATH_TO_MISP/app/webroot/img/orgs
  sudo chown -R $WWW_USER:$WWW_USER $PATH_TO_MISP/app/webroot/img/custom
}
# <snippet-end 2_permissions_RHEL.sh>
```

# 6/ Create database and user

## 6.01/ Set database to listen on localhost only
```bash
# <snippet-begin 1_prepareDB_RHEL.sh>
prepareDB_RHEL () {
  # Enable, start and secure your mysql database server
  sudo systemctl enable --now rh-mariadb102-mariadb.service
  echo [mysqld] |sudo tee /etc/opt/rh/rh-mariadb102/my.cnf.d/bind-address.cnf
  echo bind-address=127.0.0.1 |sudo tee -a /etc/opt/rh/rh-mariadb102/my.cnf.d/bind-address.cnf
  sudo systemctl restart rh-mariadb102-mariadb

  sudo yum install expect -y

  ## The following needs some thoughts about scl enable foo
  #if [[ ! -e /var/opt/rh/rh-mariadb102/lib/mysql/misp/users.ibd ]]; then

  # Add your credentials if needed, if sudo has NOPASS, comment out the relevant lines
  pw="Password1234"

  expect -f - <<-EOF
    set timeout 10

    spawn sudo scl enable rh-mariadb102 mysql_secure_installation
    expect "*?assword*"
    send -- "$pw\r"
    expect "Enter current password for root (enter for none):"
    send -- "\r"
    expect "Set root password?"
    send -- "y\r"
    expect "New password:"
    send -- "${DBPASSWORD_ADMIN}\r"
    expect "Re-enter new password:"
    send -- "${DBPASSWORD_ADMIN}\r"
    expect "Remove anonymous users?"
    send -- "y\r"
    expect "Disallow root login remotely?"
    send -- "y\r"
    expect "Remove test database and access to it?"
    send -- "y\r"
    expect "Reload privilege tables now?"
    send -- "y\r"
    expect eof
EOF

  sudo yum remove tcl expect -y

  sudo systemctl restart rh-mariadb102-mariadb

  scl enable rh-mariadb102 "mysql -u $DBUSER_ADMIN -p$DBPASSWORD_ADMIN -e 'CREATE DATABASE $DBNAME;'"
  scl enable rh-mariadb102 "mysql -u $DBUSER_ADMIN -p$DBPASSWORD_ADMIN -e \"GRANT USAGE on *.* to $DBNAME@localhost IDENTIFIED by '$DBPASSWORD_MISP';\""
  scl enable rh-mariadb102 "mysql -u $DBUSER_ADMIN -p$DBPASSWORD_ADMIN -e \"GRANT ALL PRIVILEGES on $DBNAME.* to '$DBUSER_MISP'@'localhost';\""
  scl enable rh-mariadb102 "mysql -u $DBUSER_ADMIN -p$DBPASSWORD_ADMIN -e 'FLUSH PRIVILEGES;'"

  $SUDO_WWW cat $PATH_TO_MISP/INSTALL/MYSQL.sql | sudo scl enable rh-mariadb102 "mysql -u $DBUSER_MISP -p$DBPASSWORD_MISP $DBNAME"
}
# <snippet-end 1_prepareDB_RHEL.sh>
```

# 7/ Apache Configuration

!!! notice
    SELinux note, to check if it is running:
    ```bash
    $ sestatus
    SELinux status:                 disabled
    ```
    If it is disabled, you can ignore the **chcon/setsebool/semanage/checkmodule/semodule*** commands.

```bash
# <snippet-begin 1_apacheConfig_RHEL.sh>
apacheConfig_RHEL () {
  # Now configure your apache server with the DocumentRoot $PATH_TO_MISP/app/webroot/
  # A sample vhost can be found in $PATH_TO_MISP/INSTALL/apache.misp.centos7

  sudo cp $PATH_TO_MISP/INSTALL/apache.misp.centos7.ssl /etc/httpd/conf.d/misp.ssl.conf
  sudo rm /etc/httpd/conf.d/ssl.conf
  sudo chmod 644 /etc/httpd/conf.d/misp.ssl.conf
  sudo sed -i '/Listen 80/a Listen 443' /etc/httpd/conf/httpd.conf

  # If a valid SSL certificate is not already created for the server, create a self-signed certificate:
  echo "The Common Name used below will be: ${OPENSSL_CN}"
  # This will take a rather long time, be ready. (13min on a VM, 8GB Ram, 1 core)
  if [[ ! -e "/etc/pki/tls/certs/dhparam.pem" ]]; then
    sudo openssl dhparam -out /etc/pki/tls/certs/dhparam.pem 4096
  fi
  sudo openssl genrsa -des3 -passout pass:xxxx -out /tmp/misp.local.key 4096
  sudo openssl rsa -passin pass:xxxx -in /tmp/misp.local.key -out /etc/pki/tls/private/misp.local.key
  sudo rm /tmp/misp.local.key
  sudo openssl req -new -subj "/C=${OPENSSL_C}/ST=${OPENSSL_ST}/L=${OPENSSL_L}/O=${OPENSSL_O}/OU=${OPENSSL_OU}/CN=${OPENSSL_CN}/emailAddress=${OPENSSL_EMAILADDRESS}" -key /etc/pki/tls/private/misp.local.key -out /etc/pki/tls/certs/misp.local.csr
  sudo openssl x509 -req -days 365 -in /etc/pki/tls/certs/misp.local.csr -signkey /etc/pki/tls/private/misp.local.key -out /etc/pki/tls/certs/misp.local.crt
  sudo ln -s /etc/pki/tls/certs/misp.local.csr /etc/pki/tls/certs/misp-chain.crt
  cat /etc/pki/tls/certs/dhparam.pem |sudo tee -a /etc/pki/tls/certs/misp.local.crt 

  sudo systemctl restart httpd.service

  # Since SELinux is enabled, we need to allow httpd to write to certain directories
  sudo chcon -t usr_t $PATH_TO_MISP/venv
  sudo chcon -t httpd_sys_rw_content_t $PATH_TO_MISP/app/files
  sudo chcon -t httpd_sys_rw_content_t $PATH_TO_MISP/app/files/terms
  sudo chcon -t httpd_sys_rw_content_t $PATH_TO_MISP/app/files/scripts/tmp
  sudo chcon -t httpd_sys_rw_content_t $PATH_TO_MISP/app/Plugin/CakeResque/tmp
  sudo chcon -t httpd_sys_script_exec_t $PATH_TO_MISP/app/Console/cake
  sudo chcon -t httpd_sys_script_exec_t $PATH_TO_MISP/app/Console/worker/start.sh
  sudo chcon -t httpd_sys_script_exec_t $PATH_TO_MISP/app/files/scripts/mispzmq/mispzmq.py
  sudo chcon -t httpd_sys_script_exec_t $PATH_TO_MISP/app/files/scripts/mispzmq/mispzmqtest.py
  sudo chcon -t httpd_sys_script_exec_t /usr/bin/ps
  sudo chcon -t httpd_sys_script_exec_t /usr/bin/grep
  sudo chcon -t httpd_sys_script_exec_t /usr/bin/awk
  sudo chcon -t httpd_sys_script_exec_t /usr/bin/gpg
  sudo chcon -t httpd_sys_script_exec_t /usr/bin/gpg-agent
  sudo chcon -t httpd_sys_script_exec_t /usr/bin/whoami
  sudo chcon -t httpd_sys_rw_content_t /tmp
  sudo chcon -R -t usr_t $PATH_TO_MISP/venv
  sudo chcon -R -t httpd_sys_rw_content_t $PATH_TO_MISP/.git
  sudo chcon -R -t httpd_sys_rw_content_t $PATH_TO_MISP/.gnupg
  sudo chcon -R -t httpd_sys_rw_content_t $PATH_TO_MISP/app/tmp
  sudo chcon -R -t httpd_sys_rw_content_t $PATH_TO_MISP/app/Lib
  sudo chcon -R -t httpd_sys_rw_content_t $PATH_TO_MISP/app/Config
  sudo chcon -R -t httpd_sys_rw_content_t $PATH_TO_MISP/app/tmp
  sudo chcon -R -t httpd_sys_rw_content_t $PATH_TO_MISP/app/webroot/img/orgs
  sudo chcon -R -t httpd_sys_rw_content_t $PATH_TO_MISP/app/webroot/img/custom
  sudo chcon -R -t httpd_sys_rw_content_t $PATH_TO_MISP/app/files/scripts/mispzmq
}
# <snippet-end 1_apacheConfig_RHEL.sh>
```

!!! warning
    Todo: Revise all permissions so update in Web UI works.

```bash
# <snippet-begin 1_firewall_RHEL.sh>
firewall_RHEL () {
  # Allow httpd to connect to the redis server and php-fpm over tcp/ip
  sudo setsebool -P httpd_can_network_connect on

  # Allow httpd to send emails from php
  sudo setsebool -P httpd_can_sendmail on

  # Enable and start the httpd service
  sudo systemctl enable --now httpd.service

  # Open a hole in the iptables firewall
  sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
  sudo firewall-cmd --zone=public --add-port=443/tcp --permanent
  sudo firewall-cmd --reload
}
# <snippet-end 1_firewall_RHEL.sh>
```

# 8/ Log Rotation
## 8.01/ Enable log rotation
MISP saves the stdout and stderr of its workers in /var/www/MISP/app/tmp/logs
To rotate these logs install the supplied logrotate script:

```bash
# <snippet-begin 2_logRotation_RHEL.sh>
logRotation_RHEL () {
  # MISP saves the stdout and stderr of its workers in $PATH_TO_MISP/app/tmp/logs
  # To rotate these logs install the supplied logrotate script:

  sudo cp $PATH_TO_MISP/INSTALL/misp.logrotate /etc/logrotate.d/misp
  sudo chmod 0640 /etc/logrotate.d/misp

  # Now make logrotate work under SELinux as well
  # Allow logrotate to modify the log files
  sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/MISP(/.*)?"
  sudo semanage fcontext -a -t httpd_log_t "$PATH_TO_MISP/app/tmp/logs(/.*)?"
  sudo chcon -R -t httpd_log_t $PATH_TO_MISP/app/tmp/logs
  sudo chcon -R -t httpd_sys_rw_content_t $PATH_TO_MISP/app/tmp/logs
  # Impact of the following: ?!?!?!!?111
  ##sudo restorecon -R /var/www/MISP/

  # Allow logrotate to read /var/www
  sudo checkmodule -M -m -o /tmp/misplogrotate.mod $PATH_TO_MISP/INSTALL/misplogrotate.te
  sudo semodule_package -o /tmp/misplogrotate.pp -m /tmp/misplogrotate.mod
  sudo semodule -i /tmp/misplogrotate.pp
}
# <snippet-end 2_logRotation_RHEL.sh>
```

# 9/ MISP Configuration

```bash
# <snippet-begin 2_configMISP_RHEL.sh>
configMISP_RHEL () {
  # There are 4 sample configuration files in $PATH_TO_MISP/app/Config that need to be copied
  $SUDO_WWW cp -a $PATH_TO_MISP/app/Config/bootstrap.default.php $PATH_TO_MISP/app/Config/bootstrap.php
  $SUDO_WWW cp -a $PATH_TO_MISP/app/Config/database.default.php $PATH_TO_MISP/app/Config/database.php
  $SUDO_WWW cp -a $PATH_TO_MISP/app/Config/core.default.php $PATH_TO_MISP/app/Config/core.php
  $SUDO_WWW cp -a $PATH_TO_MISP/app/Config/config.default.php $PATH_TO_MISP/app/Config/config.php

  echo "<?php
  class DATABASE_CONFIG {
          public \$default = array(
                  'datasource' => 'Database/Mysql',
                  //'datasource' => 'Database/Postgres',
                  'persistent' => false,
                  'host' => '$DBHOST',
                  'login' => '$DBUSER_MISP',
                  'port' => 3306, // MySQL & MariaDB
                  //'port' => 5432, // PostgreSQL
                  'password' => '$DBPASSWORD_MISP',
                  'database' => '$DBNAME',
                  'prefix' => '',
                  'encoding' => 'utf8',
          );
  }" | $SUDO_WWW tee $PATH_TO_MISP/app/Config/database.php

  # Configure the fields in the newly created files:
  # config.php   : baseurl (example: 'baseurl' => 'http://misp',) - don't use "localhost" it causes issues when browsing externally
  # core.php   : Uncomment and set the timezone: `// date_default_timezone_set('UTC');`
  # database.php : login, port, password, database
  # DATABASE_CONFIG has to be filled
  # With the default values provided in section 6, this would look like:
  # class DATABASE_CONFIG {
  #   public $default = array(
  #       'datasource' => 'Database/Mysql',
  #       'persistent' => false,
  #       'host' => 'localhost',
  #       'login' => 'misp', // grant usage on *.* to misp@localhost
  #       'port' => 3306,
  #       'password' => 'XXXXdbpasswordhereXXXXX', // identified by 'XXXXdbpasswordhereXXXXX';
  #       'database' => 'misp', // create database misp;
  #       'prefix' => '',
  #       'encoding' => 'utf8',
  #   );
  #}

  # Important! Change the salt key in $PATH_TO_MISP/app/Config/config.php
  # The admin user account will be generated on the first login, make sure that the salt is changed before you create that user
  # If you forget to do this step, and you are still dealing with a fresh installation, just alter the salt,
  # delete the user from mysql and log in again using the default admin credentials (admin@admin.test / admin)

  # If you want to be able to change configuration parameters from the webinterface:
  sudo chown $WWW_USER:$WWW_USER $PATH_TO_MISP/app/Config/config.php
  sudo chcon -t httpd_sys_rw_content_t $PATH_TO_MISP/app/Config/config.php

  # Generate a GPG encryption key.
  cat >/tmp/gen-key-script <<EOF
      %echo Generating a default key
      Key-Type: default
      Key-Length: $GPG_KEY_LENGTH
      Subkey-Type: default
      Name-Real: $GPG_REAL_NAME
      Name-Comment: $GPG_COMMENT
      Name-Email: $GPG_EMAIL_ADDRESS
      Expire-Date: 0
      Passphrase: $GPG_PASSPHRASE
      # Do a commit here, so that we can later print "done"
      %commit
      %echo done
EOF

  sudo gpg --homedir $PATH_TO_MISP/.gnupg --batch --gen-key /tmp/gen-key-script
  sudo rm -f /tmp/gen-key-script
  sudo chown -R $WWW_USER:$WWW_USER $PATH_TO_MISP/.gnupg

  # And export the public key to the webroot
  sudo gpg --homedir $PATH_TO_MISP/.gnupg --export --armor $GPG_EMAIL_ADDRESS |sudo tee $PATH_TO_MISP/app/webroot/gpg.asc
  sudo chown $WWW_USER:$WWW_USER $PATH_TO_MISP/app/webroot/gpg.asc

  echo "Admin (root) DB Password: $DBPASSWORD_ADMIN"
  echo "User  (misp) DB Password: $DBPASSWORD_MISP"
}
# <snippet-end 2_configMISP_RHEL.sh>
```


```bash
# In case you have no /etc/rc.local make a bare-bones one.
if [ ! -e /etc/rc.local ]
then
    echo '#!/bin/sh -e' | sudo tee -a /etc/rc.local
    echo 'exit 0' | sudo tee -a /etc/rc.local
    sudo chmod u+x /etc/rc.local
fi
```

!!! note
    There is a bug that if a passphrase is added MISP will produce an error on the diagnostic page.<br />
    /!\ THIS WANTS TO BE VERIFIED AND LINKED WITH A CORRESPONDING ISSUE.

!!! note
    The email address should match the one set in the config.php configuration file
    Make sure that you use the same settings in the MISP Server Settings tool

## 9.06/ Use MISP's background workers
### 9.06a/ Create a systemd unit for the workers
```bash
echo "[Unit]
Description=MISP background workers
After=rh-mariadb102-mariadb.service rh-redis32-redis.service rh-php72-php-fpm.service

[Service]
Type=forking
User=apache
Group=apache
ExecStart=/usr/bin/scl enable rh-php72 rh-redis32 rh-mariadb102 /var/www/MISP/app/Console/worker/start.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target" |sudo tee /etc/systemd/system/misp-workers.service
```

Make the workers' script executable and reload the systemd units :
```bash
sudo chmod +x /var/www/MISP/app/Console/worker/start.sh
sudo systemctl daemon-reload
```

### 9.06b/ Start the workers and enable them on boot
```bash
sudo systemctl enable --now misp-workers.service
```

### 9.07/ misp-modules (WIP!)
```bash
# some misp-modules dependencies
sudo yum install openjpeg-devel gcc-c++ poppler-cpp-devel -y

sudo chmod 2777 /usr/local/src
sudo chown root:users /usr/local/src
cd /usr/local/src/
$SUDO_WWW git clone https://github.com/MISP/misp-modules.git
cd misp-modules
# pip install
$SUDO_WWW $PATH_TO_MISP/venv/bin/pip install -U -I -r REQUIREMENTS
$SUDO_WWW $PATH_TO_MISP/venv/bin/pip install -U .
sudo yum install rubygem-rouge rubygem-asciidoctor zbar-devel opencv-devel -y

echo "[Unit]
Description=MISP modules
After=misp-workers.service

[Service]
Type=simple
User=apache
Group=apache
ExecStart=\"${PATH_TO_MISP}/venv/bin/misp-modules –l 127.0.0.1 –s\"
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target" |sudo tee /etc/systemd/system/misp-modules.service

sudo systemctl daemon-reload
# Test misp-modules
$SUDO_WWW $PATH_TO_MISP/venv/bin/misp-modules -l 127.0.0.1 -s
sudo systemctl enable --now misp-modules

  # Enable Enrichment, set better timeouts
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Enrichment_services_enable" true
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Enrichment_hover_enable" true
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Enrichment_timeout" 300
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Enrichment_hover_timeout" 150
  # TODO:"Investigate why the next one fails"
  #$SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Enrichment_asn_history_enabled" true
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Enrichment_cve_enabled" true
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Enrichment_dns_enabled" true
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Enrichment_services_url" "http://127.0.0.1"
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Enrichment_services_port" 6666

  # Enable Import modules, set better timeout
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Import_services_enable" true
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Import_services_url" "http://127.0.0.1"
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Import_services_port" 6666
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Import_timeout" 300
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Import_ocr_enabled" true
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Import_csvimport_enabled" true

  # Enable Export modules, set better timeout
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Export_services_enable" true
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Export_services_url" "http://127.0.0.1"
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Export_services_port" 6666
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Export_timeout" 300
  $SUDO_WWW $RUN_PHP -- $CAKE Admin setSetting "Plugin.Export_pdfexport_enabled" true

```

{!generic/misp-dashboard-centos.md!}

{!generic/MISP_CAKE_init_centos.md!}

{!generic/INSTALL.done.md!}

{!generic/recommended.actions.md!}

# 11/ LIEF Installation
*lief* is required for the Advanced Attachment Handler and requires manual compilation

The installation is explained in section **[3.01](https://misp.github.io/MISP/INSTALL.rhel7/#301-download-misp-code-using-git-in-varwww-directory)**

# 12/ Known Issues
## 12.01/ Workers cannot be started or restarted from the web page
Possible also due to package being installed via SCL, attempting to start workers through the web page will result in error. Worker's can be restarted via the CLI using the following command.
```bash
systemctl restart misp-workers.service
```

!!! note 
    No other functions were tested after the conclusion of this install. There may be issue that aren't addressed<br />
    via this guide and will need additional investigation.

{!generic/hardening.md!}
