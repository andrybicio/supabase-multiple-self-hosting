# Supabase Docker

This is a minimal Docker Compose setup for self-hosting several instances of Supabase.


# How to use it
In order to host it in a public server and in domain `mydomain.info`, clone the repo. Note that here we assume that mydomain.info is a domain that you own and that you have access to its DNS settings. As well as using that URL, you will get access both to the API and to the dashboard, as it will be the entrypoint to communicate with the API Gateway `kong`. Indeed it's the latter that will be responsible to wire up the requests to the correct service.

In addition to supabase docs and suggested steps, here are some steps to follow.

Change the environment variables in the .env:
- Change the `PROJECT_NAME` to the name of your project
- Change the `NGINX_VIRTUAL_HOST` and `LETSENCRYPT_VIRTUAL_HOST` to your domain
- Change the `DEFAULT_EMAIL` to your email
- Change the `CLERK_CLIENT_ID`, `CLERK_CLIENT_SECRET` and `CLERK_ISSUER` to your Clerk credentials
- Change the `SESSION_SECRET` to a random hex string of at least 32 characters
- Change the `SUPABASE_PUBLIC_URL` and `API_EXTERNAL_URL` to the URL you want to access

The command for spinning up the services will be:
```
docker compose -f docker-compose.yml -f docker-compose.override.yml up -d
```

## Some notes

Note that: 
- It makes use of Clerk for authentication of the dashboard
    - The image used for kong is `revomatico/docker-kong-oidc:2.8.1-1` which has already the oidc plugin installed. Else it would have been necessary to build the image from scratch which isn't that simple for we are using the CE edition.
    - If one wants to use the basic auth with the alert, simply delete/comment the oidc plugin in the `kong.yml` file and decomment the basic-auth plugin. In this case you can sticl to the default `kong:2.8.1` image.
- It makes use of nginx reverse proxy for redirecting requests to KONG, which in turn will redirect to the correct service
    - A basic example of the configuration has to be found in `nginx/` folder
    - Note that 
- No ports of the supabase-related services are exposed to the localhost. 
    - If you want to use them, simply change the ports in the docker-compose.yml file and make them unique among services to avoid port conflicts

- **I HAVE NOT TOUCHED ANYTHING in the dev folder and docker-compose.s3.yml file.**


## Some other comments for further development


### 🧩 Problem Overview
-----
Docker Compose uses internal DNS based on service names, not container names.
For example, the service name might be 'kong', while the actual container is 
called 'supabase-kong'. Services will resolve using 'http://kong:8000', not
the container name.

This can lead to conflicts when running multiple Supabase instances on the
same host **and in the same network**, due to duplicated service names.


-----
### ✅ Why It Usually Works

Normally, Supabase instances are placed in **separate Docker networks**, which
avoids service name collisions. For example:

  - 'studio' in 'network1'
  - 'studio' in 'network2'

These can safely coexist because Docker resolves service names within isolated
network scopes.


-----
### ⚠️ Special Case: nginx-proxy
Shared services like 'nginx-proxy' must be connected to **both networks** if they
need to interact with multiple Supabase instances. Otherwise, reverse proxy redirects
 will fail.

A solution is to define network aliases in `docker-compose.override.yml`, e.g.:
```
   networks:
     default:
       aliases:
         - kong-myproject
```
Then, instead of redirecting to 'http://kong:8000', use:
```
   http://kong-myproject:8000
```
This makes routing more explicit and avoids conflicts.


-----
### 🛠️ Fixing Hard-Coded Names in vector.yml

The file '/volumes/logs/vector.yml' currently uses hard-coded container names,
which prevents reuse across multiple instances as Docker engine complains when having multiple containers with the same name.

The fix proposed is to define a variable `${PROJECT_PREFIX}` and apply it
in `docker-compose.override.yml` to make service/container names dynamic:
```
   services:
     vector:
       container_name: ${PROJECT_PREFIX}-vector
```
This allows configuration reuse and consistent naming across Supabase deployments.

