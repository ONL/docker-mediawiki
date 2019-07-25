# docker-mediawiki
Mediawiki + Remote Auth Modules for ARM and x64

WARNING: The default `/etc/apache2/site-config/ldap.conf` in this image showcases the configuration for the LDAP-Module by using the public demo of Freeipa (ip1.demo.freeipa.org). Thus allowing any user in the demo LDAP access to the location `/ldap-status`, but not the wiki itself.
