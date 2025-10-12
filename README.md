![ScreenShot](logo.png)

# Setup Matrix with docker-compose

This repository helps you run your messaging application.

You can set up all you need for the matrix in less than an hour. It will install the following applications for you.

- Synapse
- Element
- PostgreSQL
- Coturn
- Caddy

> [!NOTE]  
> There are many different ways to install Matrix and its dependencies. In this tutorial we will use Caddy as a reverse proxy and webserver.
> I assume you have nothing but Docker and ComposeV2 currently installed, so we install all neccessary tools throughout the tutorial. If you already got e. g. Caddy running, you can just adapt your Caddyfile.
> If you want to use Traefik with NGINX you can find some config examples in the [original Repository](https://github.com/kamyargerami/matrix-docker).

> [!IMPORTANT]  
> I forked the original Repository since I wanted to use my own exisiting webserver and infrastructure. But also there are some issues in the tutorial and config files which are fixed here. Keep that in mind if you mix and match some config files, e.g. for NGINX and adapt everything accordingly.


# Requirements

- Docker
- Docker Compose Plugin

# Installation
1. Add these two subdomains to your DNS:

```
matrix.example.com
synapse.matrix.example.com
web.matrix.example.com
```
> [!NOTE]
> In this tutorial we will give the whole Matrix infrastructure its own subdomain (...**matrix**.example.com).
If you want to use your root domain, remove `matrix.` from every URL you see in this tutorial accordingly.

---

2. Open the following ports respectively for your setup on your server:
```
5349           TCP      # Turnserver additional Port
49160-49200    UDP      # Turnserver UDP Range
3478           TCP/UDP  # Turnserver default Port
80             TCP      # Caddy HTTP ACME challenges
443            TCP/UDP  # Caddy default HTTPS for (Matrix) traffic
```

---

3. Clone the repository and go to the `./matrix` directory. 
> [!NOTE]
> In this tutorial, this is the place where all config files and folders live. If you want to use a different folder structure, copy and adopt the config files accordingly.

---

4. Copy `.env.example` to `.env` and change `<example.com>` to your domain and `<COMPLEX_PASSWORD>` to a strong password.

---

5. Run ``docker compose up`` and after 1 minute stop it to do the next action.

---

6. Edit `./matrix/caddy/Caddyfile` and replace every `<example.com>` with your domain.

---

7. Create and edit `./matrix/coturn/_data/turnserver.conf` to apply the below configuration:

- Replace `<LONG_SECRET_KEY>` with a secure random password.
- Replace `<example.com>` with your domain.
- Change `<YOUR_SERVER_IP>` to your server's public IP address.

```
use-auth-secret
static-auth-secret=<LONG_SECRET_KEY>
realm=synapse.matrix.<example.com>
listening-port=3478
tls-listening-port=5349
min-port=49160
max-port=49200
verbose
allow-loopback-peers
cli-password=SomePasswordForCLI
external-ip=<YOUR_SERVER_IP>
```

---

8. Replace `<example.com>` with your domain in the following command and run it
```
docker run -it --rm -v ./matrix/synapse:/data -e SYNAPSE_SERVER_NAME=<example.com> -e SYNAPSE_REPORT_STATS=no matrixdotorg/synapse:latest generate
```

---

9. Edit `./matrix/synapse/_data/homeserver.yaml` and change it as below:

- You need to replace the database config with PostgreSQL
- Replace `<COMPLEX_PASSWORD>` with a secure random password.

Don't worry about the database security, this is not going to be exposed to the internet.

```
database:
  name: psycopg2
  txn_limit: 10000
  args:
    user: synapse
    password: <COMPLEX_PASSWORD>
    database: synapse
    host: matrix-synapse_db-1
    port: 5432
    cp_min: 5
    cp_max: 10
```

- Add below configuration to the end of the file
- Change every `<example.com>` to your domain address.
- Change `<LONG_SECRET_KEY>` to the secret key that you chose before in `./matrix/coturn/_data/turnserver.conf`

```
turn_uris:
  - "turn:matrix.<example.com>:3478?transport=udp"
  - "turn:matrix.<example.com>:3478?transport=tcp"
  - "turns:matrix.<example.com>:3478?transport=udp"
  - "turns:matrix.<example.com>:3478?transport=tcp"
turn_shared_secret: "<LONG_SECRET_KEY>"
turn_user_lifetime: 86400000
turn_allow_guests: False
```
> [!NOTE]  
> If you host your Turn server somewhere else or want to use an existing one replace the **whole** domains with your respective domain pointing to your Turn server.

---

10. Edit `./matrix/element/config.json` and change the following values:
- Replace `<example.com>` with your domain
- Replace `<ISO_COUNTRY_CODE>` with your ISO 3166 alpha2 Country Code

> [!NOTE]  
> You can further customize the config and thus the functionality of your Element web client to your needs. You can find information about the config [here](https://github.com/element-hq/element-web/blob/develop/docs/config.md) and about the (beta) feature flags [here](https://github.com/element-hq/element-web/blob/develop/docs/labs.md)

> [!WARNING]
> This is my current config. There will be changes in the future from Element, especially regarding the feature flags. So I recommend reading through the config documentation and also the current lab features to see if something changed.

---

11. Run the containers with `docker compose up` and if everything goes well, stop them
   and run `docker compose up -d` to run the containers in the background.

# Testing

1. The matrix URL (`https://synapse.matrix.<example.com>`) must show the synapse default page
2. Nginx must respond to these two URLs
   - https://matrix.<example.com>/.well-known/matrix/client
   - https://matrix.<example.com>/.well-known/matrix/server
3. You can test the federation on the link below
   - https://federationtester.matrix.org/
4. You can log in to your Element client at `https://web.matrix.<example.com>`

# Add new user

Run the below command to create a user.

```
docker exec -it matrix-synapse-1 register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008
```

# Enable registration

By default, registration is disabled, and users must be added using the command line. If you want to allow
everybody to register in your matrix, you can add the below line to the end of `./matrix/synapse/_data/homeserver.yaml` file.

```
enable_registration: true
enable_registration_without_verification: true
```

Run `docker compose restart` to apply the new setting.

If you need to have email verification enabled or a captcha on registration, you can read the link below:

https://matrix-org.github.io/synapse/latest/usage/configuration/config_documentation.html#registration

# For more information, you can watch the tutorials

https://www.youtube.com/watch?v=JCsw1bbBjAM

https://matrix.org/docs/guides/understanding-synapse-hosting

https://gist.github.com/matusnovak/37109e60abe79f4b59fc9fbda10896da?permalink_comment_id=3626248#optional-turn-server-video-calls

# Todo
- [ ] Add Element Call
