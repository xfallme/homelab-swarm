# Caddy as reverse proxy

This time around, I chose Caddy as my reverse proxy. More specifically [caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy).
To include more plugins, I use [GitHub Actions](../.github/workflows/caddy-image.yaml) to build a [Dockerfile](../deploy/reverse_proxy/caddy/Dockerfile) into an image that includes the plugins I want in Caddy.

## Connect Caddy with Crowdsec

Use the following command to get the Crowdsec API key for Caddy. For more details see [the documentation](https://app.crowdsec.net/hub/author/hslatman/remediation-components/caddy-crowdsec-bouncer).

    cscli bouncers add caddy-bouncer
