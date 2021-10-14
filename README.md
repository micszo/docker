# Docker blueprints

This package contains unsupported docker building blocks used for some of automated functional testing infrastructure at [Ibexa](https://ibexa.co).
Feel free to copy it for own use or look to it for some recommended settings.

**NOTE**: If you are just looking to get easily up and running and developing with Ibexa DXP,
see community-supported [eZ Launchpad](https://ezsystems.github.io/launchpad/) which is tailored for project development use cases.
_If not, be
aware of the following limitations:_

> **WARNING, made mainly for automation:** The tools within this directory are meant for use in test automation, QA,
Support and demo use cases, and with time as a blueprint for how to best configure your own setup. You are free to use
and adopt them for your needs, and we more than welcome contributions to improve it.

> **WARNING, low performance on MacOS and Windows:** For reasons mentioned above, these tools are not
optimized for use as development environment with MacOS or Windows, and are affected by known I/O performance issues caused
by Docker for MacOS/Windows use of shared folders. This is a known issue and we don't intend to add complexity to work around it.

## Overview

This setup currently requires Docker Compose 1.14 and Docker 17.06 or higher. Defaults are set in `.env`, and
files to ignore are set in `.dockerignore`. By default, `.env` specifies that the dev setup is used.

**Note:** For this and other reasons all docker-compose commands **must** be executed from root of your project directory.

#### Before you begin: Install Docker & Docker-Compose

Before going through the steps below, make sure you have recent versions of [Docker and Docker Compose](https://www.docker.com/)
installed on your machine.

*For Windows you also need to [install bash](https://msdn.microsoft.com/en-us/commandline/wsl/about), or adapt instructions below for Windows command line where needed.*


#### Concept: Docker Compose "Building blocks" for Ibexa DXP

The current Docker Compose files are made to be mixed and matched together for QA/Support use cases. Currently available:

- `base-prod.yml` (required, always needs to be first, contains: db, web and app container)
- `base-dev.yml` (alternative to `base-prod.yml`, same applies here if used)
- `create-dataset.yml` (optional, to be used together with base-prod.yml in order to set up db and vardir)
- `demo.yml` (optional, to be used together with base-prod.yml in order to set up db and vardir)
- `dfs.yml` (optional, adds DFS cluster handler. Note that you need to run the migrate script manually, see below)
- `blackfire.yml` (optional, adds a Blackfire service and lets you trigger profiling against the setup)
- `redis.yml` (optional, adds a Redis service and appends config to app)
- `redis-session.yml` (optional, stores sessions in a separate Redis instance)
- `varnish.yml` (optional, adds a Varnish service and appends config to app)
- `solr.yml` (optional, add a Solr service and configure app for it)
- `db-postgresql.yml` (optional, switches the database engine to PostgreSQL - experimental)
- `selenium.yml` (optional, always needs to be last, adds a Selenium service and appends config to app)
- `chromium.yml` (alternative to `selenium.yml`, adds headless Chrome service, same applies here if used. Experimental)
- `multihost.yml` (optional, adds multihost config to app container network)

You can use these file with the `-f` argument on docker-compose, like:

```bash
docker-compose -f doc/docker/base-prod.yml -f doc/docker/create-dataset.yml -f doc/docker/demo.yml -f doc/docker/redis.yml up -d --force-recreate
```

However below environment variable `COMPOSE_FILE` is used instead since this is also what is used to have a default in
`.env` file at root of the project.


## Project setup

### Demo "image" use

Using this approach, everything runs in containers and volumes. This means that if you for instance upload a image
using the Ibexa DXP backend, that image ends up in a volume, and not below `public/var/` in your project directory.

From the root of your project's clone of this distribution, [set up composer auth.json](#composer) and execute the following:
```sh
export COMPOSE_FILE=doc/docker/base-prod.yml:doc/docker/create-dataset.yml:doc/docker/demo.yml

# Optional step if you want to use Blackfire with the setup, change <id> and <token> with your own values
#export COMPOSE_FILE=doc/docker/base-prod.yml:doc/docker/create-dataset.yml:doc/docker/demo.yml:doc/docker/blackfire.yml BLACKFIRE_SERVER_ID=<id> BLACKFIRE_SERVER_TOKEN=<token>

# First time: Install setup, and generate database dump:
docker-compose -f doc/docker/install-dependencies.yml -f doc/docker/install-database.yml up --abort-on-container-exit

# Optionally, build dbdump and vardir images.
# The dbdump image is created based on doc/docker/entrypoint/mysql/2_dump.sql which is created by above command
# The vardir image is created based on the content of public/var
# If you don't build these image explicitly, they will automaticly be builded later when running `docker-compose up`
docker-compose build dataset-vardir dataset-dbdump

# Boot up full setup:
docker-compose up -d --force-recreate
```

After about 5-10 seconds you should be able to browse the site on `localhost:8080` and the backend on `localhost:8080/admin`.

### Development "mount" use

Whe you use this approach, your project directory is bind-mounted into the Nginx and PGP containers.
If you change a PHP file in, for instance `src`, that change is applied in automatically.

Warning: *Dev setup works a lot faster on Linux then on Windows/Mac where Docker uses virtual machines with shared folders
by default under the hood, which leads to much slower IO performance.*

From the root of your project's clone of this distribution, [set up composer auth.json](#composer) and execute the following:

```sh
# Optional: If you use Docker Machine with NFS, you'll need to specify where project is, & give composer a valid directory.
#export COMPOSE_DIR=/data/SOURCES/MYPROJECTS/ezplatform/doc/docker COMPOSER_HOME=/tmp

# First time: Install setup, and generate database dump:
docker-compose -f doc/docker/install-dependencies.yml -f doc/docker/install-database.yml up --abort-on-container-exit

# Boot up full setup:
docker-compose up -d --force-recreate
```


After about 5-10 seconds you should be able to browse the site on `localhost:8080` and the backend on `localhost:8080/admin`.


_TIP: If you are seeing 500 errors, or in the case of `SYMFONY_ENV=dev` database exceptions, then make sure to comment out `database_*` params in `app/config/parameters.yml` to make sure env variables are used correctly._

### Behat and Selenium use

*Docker Compose setup for Behat use is provided and used internally to test Ibexa DXP. It can be combined with most
setups, here shown in combination with production setup which is what you typically need to test before pushing your
image to Docker Hub/Registry.*

From the root of your project's clone of this distribution, [set up composer auth.json](#composer) and execute the following:

```sh
export COMPOSE_FILE=doc/docker/base-prod.yml:doc/docker/selenium.yml

# First time: Install setup, and generate database dump:
docker-compose -f doc/docker/install-dependencies.yml -f doc/docker/install-database.yml up --abort-on-container-exit

# Boot up full setup:
docker-compose up -d --force-recreate
```

The last step is to execute Behat scenarios using `app` container which now has access to web and Selenium containers, for example:
```
docker-compose exec --user www-data app sh -c "php /scripts/wait_for_db.php; php bin/behat -vv --profile=rest --suite=fullJson --tags=~@broken"
```


*Tip: You can typically re-run the installation command to get back to a clean installation between Behat runs by using:*
```
docker-compose exec --user www-data app composer ezplatform-install
```

Note: if you want to use the Chromium driver, use:
```
export COMPOSE_FILE=doc/docker/base-prod.yml:doc/docker/chromium.yml
```
This driver is not fully supported in our test suite and is in experimental state.

### DFS

If you want to use the DFS cluster handler, you need to run the migration script manually, after starting the
containers ( run `docker-compose up -d --force-create` first).

The migration script copies the binary files in public/var to the nfs mount point (`./dfsdata`) and adds the files'
metadata to the database. If your are going to run Ibexa DXP in a cluster you must then ensure that `./dfsdata` is a mounted
nfs share on every node/app container.

```
# Enter the app container
docker-compose exec --user www-data app /bin/bash

# Inside app container
php app/console ezplatform:io:migrate-files --from=default,default --to=dfs,nfs --env=prod

```

Once this is done, you may delete `public/var/*` if you don't intend to run the migration scripts ever again.

### Production use

#### Example: Building app with php image

In this example we'll build a app image which includes both php (php_fpm) and the Ibexa DXP application and run them
in a swarm cluster using docker stack.

Prerequisite:
- A running [swarm cluster](https://docs.docker.com/engine/swarm/swarm-tutorial/) (a one-node cluster is sufficient for running this example)
- A running NFS server. How to configure a nfs server is distro dependent, but this [ubuntu guide](https://help.ubuntu.com/community/NFSv4Howto) might be of help
- A running [docker registry](https://docs.docker.com/registry/deploying/#managing-with-compose) (Only required if your swarm cluster has more than one node)

In this example we assume your swarm manager is named `swarmmanager` and that this hostname resolves on all swarm hosts. We also assume that the nfs server and docker registry are running on `swarmmanager`.

All the commands below should be executed on your `swarmmanager`

```sh
# If not already done, install setup, and generate database dump :
docker-compose -f doc/docker/install-dependencies.yml -f doc/docker/install-database.yml up --abort-on-container-exit

# Build docker_app and docker_web images ( php and nginx )
docker-compose -f doc/docker/base-prod.yml build --no-cache app web

# Build varnish image
docker-compose -f doc/docker/base-prod.yml -f doc/docker/varnish.yml build --no-cache varnish

# Create dataset images ( my-ez-app-dataset-dbdump and my-ez-app-dataset-vardir )
# The dataset images contains a dump of the database and a dump of the var/ files ( located in public/var )
docker-compose -f doc/docker/create-dataset.yml build --no-cache

# Tag the images
docker tag docker_dataset-dbdump swarmmanager:5000/my-ez-app/dataset-dbdump
docker tag docker_dataset-vardir swarmmanager:5000/my-ez-app/dataset-vardir
docker tag docker_web swarmmanager:5000/my-ez-app/web
docker tag docker_app swarmmanager:5000/my-ez-app/app
docker tag docker_varnish swarmmanager:5000/my-ez-app/varnish

# Upload the images to the registry ( only needed if your swarm cluster has more than one node)
docker push swarmmanager:5000/my-ez-app/dataset-dbdump
docker push swarmmanager:5000/my-ez-app/dataset-vardir
docker push swarmmanager:5000/my-ez-app/web
docker push swarmmanager:5000/my-ez-app/app
docker push swarmmanager:5000/my-ez-app/varnish

# In this example we run the database in a separate stack so that you may easily have multiple Ibexa DXP installations using the same database instance
docker stack deploy --compose-file doc/docker/db-stack.yml stack-db

# Now, wait a half a minute to ensure that the database is ready to accept incomming requests before continuing

# Now, load the database dump into the db and the var dir to the nfs server
docker-compose -f doc/docker/import-dataset.yml up

# Finally, create the Ibexa DXP stack
docker stack deploy --compose-file doc/docker/my-ez-app-stack.yml my-ez-app-stack

# Cleanup
# If you want to remove the stacks again:
docker stack rm my-ez-app-stack
sleep 15
docker stack rm stack-db
sleep 15
docker volume rm my-ez-app-stack_vardir
docker volume rm stack-db_mysql
```

#### Example: Separating app and php

In this alternative way of running Ibexa DXP, the Ibexa DXP code and PHP executables are separated in two different
images. The upside of this is that it gets easier to upgrade PHP (or any other distro applications) independently
of Ibexa DXP. To do it, replace the PHP container with an updated one without having to rebuild the Ibexa DXP
image. The downside of this approach is that all Ibexa DXP code is copied to a volume so that it can be shared with
other containers. This means bigger disk space footprint and longer loading time of the containers.
It is also more complicated to make this approach work with docker stack so only a docker-compose example is provided.

```sh
export COMPOSE_FILE=doc/docker/base-prod.yml:doc/docker/create-dataset.yml:doc/docker/distribution.yml
# If not already done, install setup, and generate database dump :
docker-compose -f doc/docker/install-dependencies.yml -f doc/docker/install-database.yml up --abort-on-container-exit

# Build docker_app and docker_web images ( php and nginx )
# The docker_app image (which contain both php and Ibexa DXP) will be used as base image when creating the image which
# only contains the Ibexa DXP Platform files.
docker-compose -f doc/docker/base-prod.yml build --no-cache app

# Optional, only build the images, do not create containers
docker-compose build --no-cache distribution

# Note that if you set the environment variable COMPOSE_PROJECT_NAME to a non-default value, you'll need to use set the
# build argument DISTRIBUTION_IMAGE when building the distribution image
docker-compose build --no-cache --build-arg DISTRIBUTION_IMAGE=customprojectname_app distribution

# Build the "distribution" and dataset images, then start the containers
docker-compose up -d
```

## Further info

### <a name="composer"></a>Configuring Composer

For Composer to run correctly as part of the build process, you need to create a `auth.json` file in your project root with your GitHub readonly token:

```sh
echo "{\"github-oauth\":{\"github.com\":\"<readonly-github-token>\"}}" > auth.json
# If you use Ibexa Content,Experiece or Commerce also include your updates.ibexa.co auth token
echo "{\"github-oauth\":{\"github.com\":\"<readonly-github-token>\"},\"http-basic\":{\"updates.ibexa.co\": {\"username\":\"<installation-key>\",\"password\":\"<token-pasword>\",}}}" > auth.json
```

For further information on tokens for updates.ibexa.co, see [the installation guide](https://doc.ibexa.co/en/latest/install/#set-up-authentication-tokens).

### Debugging

For checking logs from the containers themselves, use `docker-compose logs`. Here on `app` service, but can be omitted to get all:
```sh
docker-compose logs -t app
```


You can login to any of the services using `docker-compose exec`, here shown against `app` image and using `bash`:
```sh
docker-compose exec app /bin/bash
```

To display running services:
```sh
docker-compose ps
```

### Database dumps

Database dump is placed in `doc/docker/entrypoint/mysql/`. This folder is used my mysql/mariadb which executes
everything inside the folder. This means there should only be data represent one install in the folder at any given time.


### Updating service images

To updated the used service images, you can run:
```sh
docker-compose pull --ignore-pull-failures
```

This assumed you either use `docker-compose -f` or have `COMPOSE_FILE` defined in cases where you use something else
then defaults in `.env`.

After this you can re run the production or dev steps to setup containers again with updated images.

### Cleanup

Once you are done with your setup, you can stop it, and remove the involved containers.
```sh
docker-compose down -v
```

And if you have defined any environment variables you can unset them using:
```sh
unset COMPOSE_FILE COMPOSE_DIR COMPOSER_HOME

# To unset blackfire variables
unset BLACKFIRE_SERVER_ID BLACKFIRE_SERVER_TOKEN
```

## COPYRIGHT

Copyright (C) 1999-2021 Ibexa AS (formerly eZ Systems AS). All rights reserved.

## LICENSE

This source code is available separately under the following licenses:

A - Ibexa Business Use License Agreement (Ibexa BUL),
version 2.4 or later versions (as license terms may be updated from time to time)
Ibexa BUL is granted by having a valid Ibexa DXP (formerly eZ Platform Enterprise) subscription,
as described at: https://www.ibexa.co/product
For the full Ibexa BUL license text, please see:
- LICENSE-bul file placed in the root of this source code, or
- https://www.ibexa.co/software-information/licenses-and-agreements (latest version applies)

AND

B - GNU General Public License, version 2
Grants an copyleft open source license with ABSOLUTELY NO WARRANTY. For the full GPL license text, please see:
- LICENSE file placed in the root of this source code, or
- https://www.gnu.org/licenses/old-licenses/gpl-2.0.html
