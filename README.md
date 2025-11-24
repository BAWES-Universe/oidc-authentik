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

