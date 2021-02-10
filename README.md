![ejabberd logo](./.assets/ejabberd_logo.png)

# MELK - Movim, ejabberd, LDAP and Keycloak

ejabberd XMPP Server with Keycloak registration, LDAP authentication and PostgreSQL Storage

## Perquisites

curl, jq, Python with python-dotenv and jinja2

```shell
sudo apt install curl jq
sudo pip install python-dotenv jinja2
```

## ejabberd

### Config Key Features

```yaml
hosts:
  - {{ env['HOST'] }}.{{ env['DOMAIN'] }}

host_config:
  whalesanctuary.synology.me:
    auth_method: ldap
    ldap_servers: 
      - slapd
    ldap_rootdn: "cn=readonly,{{ env['LDAP_BASE_DN'] }}"
    ldap_password: {{ env['LDAP_READONLY_USER_PASSWORD'] }}
    ldap_encrypt: starttls
    ldap_port: 389
    ldap_base: {{ env['LDAP_BASE_DN'] }}
    ldap_filter: (objectClass=person)
    ldap_uids:
      - uid
    sql_type: pgsql
    sql_server: db
    sql_database: ejabberd
    sql_username: ejabberd
    sql_password: {{ env['EJABBERD_POSTGRES_PASSWORD'] }}
    
default_db: sql
new_sql_schema: true

...

  -
    port: 5280
    ip: "::"
    module: ejabberd_http
    request_handlers:
      "/admin": ejabberd_web_admin
      "/.well-known/acme-challenge": ejabberd_acme

...

acme:
  contact: "mailto:{{ env['EJABBERD_ADMIN_USER'] }}"
  ca_url: https://acme-v02.api.letsencrypt.org/directory
  # ca_url: https://acme-staging-v02.api.letsencrypt.org/directory

```

The `hosts:` can be different to the domain used in LDAP

`default_db: sql` ensures that modules using storage use the `host_config:` sql settings where they support it. Modules used to be suffixed with `_odbc` but this is no longer the case. Now each module has a `db_type: mnesia | sql`. The `default_db` sets them all in one place.

`new_sql_schema: true` clearly there is a new schema, let's use it. But be warned you need to import/execute the sql `pg.new.sql` to build it.

Added ACME support by adding the `.well-known` path to the service on port 5280 - to make this work you must forward port 80 to 5280 as Let's Encrypt does not support ACME on any port other than 80.

For ACME to work you must remove the `certificate:` and `ca_file:` stanzas completely. It works it out for itself and grabs certificates with all the SAN's it requires. Then I modified the `acme:` stanza to use the new v2 urls.

SAN's attached to certificate:

```text
DNS Name: conference.${HOST}.${DOMAIN}
DNS Name: proxy.${HOST}.${DOMAIN}
DNS Name: pubsub.${HOST}.${DOMAIN}
DNS Name: upload.${HOST}.${DOMAIN}
DNS Name: vjud.${DOMAIN}
DNS Name: ${HOST}.${DOMAIN}
```

<s>There's also an odd DNS name related to vcards I think - `vjud.${HOST}.${DOMAIN}`, but it doesn't form part of the certificate.</s>

#### Important ACME Details

If you don't use a fixed spool directory and erlang node each time the ejabberd service is started it creates a new folder under `~/database` in the format of the randomly generated node name `ejabberd@960a4b448440`. The node name changes because the docker container gets a new hostname. New hostname equals new node name and new node name equals new mnesia database owner!

> It's important that you set a static node name for the ejabberd host, unless you are planning on using swarm. It's equally important to set a static spool directory.

The result of a new node and new spool dir is that ACME fires of a new request and challenge response every time the container starts. If you're building this then you'll run into the Let's Encrypt rate limit of 50 responses per week.

The solution is to set a fixed `hostname:` in the `docker-compose.yml` for `ejabberd`, eg.

```yaml
services:
  ejabberd:
    hostname: ${HOST?REQUIRED}
```

By using the `$HOST` variable we ensure the host uses the same name for both the spool dir and the node name.

<s>To fix this I set `ERLANG_NODE` and `SPOOL_DIR` as environment variables and also added them to the `command:` stanza for belt and braces.</s>

### Configuration Templates

To build the `ejabberd.yml` file and other templates I have switched to using Jinja2. As part of the `./deploy` script it renders the config files from `ejabberd.template.yml` substituting the contained environment variables with the value.

### Deploy Script

The `./deploy` is a bash script that renders the `ejabberd.yml`, `nginx.conf` and keycloak templates from the `.env`. After the templates it also restarts ejabberd if it was running, deploys the `nginx.conf` config to `/etc/nginx/conf.d/${SERIAL}.conf` and reloads Nginx. Then runs a series of API calls to Keycloak to deploy the realm and LDAP configuration.

## PostgreSQL

At initial db startup The `init-db.sh` __should__ take care of creating the database and assigning permissions.

As above if you're going to use a database for Storage you need to execute the relevant sql. This is how it's done with docker:

```shell
cat /srv/container-volumes/S00381/ejabberd/database/pg.new.sql |  docker exec -i ejabberd_db_1 psql -U ejabberd -d ejabberd
```

The `-U ejabberd` is important here or starting ejabberd fails with lots of permission errors.

## Keycloak

This was added to allow users to register for accounts and be added into the LDAP schema automatically. It brings self service to the external users.

Self Service URL: [http://SERVER:PORT/auth/realms/REALM/account](#)

Configuration templating is done using the python jinja2 module.

## Movim

It's important to add in the headers required to any front end Nginx proxy you are using:

```nginx
    location / {
        proxy_pass http://movim;
        # force timeouts if the backend dies
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;

        # set headers
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Host $remote_addr;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Server-Select $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Url-Scheme: $scheme;
        proxy_set_header Host $host;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Upgrade $http_upgrade;
        proxy_http_version 1.1;

        # by default, do not forward anything
        proxy_redirect off;
    }
```    