# docker-sentry

A Dockerfile to run Sentry in a container.

A mix of:
- http://iromli.github.io/tracing-err-talk/#/
- https://github.com/grue/docker-sentry

Configured to work with a PostgreSQL database and a Nginx reverse-proxy.


## Current status

This is not working, I can't make Nginx connect to the upstream Sentry
app... it always returns "502 BAD GATEWAY"... Help needed!


## Tutorial

### Purpose

Installing Sentry on Docker using:

- a PostgreSQL container,
- a Sentry app container
- a Nginx reverse-proxying container,
- a volume container for the PostgreSQL data.

The whole tutorial will guide you into making Sentry available on a public
machine through Docker-magic!


### Short version (just the commands)

Unless you've already done it with the long version, I recommend you skip
to the next chapter before running this in your terminal!

```
docker run -v /var/log/postgresql -v /data --name sentry-postgresql-data busybox true
docker run -d --name="sentry-postgresql" -e USER="sentry" -e DB="sentry" -e PASS="$(pwgen -s -1 16)" --volumes-from=sentry-postgresql-data rchampourlier/postgresql
docker run -d --name "sentry-app" -e URL_PREFIX="http://sentry.jobteaser.net" -p 127.0.0.1:9000:9000 --link sentry-postgresql:db rchampourlier/sentry start http

# Create Sentry super-user (only the first time)
docker run -i -t --rm --link sentry-postgresql:db rchampourlier/sentry createsuperuser

docker run -d --name=nginx -p 80:80 -p 443:443 -v /nginx/sites-enabled:/etc/nginx/sites-enabled -v /nginx/log:/var/log/nginx dockerfile/nginx
```


### Detailed version

#### 1. Create the volume containers

```
docker run -v /var/log/postgresql -v /data --name sentry-postgresql-data busybox true
```

(These commands create a container which returns instantly. There is nothing to run,
but the volume container is ready for use.)

#### 2. Run PostgreSQL

We run PostgreSQL with the specified database, user and password. The
PostgreSQL data is handled by the volumes container we just created.

```
docker run -d --name="sentry-postgresql" -p 127.0.0.1:5432:5432 -e USER="sentry" -e DB="sentry" -e PASS="$(pwgen -s -1 16)" --volumes-from=sentry-postgresql-data rchampourlier/postgresql
```

Details:

- we run the container in the background with `-d` (detach),
- we name it `postgresql`,
- we publish the `5432` port to the host so that we may try and connect to our
  PostgreSQL database,
- we setup some environment variables for the container to use to setup
  PostgreSQL and the database.

If you need or if you want to test, here's how you can now connect to your
database (you will need postgresql-client packages):

```
psql -h 127.0.0.1 -p 5432 -U sentry
# Enter the password
```

#### 3. Run the Sentry app container

Now you can run the Sentry app:

```
--COMPLETE--
```

If this is the first time, you will need to create the Sentry superuser.

```
docker run -i -t --rm --link sentry-postgresql:db rchampourlier/sentry createsuperuser
```

Details:

- we run the container in the background (`-d`),
- we name it `sentry-app`,
- we connect the container to the PostgreSQL container, aliasing it to `db`
  (with `--link sentrypostgresql:db`): this gives the container access to
  the PostgreSQL container's network and environment variables, which are
  registered in the Sentry configuration file (`/opt/sentry.conf.py` in the
  container),
- we expose the port 9000 (Sentry app's) to have access from the host.


#### 4. Run the Nginx reverse-proxy

Create the Nginx conf file for Sentry:

```
mkdir /nginx
mkdir /nginx/log
mkdir /nginx/sites-enabled
echo "
server {
  listen            80;
  server_name       <your.hostname>;

  proxy_set_header  Host              \$host;
  proxy_set_header  X-Real-IP         \$remote_addr;
  proxy_set_header  X-Forwarded-For   \$proxy_add_x_forwarded_for;
  proxy_set_header  X-Forwarded-Proto \$scheme;
  proxy_redirect    off;

  # keepalive + raven.js is a disaster
  keepalive_timeout 0;

  # use very aggressive timeouts
  proxy_read_timeout 5s;
  proxy_send_timeout 5s;
  send_timeout 5s;
  resolver_timeout 5s;
  client_body_timeout 5s;

  # buffer larger messages
  client_max_body_size 150k;
  client_body_buffer_size 150k;

  location / {
    proxy_pass        http://localhost:9000;
  }

  location ~* /api/(?P<projectid>\d+/)?store/ {
    proxy_pass        http://localhost:9000;

    limit_req   zone=one  burst=3  nodelay;
    limit_req   zone=two  burst=10  nodelay;
  }
}
" > /nginx/sites-enabled/docker-sentry.conf
```

Now that our Nginx configuration is ready in our `nginx-sites-enabled` directory,
we're ready to start the Nginx container.

```
docker run -d --name=nginx -p 80:80 -p 443:443 -v /nginx/sites-enabled:/etc/nginx/sites-enabled -v /nginx/log:/var/log/nginx dockerfile/nginx
```

#### (Bonus) Having some fun

#### Destroying the containers... what?!

What's great with Docker is that you can just trash your containers, it's OK.
Well, as soon as your correctly using volume containers to save your data...

In our case, it's OK and we can just destroy the `sentry-app` and `sentry-postgresql`
containers and restart them, we'll be just fine!

```
docker rm -f sentry-app
docker rm -f sentry-postgresql
```

#### Accessing the data container

```
docker run -t -i -v /var/log/postgresql -v /data --volumes-from=sentry-postgresql-data --rm ubuntu /bin/sh
```

#### Some info on the images used

##### PostgreSQL

[rchampourlier/postgresql](https://index.docker.io/u/rchampourlier/postgresql/), a fork of [paintedfox/postgresql](https://index.docker.io/u/paintedfox/postgresql) fixing the issue when reusing data (for example when using a persistent data volume like we do).

##### Sentry

[rchampourlier/sentry](https://index.docker.io/u/rchampourlier/sentry/) which is a mix of 2 approaches:

- [grue/docker-sentry](https://index.docker.io/u/grue/docker-sentry/): everything is configurable through environment variables but we'd rather use the linking feature to connect our DB.
- [A presentation to run Sentry on Docker](iromli.github.io/tracing-err-talk/#/38), but it uses MySQL instead of PostgreSQL and was missing the spoken part.