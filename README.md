# Pi-hole + Unbound

Docker Compose setup for running [Pi-hole](https://pi-hole.net/) with
[Unbound](https://nlnetlabs.nl/projects/unbound/about/) as its recursive DNS
resolver.

Pi-hole handles filtering and forwards permitted queries to Unbound. Unbound
resolves them recursively and validates DNSSEC, without using a public upstream
resolver.

## Setup

Requirements:

- Docker Engine with Docker Compose
- Ports 53 (TCP/UDP), 80 (TCP), and 443 (TCP) available on the host
- A stable LAN address for the host

Clone the repository:

```bash
git clone https://github.com/Wuodan/pihole-unbound.git
cd pihole-unbound
```

Create `.env` from the example and set the Pi-hole web password:

```bash
cp .env.example .env
```

```dotenv
PIHOLE_PASSWORD='replace-with-a-long-random-password'
```

Single quotes keep characters such as `$` literal. `.env` is ignored by Git.

Change `TZ` in `compose.yaml` to your timezone, then start the stack:

```bash
docker compose up -d
docker compose ps
```

The Pi-hole admin interface is available at:

```text
http://<host-address>/admin/
```

Configure your router's DHCP settings to advertise the host's LAN address as
the DNS server. Do not configure a public secondary DNS server if all client
queries should pass through Pi-hole. Configure the equivalent IPv6 setting as
well if your network advertises DNS over IPv6.

## Verify

Run these queries on the Docker host:

```bash
dig @127.0.0.1 pi-hole.net
dig @127.0.0.1 dnssec.works
dig @127.0.0.1 dnssec-failed.org
```

The first two should return `NOERROR`. `dnssec-failed.org` should return
`SERVFAIL`, confirming that Unbound rejects invalid DNSSEC data.

For logs and Unbound configuration validation:

```bash
docker compose logs --tail=100 pihole unbound
docker compose exec unbound unbound-checkconf /etc/unbound/unbound.conf
```

## Configuration

Pi-hole uses the following environment-controlled settings from `compose.yaml`:

- `FTLCONF_dns_upstreams: unbound`
- `FTLCONF_dns_listeningMode: ALL`, required for the Docker bridge network
- `FTLCONF_dns_dnssec: "false"`, because Unbound performs DNSSEC validation

Unbound uses the [`klutchell/unbound`](https://hub.docker.com/r/klutchell/unbound)
image and is reachable only on the private Compose network. Pi-hole is the only
service that publishes DNS ports on the host.

Pi-hole data is stored in `./etc-pihole/`. The directory is created at startup
and ignored by Git.

## Update

```bash
docker compose pull
docker compose up -d
```

The setup tracks the `latest` tag of both images. Replace the Pi-hole tag with a
[date-based image tag](https://github.com/pi-hole/docker-pi-hole/releases) and
the Unbound tag with one from [Docker Hub](https://hub.docker.com/r/klutchell/unbound/tags)
if you prefer explicit version updates.

## References

- [Pi-hole Docker documentation](https://docs.pi-hole.net/docker/)
- [Pi-hole Unbound guide](https://docs.pi-hole.net/guides/dns/unbound/)
- [`klutchell/unbound` source](https://github.com/klutchell/unbound-docker)
- [Unbound documentation](https://unbound.docs.nlnetlabs.nl/en/latest/)

## License

[Apache License 2.0](LICENSE)
