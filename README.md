# Pi-hole + Unbound

Run [Pi-hole](https://pi-hole.net/) with
[Unbound](https://github.com/klutchell/unbound-docker) as a recursive,
DNSSEC-validating resolver.

## Requirements

- Docker with Docker Compose
- Ports 53/tcp, 53/udp, 80/tcp, and 443/tcp available
- A stable LAN address for the host
- The private subnet `172.31.0.0/24` available for the Docker network

## Setup

```bash
git clone https://github.com/Wuodan/pihole-unbound.git
cd pihole-unbound
cp .env.example .env
```

Set `PIHOLE_PASSWORD` in `.env`, then run:

```bash
docker compose up -d
docker compose ps
```

To use Pi-hole network-wide, configure your router's DHCP settings to advertise
the Docker host's address as the DNS server. If the router advertises DNS over
IPv6, configure it to advertise the Docker host's IPv6 address as well.

Do not configure a public secondary DNS server. Clients may use it instead of
Pi-hole and bypass filtering.

The Pi-hole admin interface is available at `http://<docker-host>/admin/`.

## Verify

On the Docker host:

```bash
dig @127.0.0.1 pi-hole.net
dig @127.0.0.1 dnssec.works
dig @127.0.0.1 dnssec-failed.org
```

The first two queries should return `NOERROR`; `dnssec-failed.org` should return
`SERVFAIL`.

From another LAN client:

```bash
dig @<docker-host> pi-hole.net
```

Then make a normal DNS query and confirm it appears in Pi-hole's Query Log. This
verifies that the client is using the DNS server advertised by the router.

View logs:

```bash
docker compose logs --tail=100 pihole unbound
```

## Update

```bash
docker compose pull
docker compose up -d
docker image prune -f
```

### Automatic updates

Install the included weekly systemd timer:

```bash
sudo cp systemd/pihole-unbound-update@.{service,timer} /etc/systemd/system/
sudo systemctl daemon-reload
unit=$(systemd-escape --path "$PWD")
sudo systemctl enable --now "pihole-unbound-update@${unit}.timer"
```

The timer pulls both images, recreates changed containers, and waits for their
health checks.

## Documentation

- [Pi-hole®](https://github.com/pi-hole/pi-hole): [DNS sinkhole](https://en.wikipedia.org/wiki/DNS_Sinkhole) for advertisement domain names
- [Pi-hole Docker](https://github.com/pi-hole/docker-pi-hole/#docker-pi-hole): official Pi-hole Docker image
- [Pi-hole with Unbound](https://docs.pi-hole.net/guides/dns/unbound/)
- [Unbound](https://github.com/NLnetLabs/unbound#unbound): validating, recursive, and caching DNS resolver.
- [klutchell/unbound](https://github.com/klutchell/unbound-docker#unofficial-unbound-multiarch-docker-image): unofficial
  unbound multiarch docker image

## License

[Apache License 2.0](LICENSE)
