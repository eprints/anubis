# Anubis Configuration Support

**This is still experimental, use at your own risk**

Anubis is an open source project which attempts to block web scrapers. It behaves as a reverse proxy, deciding whether to forward on requests to the service being protected or block them. EPrints is susceptible to being taken offline by swarms of AI bots which are indistinguishable from a DDOS attack. The CPU heavy search and statistics pages can very easily be overwhelmed and take an entire repository offline.

The default configuration aims to reach a balance between being as open as possible - we should like to allow well behaved scrapers to index EPrints - but blocking the worst bots to prioritise staying online.

## How to configure

1. Ensure EPrints is currently configured to use SSL in the standard way [as per the wiki](https://wiki.eprints.org/w/How_to_use_EPrints_with_HTTPS) with the SSL configuration in archive/*/ssl/securevhost.conf
2. Install Anubis. [Anubis official install guide](https://anubis.techaro.lol/docs/admin/native-install/). Official deb and RPM files are available.
3. Install this ingredient: 
   1. `cd /opt/eprints3/ingredients`
   2. `git clone https://github.com/eprints/anubis.git`
   3. `git checkout v0.2` (or whichever release your desire)
   4. `echo "ingredients/anubis" >> /opt/eprints3/flavours/pub_lib/inc`
4. Copy `/opt/eprints3/ingredients/anubis/anubis_config/eprints.botPolicies.yaml` to `/opt/eprints3/archives/[YOUR ARCHIVE ID]/anubis/eprints.botPolicies.yaml`
5. Create `/etc/anubis/eprints.env` with: 
```BIND=:8923
DIFFICULTY=4
METRICS_BIND=[::1]:9090
METRICS_BIND_NETWORK=tcp
SERVE_ROBOTS_TXT=0
TARGET=http://localhost:3000
POLICY_FNAME=/opt/eprints3/archives/[YOUR ARCHIVE ID]/anubis/eprints.botPolicies.yaml
```
6. Run `/opt/eprints3/ingredients/anubis/bin/generate_apacheconf_for_anubis --replace --system` to update EPrints apache config files to set up the Anubis proxy
7. Edit `/opt/eprints3/archives/[YOUR ARCHIVE ID]/ssl/securevhost.conf` to:
   1.  include anubis.conf (`Include /opt/eprints3/cfg/apache_ssl/anubis.conf`) 
   2.  remove the include for eprints_ssl.conf
   3.  Remove `PerlTransHandler +EPrints::Apache::Rewrite`
   4.  An example SSL config file using Lets Encrypt is provided in `/opt/eprints3/ingredients/anubis/ssl/securevhost.conf.example` for reference
8. Enable and start systemd module for anubis for EPrints: `sudo systemctl enable --now anubis@eprints.service`
9. Restart apache: `sudo systemctl restart httpd` 

## How to confirm this is working

Open a new browser, or an incogneto window in a browser and navigate to your repository's search page. When first loading the search page you should briefly see the anubis logo pop up.

Log in as an administrator and navigate to the Admin page. Under "System Tools" there should be a new button "Anubis Status". This page will report Anubis' metrics. Note that the metrics are the current cumulative total. Future work could including some way of logging and graphing these metrics to keep an eye on Anubis.

## SELinux

If the Anubis Status page in EPrints shows a permission denied error, it is likely SELinux blocking apache/EPrints from making a request to Anubis' metrics.

The following (as root) can fix this by adding in a rule:

Create `anubismetrics.te` with the contents:

```

module anubismetrics 1.0;

require {
        type mysqld_tmp_t;
        type websm_port_t;
        type port_t;
        type mysqld_port_t;
        type httpd_t;
        type init_t;
        class file unlink;
        class tcp_socket name_connect;
}


#============= httpd_t ==============

#!!!! This avc can be allowed using one of the these booleans:
#     httpd_can_network_connect, httpd_can_network_connect_db
allow httpd_t mysqld_port_t:tcp_socket name_connect;

#!!!! This avc can be allowed using one of the these booleans:
#     httpd_can_network_connect, nis_enabled
allow httpd_t port_t:tcp_socket name_connect;

#!!!! This avc can be allowed using the boolean 'httpd_can_network_connect'
allow httpd_t websm_port_t:tcp_socket name_connect;

```

Create `anubismetrics.mod` with:
`checkmodule -M -m -o anubismetrics.mod anubismetrics.te`

Create `anubismetrics.pp` file with:
`semodule_package -o anubismetrics.pp -m anubismetrics.mod`

Install this rule with:
`semodule -i anubismetrics.pp`

For more help generating SELinux rules [the redhat docs](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/6/html/security-enhanced_linux/sect-security-enhanced_linux-fixing_problems-allowing_access_audit2allow) are very useful.