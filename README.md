# docker-mediawiki
Mediawiki + Remote Auth Modules for x64

WARNING: The default `/etc/apache2/site-config/ldap.conf` in this image showcases the configuration for the LDAP-Module by using the public demo of Freeipa (ip1.demo.freeipa.org). Thus allowing any user in the demo LDAP access to the location `/ldap-status`, but not the wiki itself.

## What is in here?
The main goal is to use Mediawiki with remote authentication. This image thus comes with the following installed:
- mod_mellon: SAML-based authentication
- mod_authnz_ldap: LDAP-based authentication
- ssmtp: To send e-mails from Mediawiki
- Hardened Apache Config (ServerTokens, ServerSignature, Indexes disabled)

## How to configure
The relevant configuration files have to be mounted to `/etc/apache2/site-config` inside the container.
### You want to use basic auth
Provide `mellon.conf` with the follwing content:

    ServerName YOUR_URL_GOES_HERE
    <Location "/">
        AuthType Basic
        AuthName "Authentication Required"
        AuthUserFile "/etc/apache2/site-config/.htpasswd"
        Require valid-user
    </Location>

Of course, you need to provide a pregenerated `.htpasswd` as well.

### You have a reverse proxy handling authentication
Provide `mellon.conf` in a way that adds a Header to the Request. For example, if you are using traefik:

    ServerName YOUR_CONTAINER_IP
    <Location "/">
        SetEnvIf X-Webauth-User ^(.*)$ LOGINNAME=$1
        RequestHeader add PHP_AUTH_USER %{LOGINNAME}e
    </Location>

### You want to use a SAML IdP
You don't have to provide a config file, but keys must be generated first (or provided).

The keys go to: `/etc/apache2/saml/` inside the container. You have to provide `/etc/apache2/saml/mellon_metadata.xml`, `/etc/apache2/saml/mellon.key`, `/etc/apache2/saml/mellon.crt` and `/etc/apache2/saml/idp_metadata.xml`.

You can use pregenerated keys or use the script inside the container with:

    docker run -it paulritter/tinytinyrss /opt/firstrun.sh FQDN_OF_YOUR_INSTALLATION

It outputs the generated files for you and stores them in a volume.

You can then generate your `idp_metadata.xml`. For example, if you are using Keycloak, you can use this Guide https://www.keycloak.org/docs/latest/securing_apps/index.html#_mod_auth_mellon
### You want to use LDAP
Provide a file called `ldap.conf` in `/etc/apache2/site-config` with the relevant settings. This example uses the public demo of FreeIPA

    ServerName YOUR_URL_GOES_HERE
    <Location "/ldap-status">
        SetHandler ldap-status

        AuthType Basic
        AuthName "LDAP Protected"
        AuthBasicProvider ldap
        AuthLDAPInitialBindAsUser on
        AuthLDAPInitialBindPattern (.+) uid=$1,cn=users,cn=accounts,dc=demo1,dc=freeipa,dc=org
        
        AuthLDAPUrl "ldap://ipa.demo1.freeipa.org/cn=users,cn=accounts,dc=demo1,dc=freeipa,dc=org?uid,displayName,mail??(memberOf=cn=employees,cn=groups,cn=accounts,dc=demo1,dc=freeipa,dc=org)"

        Require valid-user
    </Location>
    <Location "/">
        AuthType Basic
        AuthName "LDAP Protected"
        AuthBasicProvider ldap
        AuthLDAPInitialBindAsUser on
        AuthLDAPInitialBindPattern (.+) uid=$1,cn=users,cn=accounts,dc=demo1,dc=freeipa,dc=org
        
        AuthLDAPUrl "ldap://ipa.demo1.freeipa.org/cn=users,cn=accounts,dc=demo1,dc=freeipa,dc=org?uid,displayName,mail??(memberOf=cn=employees,cn=groups,cn=accounts,dc=demo1,dc=freeipa,dc=org)"

        Require valid-user
    </Location>

Provide `mellon.conf` as follows:

    ServerName 127.0.0.1
    <Location "/">
       RequestHeader add PHP_AUTH_USER YOUR_DESIRED_USERNAME
    </Location>

### Configure SSMTP
The ssmtp configuration goes in `/etc/ssmtp` inside the container.

## Installation
Start your container with any of the configuration options above. For example:

    docker run -it -p 80:80 -v /etc/timezone:/etc/timezone:ro \
        -v /etc/localtime:/etc/localtime:ro \
        -v /srv/ssmtp:/etc/ssmtp \
        -v /srv/mediawiki:/etc/apache2/site-config \
        paulritter/mediawiki:latest

Navigate to `http://YOUR_URL/` and configure your instance.

Save the configuration file, terminate the container and provide the file to the container on next startup. Make sure to include adjustments for the Auth_Remoteuser plugin of Mediawiki.

    docker run -d -p 80:80 -v /etc/timezone:/etc/timezone:ro \
        -v /etc/localtime:/etc/localtime:ro \
        -v /srv/ssmtp:/etc/ssmtp \
        -v /srv/mediawiki/LocalSettings.php:/var/www/html/LocalSettings.php:z \
        -v /srv/mediawiki/images:/var/www/html/images:z \
        -v /srv/mediawiki/conf:/etc/apache2/site-config:z \
        paulritter/tinytinyrss:latest
       
