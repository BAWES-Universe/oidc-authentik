# OIDC: Authentik

## Documentation
https://docs.goauthentik.io/



### Getting started

Copy `.env.template` into `.env` with your configurations

```
docker compose pull
docker compose up -d
```

### Verifying config settings
`docker compose run --rm worker dump_config`

# Configure custom ports
By default, authentik listens internally on port 9000 for HTTP and 9443 for HTTPS. 
To use different exposed ports such as 80 and 443, you can set the following variables in `.env`

COMPOSE_PORT_HTTP=80
COMPOSE_PORT_HTTPS=

### Initial setup
Go to the following link to set up initial admin account:
http://localhost:9000/if/flow/initial-setup/