# Pi-hole + Unbound

Run [Pi-hole](https://pi-hole.net/) with
[Unbound](https://github.com/klutchell/unbound-docker) as a recursive,
DNSSEC-validating resolver.

## Requirements

- Docker with Docker Compose
- Ports 53/tcp, 53/udp, 80/tcp, and 443/tcp available
- A stable LAN address for the host

## Setup

```bash
git clone https://github.com/Wuodan/pihole-unbound.git
cd pihole-unbound
cp .env.example .env
```

Set `PIHOLE_PASSWORD` in `.env` and adjust `TZ` in `compose.yaml`, then run:

```bash
docker compose up -d
docker compose ps
```

Open `http://<host-address>/admin/` and configure your router to advertise the
host address as its DNS server. Configure its IPv6 DNS setting too, if used.

Do not add a public secondary DNS server unless you want clients to bypass
Pi-hole when they choose it.

## Verify

```bash
dig @127.0.0.1 pi-hole.net
dig @127.0.0.1 dnssec.works
dig @127.0.0.1 dnssec-failed.org
```

The first two queries should return `NOERROR`; `dnssec-failed.org` should return
`SERVFAIL`.

View logs:

```bash
docker compose logs --tail=100 pihole unbound
```

## Update

```bash
docker compose pull
docker compose up -d
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

- [Pi-hole Docker](https://docs.pi-hole.net/docker/)
- [Pi-hole with Unbound](https://docs.pi-hole.net/guides/dns/unbound/)
- [`klutchell/unbound`](https://github.com/klutchell/unbound-docker)

## License

[Apache License 2.0](LICENSE)
