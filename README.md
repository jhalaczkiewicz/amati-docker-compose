# Docker Compose Configuration

This repository contains a docker compose configuration useful to run TEI Publisher and associated services. Docker compose allows us to orchestrate and coordinate the various services, while keeping each service in its own, isolated environment. Setting up a server via docker compose is fast as everything comes preconfigured and you don't need to install all the dependencies (like Java, eXist-db, Python etc.) by hand. On the downside, it certainly introduces some overhead and may never be as fast as a server, which is properly maintained. For smaller, low-traffic projects docker is a viable and cheap alternative though.

For security reasons, it is recommended to not expose TEI Publisher and eXist-db directly, but instead protect them behind a proxy. The [docker-compose](docker-compose.yml) file therefore sets up an nginx reverse proxy.

The following services are configured by the [docker-compose](docker-compose.yml):

* publisher: main TEI Publisher application
* ner: TEI Publisher named entity recognition service
* frontend: nginx reverse proxy which forwards requests to TEI Publisher
* certbot: letsencrypt certbot required to register an SSL certificate
* cantaloupe: a IIIF server for images

Clone this repository to either your local machine or a server you are installing. By default it will build and deploy the main TEI Publisher application from the master branch. The named entity recognition service is pulled as image from the corresponding github package repository.If you do not need or want the named entity recognition service, comment out the corresponding section in `docker-compose.yml`, including the `depends_on: ner` above. TEI Publisher will still work.

## Default setup (local testing only)

By default, the compose configuration will launch the proxy on port 80 of the local host, serving only http, not https. This configuration is intended for testing, not for deployment on a public facing server. To start the services on localhost using the default configuration, run:

```sh
docker compose up -d --build
```

Afterwards you should be able to access TEI Publisher using http://localhost. The cantaloupe IIIF service will be mapped to the path `/iiif`, so for testing try: http://localhost/iiif/2/test.tif/full/full/0/default.jpg.

To stop the services again, call:

```sh
docker compose down
```

## Customize the docker image

The default configuration exposes the TEI Publisher application itself. Instead you may want to build and deploy a custom application generated by TEI Publisher. Clone or fork `tei-publisher-docker-compose` to apply some modifications:

By default our configuration uses the included [Dockerfile](Dockerfile) to build the application. If you already have a Dockerfile in your app repository, change the build context in `docker-compose.yml` to point to the git repository in which your Dockerfile resides. If it is a private repository, it may require an access token (see the commented out examples for `context`). You can set tokens in the [.env](.env) file.

For the included `Dockerfile` we use configuration variables to point to the application to be built. The relevant variables start with `APP_` and should be set in the [.env](.env) environment file. The file currently includes this:

```sh
# Name of the custom app to include - should correspond to the name of the repository
APP_NAME=tei-publisher-app
# Tag or branch to build
APP_TAG_OR_BRANCH=v8.0.0
# GIT repository to clone the app from
APP_REPO=https://github.com/eeditiones/tei-publisher-app.git
# eXist-db path the root of the server will be mapped to. Specifying a path here
# will map the root to one single app. Change to empty string if you want to expose the
# entire database and also set CONTEXT_PATH=auto below.
ROOT_PATH=/exist/apps/tei-publisher
# App context path: set to 'auto' if the app should be exposed under its full path 
# (e.g. /exist/apps/tei-publisher)
CONTEXT_PATH=""
# Name of the server - irrelevant on localhost
SERVER_NAME=example.com
# When building from a Dockerfile located in a private repo, you may need to set access tokens
ACCESS_TOKEN_NAME=xxx
ACCESS_TOKEN_VALUE=yyy
```

The default settings will build version 8 of TEI Publisher. To build your own custom app instead, change the three variables to point to your git repo, a tag/branch on it (e.g. `master`), and the name of the app. The latter should correspond to the repository name.

If your app **requires additional resources**, for example, because you keep your data files in a separate data package, you will have to modify the `Dockerfile` to include the necessary extra build steps.

We also assume you're app is compatible with the libraries used by TEI Publisher 8. If not, you'll have to change the specified versions in the `Dockerfile` accordingly:

```
ARG TEMPLATING_VERSION=1.1.0
ARG PUBLISHER_LIB_VERSION=4.0.0
ARG ROUTER_VERSION=1.8.0
```

To test your changed configuration locally, rebuild and restart the services (see commands below).

## Configure a remote server

Rent a cloud server which has docker enabled. There are various offers on the market. A good specification would include 4 gb of RAM and 2 vCPU, which you can get for less than 10 Euro per month.

Once you have root access to your server, ssh into it and clone the docker compose configuration repository.

Configuring the server requires four steps:

1. modify `docker-compose.yml` to set the domain name and root path
2. enable the HTTP proxy configuration
3. launch the service once to acquire an SSL certificate
4. enable the HTTPS proxy configuration

### 1. Modify `docker-compose.yml`

Assuming that you have a domain name for the server, edit [.env](.env) and set `SERVER_NAME`:

```sh
# eXist-db path the root of the server will be mapped to
ROOT_PATH=/exist/apps/tei-publisher
# Name of the server - irrelevant on localhost
SERVER_NAME=example.com
```

`ROOT_PATH` should correspond to the database path under which your application is found.

### 2. Enable HTTP proxy config

For security reasons we always hide eXist-db and TEI Publisher behind a proxy. The proxy redirects incoming requests to the configured services and blocks everything else. Our setup uses 3 configuration templates:

|Template file | Description |
|---------|----------|
| localhost.conf.template | default (enabled) for local testing only |
| default.conf.off | disabled template for HTTP access |
| default.ssl.conf.off | disabled template for HTTPS access |

For the full server setup, we have to disable the localhost configuration and enable the HTTP configuration to acquire an SSL certificate. Once this is done, we can enable the HTTPS configuration as well.

Go to the `nginx/templates` directory and either remove `localhost.conf.template` or rename it to `localhost.conf.off`, then rename `default.conf.off` to `default.conf.template`.

### 3. Acquire an SSL certificate

Before we can fully access the server, we need to get an SSL certificate. To do so:

1. Start the services (`docker compose up -d`)
2. Run the following command to request an SSL certificate for your domain, replacing the final `example.com` with your domain name:
   ```sh
   docker compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ -d example.com
   ```

   This will ask you for an email address, verify your server and store certificate files into `certbot/conf/`.

### 4. Enable HTTPS proxy config

Rename `nginx/templates/default.ssl.conf.off` to `nginx/templates/default.ssl.conf.template`, thus enabling the ssl configuration.

Now restart the services:

   ```sh
   docker compose restart
   ```

## Useful docker compose commands

Build (without cache) and start all services

```sh
docker compose up -d --build --no-cache
```

Display log output of the TEI Publisher service (i.e. eXist-db logs)

```sh
docker compose logs publisher
```

Stop all services:

```sh
docker compose down
```

## Change the server root context

The two most important configuration settings can be found in [.env](.env):

```
# eXist-db path the root of the server will be mapped to. Specifying a path here
# will map the root to one single app. Change to empty string if you want to expose the
# entire database and also set CONTEXT_PATH=auto below.
ROOT_PATH=/exist/apps/tei-publisher
# App context path: set to 'auto' if the app should be exposed under its full path 
# (e.g. /exist/apps/tei-publisher)
CONTEXT_PATH=""
```

The `ROOT_PATH` variable specifies, which database URLs will be mapped to the root, in other words: what users will see if they access the root of the server (e.g. `http://localhost`). Using the setting above, users will be presented with the entry page of the TEI Publisher app. Other applications running on the database cannot be accessed.

If – for testing purposes – you would like to expose the entire database, set `ROOT_PATH` to the empty string. In this case you should also change the `CONTEXT_PATH` setting in the `publisher` section of `docker-compose.yml` to the value `auto`:

```
# Set to 'auto' if the app should be exposed under its full path (e.g. /exist/apps/tei-publisher)
# use empty string if the app is mapped to the root of the server (see nginx config below)
CONTEXT_PATH=auto
```

## Certificate Renewal

The LetsEncrypt SSL certificate issued above will only be valid for a certain duration and needs to be renewed from time to time. We'll thus install a cron job, which calls the script `certbot-renew.sh` once every day to check if the certificate needs to be renewed.

Register a cron job to call this script once a day. Call `crontab -e` and add a line:

```
59 18 * * * /root/my-edition-docker/certbot-renew.sh
```
replacing `/root/my-edition-docker` with the correct path to wherever you cloned the configuration.

## Backing up eXist-db

If you would like to create regular backups of the data in your eXist-db:

1. edit `docker-compose.yml` and enable the volume mapping for `/exist/backup`:
   ```
   # uncomment to map eXist-db backups to local directory
   - ./backup:/exist/backup
   ```
2. retrieve the eXist-db configuration file from the running docker container with
   ```
   docker compose cp publisher:/exist/etc/conf.xml .
   ```
3. edit conf.xml and find the section referring to consistency checks and backups. Uncomment the system job, specify the backup directory and a time (cron syntax) to trigger the backup:
   ```xml
   <job type="system" name="check1" 
      class="org.exist.storage.ConsistencyCheckTask"
      cron-trigger="0 0 4 * * ?">
      <parameter name="output" value="/exist/backup"/>
      <parameter name="backup" value="yes"/>
      <parameter name="incremental" value="no"/>
      <parameter name="incremental-check" value="no"/>
      <parameter name="max" value="2"/>
   </job>
   ```
4. copy the `conf.xml` back to the docker container:
   ```
   docker compose cp conf.xml publisher:/exist/etc
   ```
5. restart the container:
   ```
   docker compose restart publisher
   ```

## Using IIIF

The included cantaloupe IIIF server reads images from a file system directory. By default this is mapped to the folder `iiif` in the same directory as the docker-compose configuration:

```yaml
volumes:
   - ./iiif:/imageroot
```

To change the directory, replace the path before the ':'.

To access cantaloupe's admin interface, edit [docker-compose.yml](docker-compose.yml), set `CANTALOUPE_ENDPOINT_ADMIN_ENABLED` to `true`, define a secret and enable the port mapping:

```sh
cantaloupe:
    image: uclalibrary/cantaloupe:5.0.5-7
    environment:
      CANTALOUPE_ENDPOINT_ADMIN_ENABLED: false
      CANTALOUPE_ENDPOINT_ADMIN_SECRET: my_admin_pass
    volumes:
      - ./iiif:/imageroot
    # comment in to enable access to cantaloupe on port 8182, including admin interface
    # ports:
    #   - 8182:8182

```

The admin interface will be available at port 8182: http://localhost:8182/admin.