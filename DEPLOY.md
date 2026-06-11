# Deploying the Lets WiFi portal

## Linux Packages Needed

- [] apache2
- [] php
- [] libapache2-mod-php
- [] make
- [] 7zip
- [] php-xml
- [] sqlite3
- [] php-sqlite3
- [] compose

## Configure Apache

<details><summary>
Example Configuration
</summary>

```apacheconf
<VirtualHost *:443>
    ServerName   dev.net.uiowa.edu
    DocumentRoot /var/www/html/letswifi-portal/htdocs
    Alias        /simplesaml  /var/www/html/letswifi-portal/simplesamlphp/public

    # SetEnv SIMPLESAMLPHP_CONFIG_DIR /var/www/html/letswifi/etc/simplesamlphp
    # SetEnv LETSWIFI_CONFIG_DIR      /var/www/html/letswifi/etc/letswifi

    SSLEngine                on
    SSLCertificateFile      /etc/ssl/certs/uiowa_edu.ca-bundle
    SSLCertificateKeyFile   /etc/ssl/private/dev_net_uiowa_edu.key

    <Directory /var/www/html/letswifi/www>
        Require all granted
    </Directory>
    <Directory /var/www/html/letwifi-portal/simplesamlphp/public>
        Require all granted
    </Directory>
</VirtualHost>
```

</details>

## Install letswifi

```sh
cd /var/www/html
git clone -b beta https://github.com/geteduroam/letswifi-portal
cd letswifi-portal
composer --no-dev --quiet install
```

## Configure

## Install and config simplesaml

Install simplesaml in /var/www/html/letswifi-portal

```sh
export SSPVER=2.5.2
curl -L https://github.com/simplesamlphp/simplesamlphp/releases/download/v$SSPVER/simplesamlphp-$SSPVER-full.tar.gz | tar xzvf -
mv simplesamlphp-$SSPVER simplesamlphp
cp simplesamlphp/config/config.php.dist simplesaml/config/config.php
```

Edit `simplesamlphp/config/config.php`

- [] Remove leading `/` from `cachedir` path
- [] Set `technicalcontact_name` and `technicalcontact_emal`
- [] Set `secretsalt` to large random string Use the following `LC_ALL=C tr -c -d '0123456789abcdefghijklmnopqrstuvwxyz' </dev/urandom | dd bs=32 count=1 2>/dev/null;echo`
- [] Set `auth.adminpassword`
- [] Set `admin.protectmetadata` to `true`

```sh
cp cp simplesamlphp/config/authsources.php.dist simplesamlphp/config/authsources.php
```

Edit `simplesamlphp/config/authsources.php`

Top of file should looklike

```php
    // An authentication source which can authenticate against SAML 2.0 IdPs.
    'default-sp' => [
        'saml:SP',

        // The entity ID of this SP.
        'entityID' => 'https://dev.net.uiowa.edu',
        'privatekey' => 'saml.pem',
        'certificate' => 'saml.crt',
        'sign.authnrequest' => true,

        // The entity ID of the IdP this SP should contact.
        // Can be NULL/unset, in which case the user will be shown a list of available IdPs.
        'idp' => 'iowafed-test:idp:uiowa.edu',
```

Goto [https://server.com/simplesaml/admin](https://server.com/simplesaml/admin)

Login usin password setin simplesamlphp/config/config.php

Goto federation->tools, click on `[XML to SimpleSAMLphp metadata converter`

Paste in XML Metadata from your IAM team, Click `Parse`

Copy PHP code to `simplesamlphp/metadata/saml20-idp-remote.php`

```sh
head -c32 /dev/random | base64 | tr -d = >config/oauthsecret.txt
```

### Setup base config

```sh
cp config-dist/letswifi.conf.php config
```

Edit `config/letswifi.config.php`

Under default provider

- [] Configure `'realm'` array with your realm
- [] Set `'contact'` to your contact
- [] Add your local admins to the `'admin'` array
- [] Set `autoloadInclude` to `simplesamlphp/lib/_autoload.php`
- [] Under `contact` Section
  - [] Set `contact` to same string as in the above contact
  - [] Change `realm` to `uiowa.edu`
  - [] Fill out `mail`, `web`, `phone`
  
### Configure Clients

```sh
cp config-dist/clients-eduroam.conf.php conifg/clients.conf.php
```

### Setup database

```sh
mkdir config
cp config-dist/database.conf.dist-sqlite.php config/database.conf.php
mkdir var
sqlite3 var/letswifi.sqlite < sql/letswifi.sqlite.sql
```

Edit `config/database.conf.php` and remove leading '/' from `/var`

### Setup Realms

```sh
cp -r config-dist/realms config
cp config/realms/example.com.conf.php config/realms/uiowa.edu.conf.php
```

Edit `config/realms/uiowa.edu.conf.php`

- [] Change instances of `example.com` to `realm.edu`
- [] Set `contact` to same as entry in `config/letswifi-conf.php`

Edit `config/letswifi.conf.php

## Load RADIUS Certificate

```sh
mkdir config/certs
chmod 750 config/certs
```

Copy Root and Intermediate Certificates using the Subject NAme as the file name except without spaces between the equal signs in the name, end with `.pem` extension

Create Realm

```sh
bin/letswifi realm uiowa.edu \
--newca 'Example CA' \
--validity 366 \
Services' --server-name 'net-auth-1.its.uiowa.edu'
```
