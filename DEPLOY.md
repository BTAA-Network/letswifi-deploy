# Deploying the Lets WiFi portal (Single directory)

## Overview

## Table of Contents

- [Deploying the Lets WiFi portal (Single directory)](#deploying-the-lets-wifi-portal-single-directory)
  - [Overview](#overview)
  - [Table of Contents](#table-of-contents)
  - [Assumptions](#assumptions)
  - [Requirements](#requirements)
    - [Linux Packages Needed](#linux-packages-needed)
  - [Setup Apache 2](#setup-apache-2)
    - [Obtain SSL certs for web server](#obtain-ssl-certs-for-web-server)
    - [Configure Server](#configure-server)
  - [Install letswifi Source](#install-letswifi-source)
  - [Install and config simplesamlphp](#install-and-config-simplesamlphp)
    - [Install simplesaml in /var/www/html/letswifi-portal](#install-simplesaml-in-varwwwhtmlletswifi-portal)
    - [Edit `simplesamlphp/config/config.php`](#edit-simplesamlphpconfigconfigphp)
    - [Generate self signed certificate for simplesamlphp](#generate-self-signed-certificate-for-simplesamlphp)
    - [Edit `simplesamlphp/config/authsources.php`](#edit-simplesamlphpconfigauthsourcesphp)
    - [Geneate Configuration for IAM Metadate file](#geneate-configuration-for-iam-metadate-file)
  - [Configure letswifi portal](#configure-letswifi-portal)
    - [Set oauth Secret](#set-oauth-secret)
    - [Edit letswifi configuration files](#edit-letswifi-configuration-files)
    - [Configure Clients](#configure-clients)
    - [Setup database](#setup-database)
    - [Setup Realms](#setup-realms)
  - [Load RADIUS Certificate](#load-radius-certificate)
  - [Create Realm](#create-realm)
  - [Set file ownership to web server user](#set-file-ownership-to-web-server-user)
  - [Setting contact information Logo's etc](#setting-contact-information-logos-etc)

## Assumptions

- Using Ubuntu
- Using Apache2 for the webserver
- All files will be kept in a single directory hierarchy (/var/www/html/letswifi-portal)

## Requirements

### Linux Packages Needed

- apache2
- php
- libapache2-mod-php
- make
- 7zip
- php-xml
- sqlite3
- php-sqlite3
- compose
- MySQL or MariaDB

## Setup Apache 2

### Obtain SSL certs for web server

(**TODO: Point repo for gennerating certificates**)

### Configure Server

Example `/etc/apache2/sites-available/000-default.conf` (redirects http: to https:)

```apacheconf
<VirtualHost *:80>
        ServerName www.example.edu

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html/letswifi-portal/www

        Redirect permanent / https://example.edu/

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

Example `/etc/apache2/sites-enabled/default-ssl.conf` Configuration

```apacheconf
<VirtualHost *:443>
    ServerName  www.example.edu
    DocumentRoot /var/www/html/letswifi-portal/htdocs
    Alias        /simplesaml  /var/www/html/letswifi-portal/simplesamlphp/public

    SSLEngine                on
    SSLCertificateFile      /etc/ssl/certs/ca-bundle
    SSLCertificateKeyFile   /etc/ssl/private/www.example.edu.key

    <Directory /var/www/html/letswifi/www>
        Require all granted
    </Directory>
    <Directory /var/www/html/letwifi-portal/simplesamlphp/public>
        Require all granted
    </Directory>
</VirtualHost>
```

You will then want create symbolic links in  `/etc/apache2/sites-enabled` from the files in  `/etc/apache2/sites-available`

## Install letswifi Source

```bash
cd /var/www/html
git clone -b beta https://github.com/geteduroam/letswifi-portal
cd letswifi-portal
composer --no-dev --quiet install
```

**NOTE**: From here on out all commands are relaitive to the `/var/www/html/letswifi-portal` directory

## Install and config simplesamlphp

### Install simplesaml in /var/www/html/letswifi-portal

Find the latest release at `https://github.com/simplesamlphp/simplesamlphp/releases`

Set `SSPVER` environmental variable to the version of simplesamlphp you want to use

```bash
export SSPVER=2.5.2
curl -L https://github.com/simplesamlphp/simplesamlphp/releases/download/v$SSPVER/simplesamlphp-$SSPVER-full.tar.gz | tar xzvf -
mv simplesamlphp-$SSPVER simplesamlphp
cp simplesamlphp/config/config.php.dist simplesaml/config/config.php
```

### Edit `simplesamlphp/config/config.php`

- Remove leading `/` from `cachedir` path
- Set `technicalcontact_name` and `technicalcontact_email`
- Set `secretsalt` to large random string Use the following command `LC_ALL=C tr -c -d '0123456789abcdefghijklmnopqrstuvwxyz' </dev/urandom | dd bs=32 count=1 2>/dev/null;echo`
- Set `auth.adminpassword` (You will need this later to parse the SAML meta data file)
- Set `admin.protectmetadata` to `true`

```bash
cp cp simplesamlphp/config/authsources.php.dist simplesamlphp/config/authsources.php
```

### Generate self signed certificate for simplesamlphp

```bash
cd /var/www/html/letswifi-portal/simplesamlphp
openssl req -newkey rsa:3072 -new -x509 -days 3652 -nodes \
    -out cert/saml.crt \
    -keyout cert/saml.pem \
    -subj "/C=US/ST=Iowa/O=Example EDU/CN=letswifi.its.example.edu"
```

Provide the certificate (`.crt`) file to your Identity and Access Management Team

### Edit `simplesamlphp/config/authsources.php`

Top of the file should looklike:

```php
    // An authentication source which can authenticate against SAML 2.0 IdPs.
    'default-sp' => [
        'saml:SP',

        // The entity ID of this SP.
        'entityID' => 'https://www.example.edu',
        'privatekey' => 'saml.pem',
        'certificate' => 'saml.crt',
        'sign.authnrequest' => true,

        // The entity ID of the IdP this SP should contact.
        // Can be NULL/unset, in which case the user will be shown a list of available IdPs.
        'idp' => '<<ENTITY ID FROM IAM TEAM>>',
...
```

### Geneate Configuration for IAM Metadate file

Goto [https://www.example.edu/simplesaml/admin](https://www.example.edu/simplesaml/admin)

Login using admin password set in `simplesamlphp/config/config.php`

Goto federation->tools, click on `XML to SimpleSAMLphp metadata converter`

Load in the XML Metadata file from your IAM team, Click `Parse`

Copy generated PHP code to the file `simplesamlphp/metadata/saml20-idp-remote.php`

## Configure letswifi portal

### Set oauth Secret

```bash
head -c32 /dev/random | base64 | tr -d = >config/oauthsecret.txt
```

### Edit letswifi configuration files

```bash
cp config-dist/letswifi.conf.php config
```

Edit `config/letswifi.config.php`

Under default provider

- Configure `'realm'` array with your realm
- Set `'contact'` to your contact
- Add your local admins to the `'admin'` array
- Set `autoloadInclude` to `simplesamlphp/lib/_autoload.php`
- Under `contact` Section
  - Set `contact` to same string as in the above contact
  - Change `realm` to `example.edu`
  - Fill out `mail`, `web`, `phone`
  
### Configure Clients

This will create points to the various windows, iOS, Android, and Linux configuration apps.

```bash
cp config-dist/clients-eduroam.conf.php conifg/clients.conf.php
```

### Setup database

For testing we are using a flat sqlite3 database, In production you will probably want to use MySQL or MariaDB.

```bash
mkdir config
cp config-dist/database.conf.dist-sqlite.php config/database.conf.php
mkdir var
sqlite3 var/letswifi.sqlite < sql/letswifi.sqlite.sql
```

Edit `config/database.conf.php` and remove leading '/' from `/var`

### Setup Realms

```bash
cp -r config-dist/realms config
cp config/realms/example.com.conf.php config/realms/example.edu.conf.php
```

Edit `config/realms/example.edu.conf.php`

- Change instances of `example.com` to `example.edu`
- Set `contact` entry to match contact entry in `config/letswifi-conf.php`

## Load RADIUS Certificate

```bash
mkdir config/certs
chmod 750 config/certs

```

Copy Certificate Authority Root and Intermediate Certificates using the Subject Name as the file name except **without spaces between the equal signs in the name**, and ending with a `.pem` extension.

**NOTE**: The file names must match the certificates DN exactly or letswifi won't be able to locate them!

For example (using Incommon certificates)

```text
'C=GB, ST=Greater Manchester, L=Salford, O=Comodo CA Limited, CN=AAA Certificate Services.pem'

'C=US, O=Internet2, CN=InCommon RSA Server CA 2.pem'

'C=US, ST=New Jersey, L=Jersey City, O=The USERTRUST Network, CN=USERTrust RSA Certification Authority.pem'
```

## Create Realm

```bash
bin/letswifi realm example.edu \
--newca 'Example CA' \
--validity 366 \
Services' --server-name 'radius-server.example.edu'
```

This will geneate the key and certificate used to sign the EAP-TLS client certificates. The Certificate will be placed in `config/certs`

## Set file ownership to web server user 

(www-data for ubuntu)

```bash
chown -R www-data:www-data /var/www/html/letswifi-portal
```

## Setting contact information Logo's etc

See the file `config/realms/example.com.conf.php` for examples how to enter information displayed in the apps,
