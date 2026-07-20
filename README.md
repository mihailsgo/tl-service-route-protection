# PadSign: protecting the archive and container API routes

**Goal:** put `/archive/api/` and `/container/api/` (including the PDF download) behind authentication, so only in-stack traffic and callers with a valid `Authorization` header reach them. PadSign keeps working unchanged.

**Requires** `mihailsgordijenko/ps-client:8.38` or newer (assumed already deployed).

All changes are in the `nginx/` folder next to `docker-compose.yml`.

## Instructions

### 1. Credentials file `nginx/certs/htpasswd`

`nginx/certs` is already mounted at `/etc/nginx/certs`. Generate the file (bcrypt, one line per API user):

```bash
docker run --rm httpd:2.4 htpasswd -nbB api-user 'CHANGE-ME' > nginx/certs/htpasswd
```

### 2. `nginx/nginx.conf`

Find the Docker subnet, use it as `<DOCKER_SUBNET>` below:

```bash
docker network inspect <folder>_default --format '{{(index .IPAM.Config 0).Subnet}}'
```

Replace the `/archive/api/` and `/container/api/` locations with these four blocks (keep the `proxy_pass` upstreams matching your existing file). The download route accepts either credential: Basic auth (external systems) or a Keycloak Bearer token (the pad browser) - `satisfy any` grants access if any one check passes.

```nginx
    # ps-server calls all three locations internally (covered by "allow" below).
    # this one is also called directly by the pad browser (Bearer token), which
    # never talks to the other two - so it needs auth_request on top of auth_basic
    location ~ ^/archive/api/(document/[^/]+/download)$ {
        satisfy any;                        # pass if ANY check below succeeds
        allow <DOCKER_SUBNET>;               # ps-server's own downloads: no credential needed
        deny all;                            # everyone else must pass one of the checks below
        auth_basic "PadSign download API";   # check 1: external systems (Authorization: Basic)
        auth_basic_user_file /etc/nginx/certs/htpasswd;
        auth_request /_kc_token;             # check 2: pad browser (Authorization: Bearer <keycloak-token>)

        proxy_pass http://host.docker.internal:86/api/$1$is_args$args;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }

    location = /_kc_token {
        internal;                                    # only reachable via auth_request, not directly
        proxy_pass http://keycloak:8080/auth/realms/padsign/protocol/openid-connect/userinfo;
        proxy_pass_request_body off;                 # this is a permission check, no body to forward
        proxy_set_header Content-Length "";
        # Host + X-Forwarded-* are mandatory, without them Keycloak resolves the wrong
        # issuer and rejects every valid token
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header Authorization $http_authorization;  # forward the caller's token to Keycloak
    }

    location /archive/api/ {
        satisfy any;                          # pass if ANY check below succeeds
        allow <DOCKER_SUBNET>;                 # ps-server's own calls: no credential needed
        deny all;                              # outside callers must send valid Basic auth
        auth_basic "PadSign archive API";
        auth_basic_user_file /etc/nginx/certs/htpasswd;

        proxy_pass http://host.docker.internal:86/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }

    location /container/api/ {
        satisfy any;                          # pass if ANY check below succeeds
        allow <DOCKER_SUBNET>;                 # ps-server's own calls: no credential needed
        deny all;                              # outside callers must send valid Basic auth
        auth_basic "PadSign container API";
        auth_basic_user_file /etc/nginx/certs/htpasswd;

        proxy_pass http://host.docker.internal:84/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
```

### 3. Apply

```bash
docker compose exec nginx nginx -t && docker compose restart nginx
```

Use `restart`, not `nginx -s reload` (reload can keep old workers on the previous config).

### 4. Close the direct ports

The compose file publishes services straight to host ports: 84, 86, 93, 3001, 8080. Block external access to them in your firewall; only 80/443 need to be reachable.

## End-to-end test

### 1. Archive / container routes (Basic auth)

```bash
HOST=https://padsign.trustlynx.com

curl -sk -o /dev/null -w "no auth:   %{http_code}\n" $HOST/archive/api/document/create
curl -sk -o /dev/null -w "bad auth:  %{http_code}\n" -u api-user:wrong-pass $HOST/archive/api/document/create
curl -sk -o /dev/null -w "good auth: %{http_code}\n" -u api-user:CHANGE-ME $HOST/archive/api/document/create
```

Expected: `401`, `401`, anything other than `401` (this is a bare GET on a POST-only path, so a real call returns `405` - that's nginx letting it through, not a failure). Repeat against `/container/api/...` the same way.

### 2. Download route (Basic auth or Keycloak token)

```bash
DOCID=ac89c36a-7995-483a-b3df-f8303b32d121

curl -sk -o /dev/null -w "no creds:   %{http_code}\n" $HOST/archive/api/document/$DOCID/download
curl -sk -o /dev/null -w "bad auth:   %{http_code}\n" -u api-user:wrong-pass $HOST/archive/api/document/$DOCID/download
curl -sk -o /dev/null -w "good auth:  %{http_code}\n" -u api-user:CHANGE-ME $HOST/archive/api/document/$DOCID/download
```

Expected: `401`, `401`, `200` - this is what a 3rd-party system uses (same Basic auth credential as the other two routes).

The pad browser instead sends a Keycloak Bearer token (no password anywhere - it reuses the token the logged-in session already holds):

```js
// DevTools (F12) -> Console, on the pad
copy(window.keycloak.token)
```

```bash
TOKEN=<paste-token>
curl -sk -o /dev/null -w "good token: %{http_code}\n" -H "Authorization: Bearer $TOKEN" $HOST/archive/api/document/$DOCID/download
```

Expected: `200`. (A `client_credentials` service token does **not** pass this check.)
