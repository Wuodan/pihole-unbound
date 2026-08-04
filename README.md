# Pi-hole + Unbound with Docker Compose

A small, self-contained Pi-hole installation that uses Unbound as a recursive,
DNSSEC-validating resolver instead of forwarding queries to a public DNS provider.
It is the configuration running on a 64-bit Raspberry Pi 3, generalized so it can
be reused on other ARM64 or x86-64 Linux hosts.

```text
LAN clients -> host :53 -> Pi-hole -> unbound:5335 -> authoritative DNS servers
                         blocks ads      validates DNSSEC
```

This is a community setup, not an official combined image. It builds Unbound from
Alpine's package and uses the official Pi-hole container. The relevant upstream
documentation is the [Pi-hole Docker guide](https://docs.pi-hole.net/docker/),
[Pi-hole's Unbound guide](https://docs.pi-hole.net/guides/dns/unbound/), and the
[Unbound documentation](https://unbound.docs.nlnetlabs.nl/en/latest/).

## What this deploys

- Pi-hole publishes DNS on TCP/UDP port 53 and its web interface on ports 80/443.
- Unbound is reachable only on the private Compose network, at `unbound:5335`.
- Pi-hole filtering data and settings persist in the local `etc-pihole/` directory.
- Unbound performs DNSSEC validation. Pi-hole's own DNSSEC option is deliberately
  off to avoid validating the same response twice.
- Unbound uses IPv4 for its outgoing recursive queries. LAN clients may still reach
  Pi-hole over IPv4 or IPv6 because Docker publishes Pi-hole's host ports.

The Compose subnet `172.31.0.0/24` must not overlap a Docker, LAN, or VPN network on
your host. Change it consistently in `compose.yaml` and `unbound/unbound.conf` if it
does.

## Prerequisites

- A Linux machine with a fixed or DHCP-reserved LAN address
- A 64-bit ARM or x86-64 installation supported by the container images
- Docker Engine with the Compose plugin (`docker compose`)
- TCP/UDP port 53 and TCP ports 80 and 443 free on the host
- Router access, so DHCP clients can be told to use the Pi-hole host as DNS

This exact setup was verified on 64-bit Raspberry Pi OS on a Raspberry Pi 3. Pi-hole
is not configured as a DHCP server here; the router remains the DHCP server.

Check for an existing DNS listener before starting:

```bash
sudo ss -lntup | grep -E ':(53|80|443)\b'
```

If another service owns one of these ports, resolve that conflict before continuing.

## Install

Clone the repository and enter it:

```bash
git clone https://github.com/Wuodan/pihole-unbound.git
cd pihole-unbound
```

Create the local environment file and set a strong web-interface password:

```bash
cp .env.example .env
nano .env
```

`.env` is ignored by Git. Docker Compose applies interpolation to unquoted and
double-quoted values; single-quote a password containing `$` so it remains literal.
See Docker's [`.env` syntax](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/#env-file-syntax).

The timezone is set to `Europe/Zurich` in `compose.yaml`. Change `TZ` there if needed.
Validate the resolved configuration, then build and start both services:

```bash
docker compose config --quiet
docker compose up -d --build
docker compose ps
```

Open `https://<PIHOLE_IP>/admin/` (or HTTP if preferred). The browser will warn about
Pi-hole's automatically generated self-signed HTTPS certificate; that is expected.

## Configure the router

Reserve a stable IPv4 address for the Docker host. In the router's LAN/DHCP settings,
advertise that address as the DNS server. Do not advertise a public resolver as a
secondary DNS server: clients are free to use it and thereby bypass Pi-hole.

IPv6 needs separate attention. If the router advertises DNS through router
advertisements or DHCPv6, point that setting to a stable IPv6 address of the Pi-hole
host as well, or clients may bypass Pi-hole over IPv6. Router interfaces differ, so
verify the DNS servers actually received by at least one client after renewing its
lease or reconnecting it.

On a Linux client, one of these usually shows the active DNS configuration:

```bash
resolvectl status
cat /etc/resolv.conf
```

The active server should be the Pi-hole host, not the router or a public resolver.

### FRITZ!Box notes

For a typical FRITZ!Box, set the Pi's reserved IPv4 address as the **Local DNS server**
under **Home Network > Network > Network Settings > IPv4 Settings**. IPv6 options and
menu labels vary by FRITZ!OS and model; use a stable ULA for the Pi when the router
allows a local IPv6 DNS server to be advertised. AVM documents the distinction
between changing the router's upstream resolver and the DNS server announced to
home-network clients in its [DNS server guide](https://en.avm.de/service/knowledge-base/dok/FRITZ-Box-7590/165_Configuring-different-DNS-servers-in-the-FRITZ-Box/).

## Verify the resolver chain

Install `dig` on Raspberry Pi OS/Debian if necessary:

```bash
sudo apt update
sudo apt install dnsutils
```

Query Pi-hole through the host-published DNS port:

```bash
dig @127.0.0.1 pi-hole.net
dig @127.0.0.1 dnssec.works
dig @127.0.0.1 dnssec-failed.org
```

The first two queries should return `status: NOERROR`. The intentionally broken
DNSSEC domain should return `status: SERVFAIL`, demonstrating that Unbound rejects an
invalid chain of trust. You can also replace `127.0.0.1` with the Pi-hole host's LAN
address to test from another machine.

Inspect service state and logs when a check fails:

```bash
docker compose ps
docker compose logs --tail=100 pihole
docker compose logs --tail=100 unbound
docker compose run --rm unbound unbound-checkconf /etc/unbound/unbound.conf
```

`SERVFAIL` for every domain commonly means Unbound cannot reach authoritative DNS
servers over outbound UDP/TCP port 53. A timeout from clients usually means a host
firewall, port conflict, incorrect address, or router configuration problem.

## Operations

Pull the current Pi-hole release, rebuild Unbound to pick up Alpine package and root
hints updates, and recreate the services:

```bash
docker compose pull pihole
docker compose build --pull unbound
docker compose up -d
docker image prune
```

`pihole/pihole:latest` follows new releases, matching the installation documented
here. For stricter reproducibility, replace `latest` with one of Pi-hole's date-based
tags and update it deliberately. Review the [Pi-hole release notes](https://github.com/pi-hole/docker-pi-hole/releases)
before upgrading.

Back up the persistent configuration before upgrades or experiments:

```bash
sudo tar -czf "pihole-backup-$(date +%F).tar.gz" etc-pihole
```

Stop the services without deleting persisted Pi-hole data:

```bash
docker compose down
```

To restore this deployment on another host, clone the repository, restore
`etc-pihole/` if desired, create a fresh `.env`, and run `docker compose up -d --build`.

## Security and design notes

- Never commit `.env`, `etc-pihole/`, database files, TLS private keys, or exported
  support conversations. The included ignore rules cover these paths.
- Do not expose DNS or the admin interface directly to the public internet. Restrict
  them to trusted LANs with the host/router firewall.
- Environment-defined Pi-hole settings are read-only in the UI. Edit `compose.yaml`
  and recreate the container to change them.
- The static container addresses make the access-control boundary explicit. Pi-hole
  uses the Compose service name `unbound`, so its upstream remains readable.
- Root hints are downloaded when the Unbound image is built. `unbound-anchor` prepares
  the DNSSEC root trust anchor when the container starts; Unbound then maintains it
  using RFC 5011 while that container exists. Its unusual success statuses are why
  the Docker command does not join `unbound-anchor` to `chown` with `&&`.

## License

Apache License 2.0; see [LICENSE](LICENSE).
