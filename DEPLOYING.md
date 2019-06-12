# Deployment

This is a step by step guide covering all the necessary steps to deploy CKAN using custom template on a new machine.

The guide will try to cover all Linux-based operating systems, but some commands might be specific to Ubuntu and its derivatives. However, links containing further information were added in order to simplify the installation process on other operating systems as well.

## Installation

### Dependencies

* A GNU/Linux operating system with good internet connectivity.  
* Unrestricted access to external domains, such as **[GitHub](https://github.com/)**, **[DockerHub](https://hub.docker.com/)**, etc.
* git, docker-engine (docker) and docker-compose

First we need to install **[git](https://help.github.com/en/articles/set-up-git)** and **[docker-compose](https://docs.docker.com/compose/install/)** \(*docker-compose* should already have *docker* as dependency. If this is not the case follow the **[official documentation on installing docker](https://docs.docker.com/v17.12/install/)**\):

```
sudo apt update
sudo apt instal git docker-compose
```

Then start and enable the docker service and verify operation

```
sudo systemctl start docker
sudo systemctl enable docker
sudo docker info
```

#### Extra dependencies

- You will also need a SMTP server and its credentials for CKAN to work properly. This will not obstacle deployment, CKAN will be up and running, but won't be able to send emails (e.g. on password reset).

### Source files and configuration

Now we have the runtime, next we need to download the cluster configuration files and build the services.

Navigate to where you want the source files to live on your server (e.g. `/opt`) and clone the repository:

```
cd /opt
git clone https://github.com/ViderumGlobal/ckan-cloud-docker.git
cd ckan-cloud-docker
```

#### Environment variables

To change the default env vars used throughout the [CKAN configuration file](./docker-compose/ckan-conf-templates), adjust the secrets in `docker-compose/ckan-secrets.sh`:

```
vim docker-compose/ckan-secrets.sh
```

Also, set or adjust deployment related environment variables in the [docker-compose.yaml](./docker-compose.yaml) and [.docker-compose.vital-strategies-theme.yaml](./.docker-compose.vital-strategies-theme.yaml) Few of them worth to talk about:

Database passwords - This password will be set to the databases so make sure to update and choose them carefully
```
# in docker-compose.yaml and docker-compose-db.yaml
POSTGRES_PASSWORD=123456
DATASTORE_RO_PASSWORD=123456
DATASTORE_PUBLIC_RO_PASSWORD=123456
```

Database URL - this URLs (URIs) will be used by CKAN to connect databases. Update them according to passwords set above
```
# in docker-compose/ckan-secrets.sh
SQLALCHEMY_URL=postgresql://ckan:123456@db/ckan
CKAN_DATASTORE_WRITE_URL=postgresql://postgres:123456@datastore-db/datastore
CKAN_DATASTORE_READ_URL=postgresql://readonly:123456@datastore-db/datastore
```

SMTP service credentials - this credentials are used by CKAN to send email from Eg: password reset
```
# docker-compose/ckan-secrets.sh
SMTP_SERVER=your.mailservice.org.ro:587 #Note port number should be present
SMTP_USER=noreply@mailservice.org.ro
SMTP_PASSWORD=Secret
```

Sentry DNS - Sentry endpoint to recored and report CKAN errors (sentry extension should be enabled)
```
SENTRY_DSN=https://user:oassword@sentry.io/123456
```

#### Traefik proxy service

Traefik is the entry point from the outside world. It binds to the default HTTP (80) and HTTPS (443) ports and handles requests by forwarding them to the appropriate services within the stack. In our case, it will point to Nginx serving the CKAN web app.

Traefik needs strict permissions in order to run \(**[more info](https://www.digitalocean.com/community/tutorials/how-to-use-traefik-as-a-reverse-proxy-for-docker-containers-on-ubuntu-18-04)**\):
```
chmod 600 traefik/acme.json
```

Finally, edit traefik/traefik.toml file

```
vim traefik/traefik.toml
```

Traefik will attempt to obtain a Let's Encrypt SSL certificate. In order for this to happen, the following configuration items need to be filled in:

* `email = "admin@example.com"`

> IMPORTANT: Use FQDN for both ‘main’ and ‘rule’ entries.


  This is the [contact email](https://letsencrypt.org/docs/expiration-emails/) for Let's Encrypt
* `main = "example.com"`
  This is the domain for which Let's Encrypt will generate a certificate for

In addition to Let's Encrypt specific configuration, there is one more line you need to adjust:

* `rule = "Host:example.com"`
  This is the domain name that Traefik should respond to. Requests to any other domain not configured as a `Host` rule will result in Traefik not being able to handle the request.


> Note: All the necessary configuration items are marked with `TODO` flags in the `traefik.toml` configuration file.


This should be enough for the basic installation. In case you need to tweak versions or other initialization parameters for CKAN, you need these two files:

* `docker-compose/ckan-conf-templates/vital-strategies-theme-production.ini`
  This is the file used to generate the CKAN main configuration file.

* `.docker-compose.vital-strategies-theme.yaml`
  This is the file that defines the services used by this instance.


## Running

**To run the `vital-strategies` instance:**

```
sudo docker-compose -f docker-compose.yaml -f .docker-compose-db.yaml -f .docker-compose.vital-strategies-theme.yaml up -d --build nginx
```

**To stop it, run:**
```
sudo docker-compose -f docker-compose.yaml -f .docker-compose-db.yaml -f .docker-compose.vital-strategies-theme.yaml stop
```

**To destroy the instance, run:**
```
sudo docker-compose -f docker-compose.yaml -f .docker-compose-db.yaml -f .docker-compose.vital-strategies-theme.yaml down
```

**To destroy the instance, together with volumes, databases etc., run:**
```
sudo docker-compose -f docker-compose.yaml -f .docker-compose-db.yaml -f .docker-compose.vital-strategies-theme.yaml down -v
```

*If you want to tweak the source files, typically you need to destroy the instance and run it again once you're done editing. The choice of removing the volumes in the process is up to you.*

## Debugging

To check all the logs at any time:  
```
sudo docker-compose -f docker-compose.yaml -f .docker-compose-db.yaml -f .docker-compose.vital-strategies-theme.yaml logs -f
```  

To check the logs for a specific service:  
```
sudo docker-compose -f docker-compose.yaml -f .docker-compose-db.yaml -f .docker-compose.vital-strategies-theme.yaml logs -f ckan
```  
*(exit the logs by pressing Ctrl+C at any time)*

To get inside a running container

```
sudo docker-compose -f docker-compose.yaml -f .docker-compose-db.yaml -f .docker-compose.vital-strategies-theme.yaml exec {service-name} bash
```

Note: for some services (Eg: Nginx) you mite need to use `sh` instead of `bash`

## Creating the sysadmin user

In order to create organizations and give other user proper permissions, you will need sysamin user(s) who has all the privileges. You can add as many sysadmin as you want. To create sysamin user:

```
# Create sysadmin user using paster CLI tool
sudo docker-compose -f docker-compose.yaml -f .docker-compose-db.yaml -f .docker-compose.vital-strategies-theme.yaml \
 exec ckan /usr/local/bin/ckan-paster --plugin=ckan sysadmin add {username} password={password} email={email} -c /etc/ckan/production.ini

# Example
sudo docker-compose -f docker-compose.yaml -f .docker-compose-db.yaml -f .docker-compose.vital-strategies-theme.yaml \
 exec ckan /usr/local/bin/ckan-paster --plugin=ckan sysadmin add ckan_admin password=iemae7Ai email=info@datopian.com -c /etc/ckan/production.ini
```

You can also give sysadmin role to the existing user.

```
sudo docker-compose -f docker-compose.yaml -f .docker-compose-db.yaml -f .docker-compose.vital-strategies-theme.yaml \
 exec ckan /usr/local/bin/ckan-paster --plugin=ckan sysadmin add ckan_admin -c /etc/ckan/production.ini
```

## Sysadmin Control Panel

Here you can edit portal related configuration, like website title, site logo or add custom styling. Login as sysadmin and navigate to `ckan-admin/config` page and make changes you need. Eg: https://demo.ckan.org/ckan-admin/config


## Installing and enabling a new extension

CKAN allows installing various extensions (plugins) to the existing core setup. In order to enable/disable them you will have to install them and include into the ckan config file.

To install extension you need to modify `POST_INSTALL` section of ckan service in `.docker-compose.vital-strategies-theme.yaml`. Eg to install s3filestore extension

```
POST_INSTALL: |
  install_standard_ckan_extension_github -r datopian/ckanext-s3filestore &&\
```

And add extension to the list of plugins in `docker-compose/ckan-conf-templates/vital-strategies-theme-production.ini.template`

```
# in docker-compose/ckan-conf-templates/vital-strategies-theme-production.ini.template
ckan.plugins = image_view
   ...
   stats
   s3filestore
```

Note: depending on extension you might also need to update extensions related configurations in the same file. If needed this type of information is ussually included in extension REAMDE.

```
# in docker-compose/ckan-conf-templates/vital-strategies-theme-production.ini.template
ckanext.s3filestore.aws_access_key_id = Your-Access-Key-ID
ckanext.s3filestore.aws_secret_access_key = Your-Secret-Access-Key
ckanext.s3filestore.aws_bucket_name = a-bucket-to-store-your-stuff
ckanext.s3filestore.host_name = host-to-S3-cloud storage
ckanext.s3filestore.region_name= region-name
ckanext.s3filestore.signature_version = signature (s3v4)
```

In order to disable extension you can simple remove it from the list of plugins. You will probably also want to remove it from `POST_INSTALL` part to avoid redundant installs, although this is not must.
