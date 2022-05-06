<div id="top"></div>

<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://prestashop.com">
    <img src="https://www.prestashop.com/sites/all/themes/prestashop/images/logos/logo-fo-prestashop-colors.svg" alt="Logo" width="420" height="80">
  </a>

  <h1 align="center">Dev tools for SaaS App</h1>
</div>

<!-- TABLE OF CONTENTS -->
<details open>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li>
          <a href="#installation">Installation</a>
          <ul>
            <li><a href="#quick-install">Quick Install</a></li>
          </ul>
        </li>
        <li>
          <a href="#configuration">Configuration</a>
          <ul>
            <li><a href="#environment-variables">Environment variables</a></li>
            <li><a href="#custom-install">Custom install</a></li>
          </ul>
        </li>
      </ul>
    </li>
    <li>
      <a href="#usage">Utils</a>
    </li>
    <li>
      <a href="#troubleshooting">Troubleshooting</a>
      <ul>
        <li><a href="#mac-os">MacOS</a></li>
        <li><a href="#common-problems">Common Problems</a></li>
      </ul>
    </li>
    <li>
      <a href="#what-next">What next ?</a>
      <ul>
        <li><a href="#saas-app-documentation">SaaS App documentation</a></li>
        <li><a href="#saas-app-module-example">SaaS App module example</a></li>
      </ul>
    </li>
  </ol>
</details>


<!-- ABOUT THE PROJECT -->
## 🧐 About The Project

This tool will help you set up your development environement to kickstart the creation of a PrestaShop SaaS App.

You will find an exemple of a simple SaaS App Module within the `rbm_example` folder.

Once you launch the services through docker-compose, you will get access to a PrestaShop instance configured with all the needed modules.

<!-- GETTING STARTED -->
## 💡 Getting Started

This is an example of how you may setup your project locally.

### Prerequisites

🐳 [Docker and docker-compose installed](https://www.docker.com/products/docker-desktop)

### Installation

#### Quick install
2. Configure git to ignore [core.fileMode](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corefileMode)
```sh
git config core.fileMode false
```
2. Clone the repo
```sh
git clone https://github.com/PrestaShopCorp/rbm-devtools.prestashop.com.git
```
3. Create your dot env file
```sh
cp .env.example .env
```
4. Customize your .env
```sh
MODULE_NAME=CHANGEME123
PORT=8080
PMA_PORT=8081
DB_PORT=3307
```
> 💡 more info in <a href="#environment-variables">Environment variables</a> section

5. Run the project
```sh
./install.sh
```

<p align="right">(<a href="#top">back to top</a>)</p>

#### Install RBM example

1. Follow the instruction in [README.md](modules/rbm_example/README.md)

2. Search RBM Example module within the Module Catalog

3. Click on "Install" button

<p align="right">(<a href="#top">back to top</a>)</p>

### Configuration
#### Environment variables

* ``PS_NAME`` - Define the subdomain for the http tunnel (default value: CHANGEME123)
> 💡 ``PS_NAME`` is automatically generated by localtunnel client
* ``MODULE_NAME`` - Define the subdomain for the http tunnel (default value: CHANGEME123)
> 💡 ``MODULE_NAME`` needs to be setup before installing
* ``TUNNEL_DOMAIN`` - Define tunnel domain (default value: localtunnel.distribution.prestashop.net)
> 💡 ``TUNNEL_DOMAIN`` can be changed if you host your own [localtunnel server](https://github.com/localtunnel/server)
* ``PS_LANGUAGE`` - Change the default language installed on PrestaShop (default value: en)
* ``PS_COUNTRY`` - Change the default country installed on PrestaShop (default value: GB)
* ``PS_ALL_LANGUAGES`` - Install all the existing languages for the current version. (default value: 0)
* ``ADMIN_MAIL`` - Override default admin mail (default value: admin@prestashop.com)
* ``ADMIN_PASSWD`` - Override default admin password (default value: prestashop)
* ``PORT`` - Define port of the PrestaShop and the http proxy client (default value: 8080)
* ``PMA_PORT`` - Define port of PhpMyAdmin (default value: 8081)
* ``DB_PORT`` - Define port of the MySQL (default value: 3307)


<p align="right">(<a href="#top">back to top</a>)</p>

#### Custom install

Let's break the install.sh down: 

First we create a network layer for our container. If the network already exists, we skip this part: 
```sh
# Create a network for containers to communicate
NETWORK_NAME=prestashop_net
if [ -z $(docker network ls --filter name=^${NETWORK_NAME}$ --format="{{ .Name }}") ] ; then
  echo -e "Create ${NETWORK_NAME} network for containers to communicate"
  docker network create ${NETWORK_NAME} ;
else
  echo -e "Network ${NETWORK_NAME} already exists, skipping"
fi
```

> 💡 If you want to use another network you will need to edit docker-compose.yml

We create an http tunnel container, which will generate your shop url
```sh
# Create http tunnel container
echo -e "Create HTTP tunnel service"
if [[ `uname -m` == 'arm64' ]]; then
  DOCKER_SCAN_SUGGEST=false docker-compose -f docker-compose.yml -f docker-compose.arm64.yml up -d --no-deps --force-recreate --build prestashop_tunnel
else
  DOCKER_SCAN_SUGGEST=false docker-compose up -d --no-deps --force-recreate --build prestashop_tunnel
fi

echo -e "Checking if HTTP tunnel is available..."
LOCAL_TUNNEL_READY=`docker inspect -f {{.State.Running}} ps-tunnel.local`
until (("$LOCAL_TUNNEL_READY"=="true"))
do
  echo -e "Waiting for confirmation of HTTP tunnel service startup"
  sleep 5
done;
echo -e "HTTP tunnel is available, let's continue !"
```

We setup the .env file with your subdomain, if .env already exist we skip this part and we replace ``PS_NAME``
```sh
# Setting up env file
echo -e "Setting up env file"

ENV_FILE=.env
SUBDOMAIN_NAME=`docker logs ps-tunnel.local 2>/dev/null | awk -F '/' '{print $3}' | awk -F"." '{print $1}' | awk 'END{print}' | tr -d "[:space:]"`
if [ ! -s "$ENV_FILE" ]; then
  echo -e "Create env file"
  cp .env.example $ENV_FILE
fi

sed -i $SED_OPTIONS -E "s|(PS_NAME=).*|PS_NAME=${SUBDOMAIN_NAME}|g" $ENV_FILE
if [[ "$OSTYPE" == "darwin"* ]]; then
  rm -f "${ENV_FILE}${SED_OPTIONS}"
fi
PS_NAME=$(readEnv PS_NAME $ENV_FILE)
```

> 💡 Note: you can customize the .env file, see below in <a href="#environment-variables">Environment variables</a>


We do a small trick to always get the same URL
```sh
# Handle restart to avoid new subdomain
TUNNEL_FILE=tunnel/.config
if [ ! -s "$TUNNEL_FILE" ]; then
  echo -e "Handle restart to avoid new subdomain"
  echo $SUBDOMAIN_NAME > $TUNNEL_FILE
fi
docker cp $TUNNEL_FILE ps-tunnel.local:/tmp/.config
```

We create the MySQL and PrestaShop container
```sh
# Create MySQL and PrestaShop service
echo -e "Create MySQL & PrestaShop service"
if [[ `uname -m` == 'arm64' ]]; then
  DOCKER_SCAN_SUGGEST=false docker-compose -f docker-compose.yml -f docker-compose.arm64.yml up -d --no-deps --force-recreate --build prestashop_rbm_shop prestashop_rbm_db
else
  DOCKER_SCAN_SUGGEST=false docker-compose up -d --no-deps --force-recreate --build prestashop_rbm_db prestashop_rbm_shop
fi
```

We wait until PrestaShop is ready to use
```sh
echo -e "Checking if PrestaShop is available..."
LOCAL_PORT=$(readEnv PORT $ENV_FILE)
if [ -z $LOCAL_PORT ] ; then
  LOCAL_PORT=`docker port ps-rbm.local 80 | awk -F ':' '{print $2}' | tr -d "[:space:]"`
fi

PRESTASHOP_READY=`curl -s -o /dev/null -w "%{http_code}" localhost:$LOCAL_PORT`
until (("$PRESTASHOP_READY"=="302"))
do
  # avoid infinite loop...
  PRESTASHOP_READY=`curl -s -o /dev/null -w "%{http_code}" localhost:$LOCAL_PORT`
  echo "Waiting for confirmation of PrestaShop is available (${PRESTASHOP_READY})"
  sleep 5
done;

```

Finaly, we display your URL
```sh
BO Url: http://CHANGEME123.localtunnel.distribution.prestashop.net/admin-dev
FO Url: http://CHANGEME123.localtunnel.distribution.prestashop.net
```

<p align="right">(<a href="#top">back to top</a>)</p>


<!-- USAGE EXAMPLES -->
## Utils

Get your shop URL
``` sh
./get-url.sh

BO Url: http://CHANGEME123.localtunnel.distribution.prestashop.net/admin-dev
FO Url: http://CHANGEME123.localtunnel.distribution.prestashop.net
```

Update the shop URL
``` sh
./update-domain.sh
Updating PrestaShop domains ...

BO Url: http://CHANGEME123.localtunnel.distribution.prestashop.net/admin-dev
FO Url: http://CHANGEME123.localtunnel.distribution.prestashop.net
```


## 🐛 Troubleshooting

### Mac OS
[Mac network_mode: "host" not working as expected](https://docs.docker.com/desktop/mac/networking/#known-limitations-use-cases-and-workarounds)

### Error on database port

```
ERROR: for prestashop_rbm_db  Cannot start service prestashop_rbm_db: Ports are not available: listen tcp 0.0.0.0:3307:
```

You should override the port `DB_PORT` in the [environement file](#environment-variables)

### Database doesn't start

If the database doesn't start because of this error :

```
[ERROR] Plugin 'InnoDB' init function returned error.
[ERROR] Plugin 'InnoDB' registration as a STORAGE ENGINE failed.
```

You should delete the `./mysql/` folder and relaunch everything.

### Error within ps-tunnel.local (only on Windows)

If you get an error on line 14 of /tmp/run.sh, you should convert the end of line to Unix format, then relaunch the `./install.sh` command.


## 🚀 What next ?

### SaaS App documentation

Documentation about developping a SaaS App is available [here](https://billing-docs.netlify.app/).

## SaaS App example

See module [README.md](/modules/rbm_example/README.md)

<p align="right">(<a href="#top">back to top</a>)</p>