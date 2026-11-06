ARG NGINX_VERSION=1-alpine-slim

FROM docker.io/library/nginx:${NGINX_VERSION}

# Redirect /{id} to the named-query describe endpoint.
# $request_uri is used (not $uri) so the id keeps its original percent-encoding.
COPY <<'CONF' /etc/nginx/conf.d/default.conf
server {
    listen 4040;

    location / {
        if ($request_uri ~ ^/([^?]+)) {
            return 303 http://localhost:8080/describe?id=$1;
        }
        return 404;
    }
}
CONF

EXPOSE 4040
