# docker-sentry

A Dockerfile to run Sentry in a container.

A mix of:
- http://iromli.github.io/tracing-err-talk/#/
- https://github.com/grue/docker-sentry

Configured to work with a PostgreSQL database and a Nginx reverse-proxy.

## Tutorial

### Purpose

Installing Sentry on Docker using:

- a PostgreSQL container,
- a Sentry app container
- a Nginx reverse-proxying container,
- a volume container for the PostgreSQL data.

The whole tutorial will guide you into making Sentry available on a public
machine through Docker-magic!


### Short version

```
git clone https://github.com/rchampourlier/docker-sentry.git
cd docker-sentry
cp -Rf conf /

# Edit the configuration files in /conf. You should change:
#   - the `server_name` (replace `sentry.example.com` with your own hostname
#     in `/conf/nginx/sentry.conf.tmpl`,
#   - in `/conf/sentry/sentry.conf.py`:
#     - the `SECRET_KEY` (you may use [this site](http://www.miniwebtool.com/django-secret-key-generator/))
#       to generate a Django key,
#     - the `SENTRY_URL_PREFIX` (again, replace `sentry.example.com`).

# Dependencies
apt-get install pwgen -y

docker run -v /data --name sentry-postgresql-data busybox true
docker run -d --name="sentry-postgresql" -e USER="sentry" -e DB="sentry" -e PASS="$(pwgen -s -1 16)" -v /var/log/postgresql:/var/log/postgresql --volumes-from=sentry-postgresql-data rchampourlier/postgresql
docker run -d --name="sentry-app" -v /conf/sentry:/opt --link sentry-postgresql:db rchampourlier/sentry start http

# The first time, we create the Sentry superuser
docker run -i -t --rm --link sentry-postgresql:db rchampourlier/sentry createsuperuser

docker run -d --name=nginx -p 80:80 -p 443:443 -v /conf/nginx/www:/var/www -v /conf/nginx/sites-templates:/etc/nginx/sites-templates -v /var/log/containers/nginx:/var/log/nginx --link sentry-app:sentry shepmaster/nginx-template-image
```


### Detailed version


#### 1. Setup configuration

Configuration for Sentry and Nginx will be provided by the host in the
`/conf` directory.

```
git clone https://github.com/rchampourlier/docker-sentry.git
cd docker-sentry
cp -Rf conf /
```

If you need to customize the configuration, edit the files in `/conf`.


#### 2. Create the volume container

```
docker run -v /data --name sentry-postgresql-data busybox true
```

This command creates a container which will store our PostgreSQL data.


#### 3. Run PostgreSQL

We run PostgreSQL with the specified database, user and password. The
PostgreSQL data is handled by the volumes container we just created.

```
docker run -d --name="sentry-postgresql" -e USER="sentry" -e DB="sentry" -e PASS="$(pwgen -s -1 16)" -v /var/log/postgresql:/var/log/postgresql --volumes-from=sentry-postgresql-data rchampourlier/postgresql
```

Details:

- we run the container in the background with `-d` (detach),
- we name it `sentry-postgresql`,
- we setup some environment variables for the container to use to setup
  PostgreSQL and the database,
- we connect the `/var/log/postgresql` directory on the container to the host',
- we connect the `/data` directory to our dedicated volume container.


#### 4. Run the Sentry app container

```
docker run -d --name="sentry-app" -v /conf/sentry:/opt --link sentry-postgresql:db rchampourlier/sentry start http
```

Details:

- we run the container in the background (`-d`),
- we name it `sentry-app`,
- we connect the container to the PostgreSQL container, aliasing it to `db`
  (with `--link sentrypostgresql:db`): this gives the container access to
  the PostgreSQL container's network and environment variables;
- we mount the host `/conf/sentry` directory on `/opt` so that the Sentry
  app can use the `sentry.conf.py` file on the host as its configuration.


If Sentry has not yet been configured with the current database, we have to
create the Sentry superuser before being able to connect.

```
docker run -i -t --rm --link sentry-postgresql:db rchampourlier/sentry createsuperuser
```

#### 5. Run the Nginx reverse-proxy

```
docker run -d --name=nginx -p 80:80 -p 443:443 -v /conf/nginx/www:/var/www -v /conf/nginx/sites-templates:/etc/nginx/sites-templates -v /var/log/containers/nginx:/var/log/nginx --link sentry-app:sentry shepmaster/nginx-template-image
```

### Maintenance

You should regularly cleanup your Sentry DB, otherwise it may get a bit too big... Let's do this by cleaning the entries more than 360-day old:

```
docker run --name="sentry-app-cleanup" -v /conf/sentry:/opt --link sentry-postgresql:db rchampourlier/sentry cleanup --days 360
```

### Troubleshooting

If you need to inspect the `sentry-app` container because something's wrong, run this:

```
docker run -i -t --entrypoint=/bin/sh --name="sentry-app" -v /conf/sentry:/opt --link sentry-postgresql:db rchampourlier/sentry -c bash
```

### Some info on the images used

#### PostgreSQL

[rchampourlier/postgresql](https://index.docker.io/u/rchampourlier/postgresql/), a fork of [paintedfox/postgresql](https://index.docker.io/u/paintedfox/postgresql) fixing the issue when reusing data (for example when using a persistent data volume like we do).

#### Sentry

[rchampourlier/sentry](https://index.docker.io/u/rchampourlier/sentry/) is a mix of 2 inspirations:

- [grue/docker-sentry](https://index.docker.io/u/grue/docker-sentry/): everything is configurable through environment variables but we'd rather use the linking feature to connect our DB.
- [A presentation to run Sentry on Docker](iromli.github.io/tracing-err-talk/#/38), but it uses MySQL instead of PostgreSQL and was missing the spoken part.
