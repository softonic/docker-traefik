# softonic/traefik

Extension of the official image that provides a self-healing mechanism (Docker 1.12+ required).

## Usage

You don't need to do anything special, just launch this image like the original traefik image.

We have observed that from time to time a traefik task gets hung and isn't able to route the traffic to any configured backend.
Instead of manually killing those tasks to allow the orchestrator to start another clean task we have added a simple HEALTHCHECK rule.

```
HEALTHCHECK --interval=10s --timeout=2s --retries=3 \
  curl -I -k --connect-timeout 2 --resolve foo-bar.com:443:127.0.0.1 https://foo-bar.com/ || exit 1
```

### Example usage in Swarm Mode
 
You can use Traefik in a Docker Swarm cluster to use it as a reverse-proxy that routes your published services.
A valid configuration that would provide TLS support via LetsEncrypt would be:

```
export CLUSTER_DOMAIN=mydomain.com
export ADMIN_EMAIL=admin@mydomain.com

# Network to attach the services that you want to route using traefik
docker \
    network create --driver overlay proxy

# This allows you to launch traefik instance in worker nodes
docker \
    service create --name docker-proxy \
    --network proxy \
    --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock,readonly \
    --constraint 'node.role==manager' \
    rancher/socat-docker

# Exposed on ports with public access: 80, 443. Dashboard in private port: 8080
docker \
    service create --name proxy \
    --publish 80:80 --publish 443:443 --publish 8080:8080 \
    --network proxy \
    softonic/traefik:latest \
    --docker \
    --docker.swarmmode \
    --docker.endpoint=tcp://docker-proxy:2375 \
    --docker.domain="${CLUSTER_DOMAIN}" \
    --docker.watch \
    --entryPoints='Name:https Address::443 TLS' \
    --entryPoints='Name:http Address::80' \
    --acme.entrypoint=https \
    --acme=true \
    --acme.domains="${CLUSTER_DOMAIN}" \
    --acme.email="${ADMIN_EMAIL}" \
    --acme.ondemand=true \
    --acme.onhostrule=true \
    --acme.storage=/etc/traefik/acme/acme.json \
    --web
```

If you want to publish a service you just need to use the labels like:

```
docker \
  service create --name nginx-example \
    --network proxy \
    --label traefik.docker.network=proxy \
    --label traefik.backend=nginx-example \
    --label traefik.frontend.entryPoints=https,http \
    --label traefik.frontend.rule=Host:nginx-example.${CLUSTER_DOMAIN} \
    --label traefik.port=80 \
  nginx:latest
```

This allows you to access to the service in the 443 port via a runtime generated certificate by Traefik (using letsencrypt), and the same service in the 80 port.

> As Letsencrypt needs to check the domain to generate the certificate (`nginx-example.mydomain.com` in the example) you need to be sure that the 443 port is accessible via Internet.
