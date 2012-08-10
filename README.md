# PHP OAuth Authorization Server

This project aims at providing a stand-alone OAuth v2 Authorization Server that
is easy to integrate with your existing REST services, written in any language, 
without requiring extensive changes.

# Features
* PDO (database abstraction layer for various databases) storage backend for
  OAuth tokens
* OAuth v2 (authorization code and implicit grant) support
* SAML authentication support ([simpleSAMLphp](http://www.simplesamlphp.org)) 
* [BrowserID](http://browserid.org) authentication support using 
([php-browserid](https://github.com/fkooman/php-browserid/))

# Requirements
The installation requirements on Fedora/CentOS can be installed like this:

    $ su -c 'yum install git php-pdo php httpd unzip wget'

On Debian/Ubuntu:

    $ sudo apt-get install git sqlite3 php5 php5-sqlite wget unzip

# Installation
*NOTE*: in the `chown` line you need to use your own user account name!

    $ cd /var/www/html
    $ su -c 'mkdir php-oauth'
    $ su -c 'chown fkooman.fkooman php-oauth'
    $ git clone git://github.com/fkooman/php-oauth.git
    $ cd php-oauth

Now you can create the default configuration files, the paths will be 
automatically set, permissions set and a sample Apache configuration file will 
be generated and shown on the screen (see below for more information on
Apache configuration).

    $ docs/configure.sh

Next make sure to configure the database settings in `config/oauth.ini`, and 
possibly other settings. If you want to keep using SQlite you are good to go 
without fiddling with the database settings. Now to initialize the database:

    $ php docs/initOAuthDatabase.php

*NOTE*: On Ubuntu (Debian) you would typically install in `/var/www/php-oauth` and not 
in `/var/www/html/php-oauth` and you use `sudo` instead of `su -c`.

# Management Client
There are two reference management clients available:

* [Manage Applications](https://github.com/fkooman/html-manage-applications/). 
* [Manage Authorizations](https://github.com/fkooman/html-manage-authorizations/). 

These clients are written in HTML, CSS and JavaScript only and can be hosted on 
any (static) web server. See the accompanying READMEs for more information.

# SELinux
The install script already takes care of setting the file permissions of the
`data/` directory to allow Apache to write to the directory. If you want to use
the BrowserID authentication plugin you also need to give Apache permission to 
access the network. These permissions can be given by using `setsebool` as root:

    $ sudo setsebool -P httpd_can_network_connect=on

This is only for Red Hat based Linux distributions like RHEL, CentOS and 
Fedora.

# Apache
There is an example configuration file in `docs/apache.conf`. 

On Red Hat based distributions the file can be placed in 
`/etc/httpd/conf.d/php-oauth.conf`. On Debian based distributions the file can
be placed in `/etc/apache2/conf.d/php-oauth`. Be sure to modify it to suit your 
environment and do not forget to restart Apache. 

The install script from the previous section outputs a config for your system
which replaces the `/PATH/TO/APP` with the actual directory.

# Authentication
There are thee plugins provided to authenticate users:

* DummyResourceOwner - Static account configured in `config/oauth.ini`
* SspResourceOwner - simpleSAMLphp plugin for SAML authentication
* BrowserIDResourceOwner - BrowserID / Mozilla Persona plugin

You can configure which plugin to use by modifying the `authenticationMechanism`
setting.

## Entitlements
A more complex part of the authentication and authorization is the use of 
entitlements. This is a bit similar to scope in OAuth, only entitlements are 
for resource owner, while scope is only for OAuth clients.

The entitlements are used in the `php-oauth` API. It is for example possible to 
write a client application that uses the `php-oauth` API to manage OAuth client 
registrations. The problem now is how to decide who is allowed to manage 
OAuth client registrations. Clearly not all users who can successfully 
authenticate, but only a subset. The way now to determine who gets to do what
is accomplished through entitlements. 

In the `[Api]` section the following is configured:

    ; manage OAuth clients (global)
    requireEntitlement["applications"] = TRUE
    ; manage OAuth consent/authorization/approval (per resource owner)
    requireEntitlement["authorizations"] = FALSE
    ; retrieve resource owner information (per resource owner)
    requireEntitlement["resource_owner"] = FALSE

This tells the `php-oauth` API that for management applications an entitlement
is required in this case the `applications` entitlement. In the authentication
module it can then be specified what particular resource owner will get this
entitlement. For instance in the `DummyResourceOwner` section:

    ; Dummy Configuration
    [DummyResourceOwner]
    resourceOwnerId = "fkooman"

    resourceOwnerEntitlement["applications"] = "fkooman"
    ;resourceOwnerEntitlement["authorizations"] = "fkooman"

Here you can see that the resource owner `fkooman` will be granted the 
`applications` entitlement. As there is only one account in the 
`DummyResourceOwner` configuration it is quite boring.

Now, for the `SspResourceOwner` configuration it is a little bit more complex, 
all non relevant configuration was stripped:

    [SspResourceOwner]
    ; by default we recommend to use an entitlement SAML attribute to indicate
    ; who is allowed to access certain restricted API calls
    entitlementAttributeName = "urn:mace:dir:attribute-def:eduPersonEntitlement"
    entitlementFromAttributeValue["applications"] = "urn:vnd:oauth2:applications"

This means that the entitlement is determined by looking at the SAML attribute 
`urn:mace:dir:attribute-def:eduPersonEntitlement` provided as part
of the SAML assertion. If (one of the values) of this attribute contains
`urn:vnd:oauth2:applications` that particular user will be granted the
`applications` entitlement. The SAML IdP will have to set this entitlement for
users that are allowed to perform OAuth client registrations. This is 
convenient as you no longer need to modify the configuration of `php-oauth` to
add a new "administrator", but can just add the entitlement to the user in the
IdP.

## simpleSAMLphp
In the configuration file `config/oauth.ini` various aspects can be configured. 
To configure the SAML integration, make sure the following settings are correct:

    authenticationMechanism = "SspResourceOwner"

    ; simpleSAMLphp configuration
    [SspResourceOwner]
    sspPath = "/var/simplesamlphp"
    authSource = "default-sp"

    ; by default we use the (persistent) NameID value received from the SAML 
    ; assertion as the user identifier (RECOMMENDED)
    useNameID = TRUE

    ; you can also use an attribute to determine the UID, but this is not 
    ; recommended and should only be used for testing purposes!
    ;resourceOwnerIdAttributeName = "uid"
    ;resourceOwnerIdAttributeName = "urn:mace:dir:attribute-def:uid"
