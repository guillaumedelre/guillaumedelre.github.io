---
layout: post
title: "Local HTTPS with Traefik: traefik.me is dead, long live sslip.io"
date: 2025-04-17
categories: [devops]
tags: [docker, traefik, mkcert, tls, devops]
description: "traefik.me's wildcard cert was revoked in 2025. Here's how to replace it with sslip.io, mkcert, and a local Traefik setup."
---

The setup seemed perfect. Point `*.traefik.me` at 127.0.0.1, download a wildcard certificate from the same domain, drop it into Traefik, and every local service gets a clean HTTPS URL with no IP in the address bar. No Let's Encrypt rate limits, no `mkcert` to explain to teammates, no self-signed warnings to click through. Just `https://myapp.traefik.me` and a green padlock.

Then in March 2025, Let's Encrypt revoked the certificate. The wildcard cert for traefik.me is gone and it's not coming back.

## :label: What traefik.me was actually selling

traefik.me is a wildcard DNS resolver. Type `anything.traefik.me` and it resolves to 127.0.0.1. Type `anything.10.0.0.1.traefik.me` and it resolves to 10.0.0.1. No account, no configuration, no infrastructure to maintain. The DNS part still works fine, by the way.

The certificate was the bonus: a wildcard cert for `*.traefik.me` that pyrou, the maintainer, generated with Let's Encrypt and distributed at `https://traefik.me/cert.pem` and `https://traefik.me/privkey.pem`. It was convenient precisely because it was shared: download, drop into Traefik, done.

Sharing a private key is why it died.

The CA/Browser Forum Baseline Requirements, section 9.6.3, require subscribers to "maintain sole control" over their private key. Distributing it to anyone who visits a URL is the exact opposite of sole control. Let's Encrypt sent a notice, blocked future issuance for the domain, and revoked the existing certificate. Pyrou confirmed the situation and recommended mkcert as an alternative. The project will live on as a DNS resolver only.

The cert had already been revoked twice before 2025. Third time was the last.

## :twisted_rightwards_arrows: sslip.io does the same thing, differently

sslip.io is also a wildcard DNS resolver, with one difference: the IP is encoded in the hostname rather than resolved from a fallback. `10-0-0-1.sslip.io` resolves to `10.0.0.1`. `myapp.192-168-1-10.sslip.io` resolves to `192.168.1.10`. IPv6 works too.

The infrastructure behind sslip.io is also more visible: three nameservers in Singapore, the US, and Poland, handling over 10,000 requests per second, with public monitoring. About 1,000 GitHub stars and active maintenance under the Apache 2.0 licence.

Strip away the certificate story and the comparison is pretty straightforward:

| | traefik.me | sslip.io |
|---|---|---|
| DNS wildcard | yes | yes |
| Fallback to 127.0.0.1 | yes | no |
| IPv6 | no | yes |
| Wildcard certificate | ~~yes~~ revoked | no |
| Infrastructure | opaque | documented |
| Project activity | stalled | active |

traefik.me's only remaining advantage is the 127.0.0.1 fallback: URLs without an IP segment. That matters if you really want `myapp.traefik.me` instead of `myapp.127-0-0-1.sslip.io`. Whether that difference is worth the infrastructure uncertainty is a short conversation.

## :key: mkcert fills the gap

mkcert creates a local certificate authority, installs it in the system trust store and whatever browsers it finds, then issues certificates signed by that CA. Browsers see a trusted chain. No warning, no click-through, no "proceed anyway".

```bash
mkcert -install
```

That's the one-time setup. After that, generating a certificate is one command:

```bash
mkcert "*.127-0-0-1.sslip.io"
# produces _wildcard.127-0-0-1.sslip.io.pem
#          _wildcard.127-0-0-1.sslip.io-key.pem
```

The limitation is that mkcert's CA is local. Other machines on the network won't trust it by default. For a solo dev setup that's fine. For a shared team environment, you'd need to distribute the CA root, which is essentially the same operational problem traefik.me was trying to avoid, just smaller in scope.

## :whale: The Traefik configuration

The setup is the same regardless of which DNS service you pick. Traefik needs the certificate mounted as a volume and a static file provider pointing at a TLS configuration file.

```yaml
# traefik/config/tls.yml
tls:
  certificates:
    - certFile: /certs/cert.pem
      keyFile: /certs/key.pem
  stores:
    default:
      defaultCertificate:
        certFile: /certs/cert.pem
        keyFile: /certs/key.pem
```

The key practice: run Traefik in its own Compose project, separate from the services it routes to. Each service project connects to Traefik through a shared external network. Start and stop services independently without touching the reverse proxy.

Start by creating the external network once:

```bash
docker network create traefik-public
```

**`traefik/compose.yml`** - Traefik alone, owning the network:

```yaml
services:
  traefik:
    image: traefik:v3
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./config:/etc/traefik/config
      - ./certs:/certs
    command:
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --providers.docker=true
      - --providers.docker.network=traefik-public
      - --providers.file.directory=/etc/traefik/config
    networks:
      - traefik-public

networks:
  traefik-public:
    external: true
```

Copy the mkcert output into `./certs/`, rename to `cert.pem` and `key.pem`, then:

```bash
docker compose -f traefik/compose.yml up -d
```

Traefik is up, listening on 80 and 443, watching Docker for new containers. Nothing is routed yet.

**`whoami/compose.yml`** - a service that joins the same network:

```yaml
services:
  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.127-0-0-1.sslip.io`)"
      - "traefik.http.routers.whoami.tls=true"
      - "traefik.http.routers.whoami.entrypoints=websecure"
    networks:
      - traefik-public

networks:
  traefik-public:
    external: true
```

```bash
docker compose -f whoami/compose.yml up -d
```

Traefik detects the new container via the Docker provider, reads its labels, and adds the route. `https://whoami.127-0-0-1.sslip.io` responds immediately. Bring `whoami` down and the route disappears. Traefik keeps running without noticing.

The `external: true` declaration is the load-bearing line. Without it, Compose creates a project-scoped network: Traefik and `whoami` end up on different networks and can't reach each other, even though both are running. The external network is the shared bus every service project must explicitly opt into.

If you prefer traefik.me URLs, replace the mkcert command and the host label:

```bash
mkcert "*.traefik.me"
```

```yaml
- "traefik.http.routers.whoami.rule=Host(`whoami.traefik.me`)"
```

The DNS fallback to 127.0.0.1 handles the rest.

## :bulb: What the traefik.me story actually teaches

The certificate distribution model was always fragile. A "public-private key pair" is a contradiction in terms. Every revocation was a warning that the next one could be permanent. Eventually it was.

The lesson isn't specific to traefik.me. Any service that provides convenience by quietly removing a security boundary will eventually hit that boundary. mkcert is the right tool for this problem because it operates entirely within your own trust domain: you generate the CA, you install it, you issue the certificates. Nothing depends on a third party's continued willingness to bend certificate issuance rules.

sslip.io solves the DNS part cleanly. mkcert solves the TLS part cleanly. They compose well. The traefik.me setup was simpler, for a while. Until it wasn't.
