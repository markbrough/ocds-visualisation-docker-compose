# DOCKER COMPOSE SPA VISUALISATION

This repository contains files to get the SPA open data visualisation up and running quickly using docker compose.

There are two main components: the front end (running on VueJS) and the back-end (running on Flask/PostgreSQL). However, these rely on a few different services. Docker compose is used to get the various components working together:
* nginx (for the web server)
* postgresql (for the database back-end)
* redis (for the task queue, used for generating Excel files)
* visualisation_frontend (the frontend visualisation: a vueJS static site)
* visualisation_backend (the backend API: a Python Flask application)
* visualisation_backend_celery (the backend task queue: celery container that sends tasks to redis)

In addition, there are a couple of tasks that can be run occasionally:
* setup (to initially set up the database)
* update (to periodically - e.g. nightly - check for new data)

## Installation

### Install dependencies

```sh
sudo yum -y -q install python2 python-simplejson python-dnf libselinux-python make git
```

### User creation and limits

```sh
sudo groupadd docker

sudo adduser -G docker spage

cat <<EOF | sudo tee /etc/security/limits.conf
*  soft  nofile  32768
*  hard  nofile  32768
*  soft  nproc  31875
*  hard  nproc  31875
EOF

cat <<EOF | sudo tee /etc/sysctl.conf
vm.max_map_count=262144
EOF

sudo sysctl --system
```

### Docker and docker-compose

```sh
sudo yum -y -q install docker

sudo curl -L -o /usr/bin/docker-compose "https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m)"

sudo chmod +x /usr/bin/docker-compose

sudo systemctl enable docker
sudo systemctl start docker

sudo docker --version
# Docker version 1.13.1, build 8633870/1.13.1

docker-compose --version
# docker-compose version 1.22.0, build f46880fe
```

### Git and environment variables (if needed)

```sh
sudo -iu spage

mkdir ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

cat <<EOF >> ~/.ssh/authorized_keys
# Add SSH public key(s) here
EOF
```

#### Clone repository

```sh
git clone git@gitlab.com:spa.ge/opendata/docker-compose.git
```

or:

```
git clone https://gitlab.com/spa.ge/opendata/docker-compose.git
```

#### Modify `.env` to suit the project

```sh
cd docker-compose
vi visualisation_backend.env
```

### SSL

#### generete certs:

```sh
make configs
make create_certs
```

#### or, if selfsigned certs needed:

```sh
make selfsigned_certs
```

## Run the backend server

Run this command; the backend services will start for the first time:

```sh
docker-compose up
```

You can check everything worked by going to this URL:

http://0.0.0.0:5050/api/buyers/

You should see the following JSON output (there will initially be no data in the system - see below):

```json
{
  "data": [],
  "limit": 10,
  "offset": 0,
  "values": [
    "awards_value",
    "count_awards"
  ]
}
```

## Setup the database

Creates database tables and imports some basic reference data (e.g. lists of CPV codes). Must be run the first time to setup the database (before running the below `update` command).

```sh
docker-compose run visualisation_backend setup
```

## Update the database

Downloads new data from the SPA open data website (in zip file format). Must be run the first time to populate the database; can be run subsequently every night to fetch and import new data (overwriting existing data).

```sh
docker-compose run visualisation_backend update
```

## Set up the front end

The front end needs to be built locally so that you can specify required variables before building the container, as well as build the container on the basis of the complete set of existing data.

```sh
cd ocds-visualisation-frontend
git submodule init
git submodule update
export VISUALISATION_BACKEND_API_URL=http://127.0.0.1:5050
./build.sh
docker build --tag=visualisation_frontend .
cd ..
```

Then you can start the frontend container:

```sh
docker-compose -f docker-compose.yml -f docker-compose-visualisation-frontend.yml up
