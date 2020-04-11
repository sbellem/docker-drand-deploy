# Drand Deployment Scripts
Scripts to deploy a [drand](https://github.com/drand/drand) node behind nginx
with docker-compose. Both drand and nginx are run in docker containers managed
with docker-compose.

**WORK in PROGRESS**

**NOTE** This work is based on
https://github.com/drand/drand/blob/master/docker/README_docker.md#first-steps
which provides a containerized drand daemon that can be run alone or behind
nginx which is not containerized. This work includes nginx in a docker
container as well and also wishes to eventually attempt to go some steps
further in terms of automating the deployment process by using `salt` or
`ansible` to manage the provisioning of the server and also to the automate
the configuration/paramterization steps that are specific to each setup.

For now, one has to manually create an nginx server config file, and also,
possibly move the TLS certificates or edit the docker-compose.yml file
accordingly.

## Quickstart
```shell
$ git clone https://github.com/sbellem/docker-drand-deploy.git
$ cd docker-drand-deploy
```

The things to be aware of:

* (`server_name`, `public_port`) Public facing address of the drand node,
  including the port.
* (`ssl_certificate`, `ssl_certificate_key`) - Path to the ssl certificate
  files: `fullchaim.pem` and `privkey.pem`.


### TLS Certificates
If no certs, create standalone ones, using [certbot](https://certbot.eff.org):

```shell
$ sudo certbot certonly --standalone
```

Move the TLS certificates under `.letsencrypt/live/<server_name>`. Maybe they
call also be symbolically linked ... (**to try**). Also check and possibly
modify file permissions.

```shell
$ mkdir -p .letsencrypt/live/<server_name>
$ cp /path/to/certs/fullchain.pem .letsencrypt/live/<server_name>/
$ cp /path/to/certs/privkey.pem .letsencrypt/live/<server_name>/
```

**todo**:
* Can certs be symlinked to? **No.** First attempt did not work, as they need
  to mounted into the nginx container and it failed.
* Must file permissions be changed (e.g. chmod 740 ... as in
  https://github.com/drand/drand/blob/master/docker/README_docker.md#first-steps).

### nginx
Create an nginx server configuration file under
`nginx/conf.d/<server_name>.conf` based on the following template:

```nginx
server {
    listen              443 ssl http2;
    server_name         <server_name>;

    # all grpc calls
    location / {
        grpc_pass grpc://drand:8080;
    }

    # JSON REST endpoints, converted from gRPC
    location /api {
        proxy_pass http://drand:8080;
        proxy_set_header Host $host;
    }

    ssl_certificate /etc/letsencrypt/live/<server_name>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<server_name>/privkey.pem;
}
```

Replace `<server_name>` with your drand's node domain.

For instance, if drand's public facing address can be reached at

    https://drand.spring.joy:8888

and the TLS certificates (`fullchain.pem`, `privkey.pem`) have been copied
under

    .letsencrypt/live/drand.spring.joy/

and, assuming one uses the `docker-compose.yml` as is, then the config would
be somewhat similar to:

```nginx
server {
    listen              443 ssl http2;
    server_name         drand.spring.joy;

    # all grpc calls
    location / {
        grpc_pass grpc://drand:8080;
    }

    # JSON REST endpoints, converted from gRPC
    location /api {
        proxy_pass http://drand:8080;
        proxy_set_header Host $host;
    }

    ssl_certificate /etc/letsencrypt/live/drand.spring.joy/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/drand.spring.joy/privkey.pem;
}
```

### drand: generate the keypair
Make sure `.drand/` has required permissions:

```shell
$ chmod 740 .drand/
```

Generate the keypair:

```shell
$ docker-compose run --rm drand generate-keypair <server_name>:<public_port>
```

### drand: running the daemon
The provided `docker-compose.yml` file may need to be modified with respect to
the port mappings. That is, the `public_port` of the drand node must be
specified in the nginx service block.

```yml
version: '3.7'

services:
  nginx:
    image: nginx
    ports:
      - '<public_port>:443'
    # ...
```

Run it all:

```shell
$ docker-compose up
```

Test:

```shell
$ curl https://<server_name>:<public_port>/api
{"status":"drand up and running on <server_name>:<public_port>"
```

## Todo
* control port mapping, check that control commands work ...


## Acknowledgements
Work based on
https://github.com/drand/drand/blob/master/docker/README_docker.md#first-steps.
