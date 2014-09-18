buildout.webserver
=================

Buildout installing the necessary tools to run a webserver on a specific domU.
Just boostrap the buildout and run

    python bootstrap.py -c deployment.cfg
    bin/buildout -c deployment.cfg


Note:
-----

In order to make this work locally you need to take a view extra steps. Please
refer to the [setup local directory documentation](docs/setup.md)


Provided services:
------------------

* Nginx (Port 80)
* Varnish (Port 8100) - currently disabled
* runscript
* logrotation
* supervisord (controlling the isntalled zope instances)

Configuration
------------

All configuration is based in variables. In order to extend the buildout for
a new site, add or copy the relevant parts starting with "zopeX", by simply 
appending a higher number, e.g for haproxy context switching:

    acl ${sites:zope1}_cluster hdr_beg(host) -i ${hosts:zope1}
    # Check that we have at least one node up in the zope1 cluster
    acl ${sites:zope1}_cluster_up nbsrv(default) gt 0

where you would simply copy the part and replace zope1 with zope2 accordingly.

Howto
=====

In order to extend this buildout and add an additional site, follow these steps:


Update deployment.cfg
---------------------

Add a new variable/hostname pair to the *[hosts]* part, e.g.

```
static1   = example.tld
static1-1  = example2.tld 
```


Add new virtual host to "${buildout:directory}/vhosts/"
-----------------------------------------------------------

Copy the existing *example.tld* file and replace the *zopeX* variable with the 
new variable, e.g. *zope1*.


Update "/buildout.d/vhosts.cfg"
-------------------------------

Add the new vhost configuration to the *vhost.cfg* file, by appending a new
part, specifying the Zope instance location and add the corresponding
configuration file compilation

```
[buildout]
vhosts-parts =
    vhost-staticX
    ...

[site-locations]
zopeX         = /opt/sites/${sites:zopeX}/buildout.${sites:zopeX}
...

[vhost-staticX]
recipe = collective.recipe.template
input = ${locations:templates}/${sites:zopeX}.conf
output =
```


Update "/buildout.d/templates/nginx.conf"
-----------------------------------------

Finally include the new virtual host configuration in the main webserver
congfig file by adding a single line at the bottom of the file

```
include ${locations:config}/${sites:zopeX}.conf;
````

Update /buildout.d/templates/serverdetails.json
-----------------------------------------------

Add the new vhost to the server status info file we will use in "wigo.sqapp"
to display host details.

```
...
{
    "title": "${sites:zopeX}",
    "type": "plone",
    "port": "${ports:zopeX}",
    "url": "${hosts:zopeX}"
}
```

*Note:* you can simply copy the last server block in the template file, but
make sure that you separate serverblocks via `,`. The correct syntax
(simplified) would be:

``` json
{
  "server": "zope10",
  "servername": "${hosts:public}",
  "sites": [
    {
      zope1 block
    },
    {
      zope2 block
    },
    {
      zope3 block
    }
  ]
}
```


Deploy the new configuration
=============================

In the last step login to the target deployment server and update the 
configuration, example:

``` bash

$ cd /opt/webserver/buildout.webserver
$ git pull
$ bin/buildout -N -c deployment.cfg
$ bin/supervisorctl reread
$ bin/supervisorctl update
$ bin/supervisorctl start instance-zopeX
$ bin/supervisorctl restart haproxy
$ bin/supervisorctl restart nginx

```


