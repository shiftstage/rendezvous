# ShiftStage Rendezvous node

The node that helps the [ShiftStage](https://shiftstage.app/r/github-rendezvous) app on your
iPhone find your Mac away from home. Run your own and no request reaches us on the path
between your phone and your Mac.

This repository publishes releases of the node: a container image and static binaries.

## What a node does

On a local network no node takes part: the phone and the Mac find each other directly.
Away from home a node does two separate jobs, switched on with `--roles`:

- **Signalling.** Two devices that already trust each other learn how to reach each other.
  The node sees key fingerprints and network addresses; it cannot read anything the
  devices say.
- **Relay.** When the networks refuse a direct connection, the node forwards an opaque,
  end-to-end encrypted stream. It cannot decrypt it and keeps nothing.

A node with signalling only is a working configuration. Relay is what costs bandwidth.

Running your own node does not replace the subscription: connecting from outside your own
network needs one wherever the traffic goes. See
[shiftstage.app/self-hosting](https://shiftstage.app/self-hosting/).

## Requirements

- Any Linux server with a public address, x86-64 or arm64. The smallest tier is enough.
- Docker, or nothing at all: the binary is fully static (musl) and needs neither Swift nor
  a particular glibc. Tested on bare Ubuntu 24.04, 22.04, 20.04 and Debian 12.
- Open ports:

| Port | Role |
|---|---|
| 7443/tcp, 3478/udp, 3479/udp | signalling |
| 7444/udp, 7445/tcp | relay |
| 443/tcp, 443/udp | optional: reach from networks that allow only 443 |

## Up in two commands (Docker)

Issue the node's key. The command prints the key **fingerprint**:

```sh
docker run --rm -v "$PWD/tls:/tls" ghcr.io/shiftstage/rendezvous keygen --out /tls
```

Run the node:

```sh
docker run -d --name rendezvous --restart unless-stopped \
  -p 7443:7443 -p 3478:3478/udp -p 3479:3479/udp \
  -p 7444:7444/udp -p 7445:7445 \
  -v "$PWD/tls:/tls:ro" \
  ghcr.io/shiftstage/rendezvous \
  --tls-certificate /tls/cert.pem --tls-key /tls/key.pem \
  --advertise your.server.address --roles signaling,relay
```

Then, on your Mac, open the ShiftStage agent's settings and add the node to the list of
nodes: its address and the fingerprint from `keygen`.

No certificate authority is involved: your devices trust the node by its pinned
fingerprint, the way they trust your Mac. A new key means every device stops trusting the
node until it gets the new fingerprint, so `keygen` never overwrites existing files
without `--force`.

## Without Docker

Download the binary for your architecture from the
[latest release](https://github.com/shiftstage/rendezvous/releases/latest) and check it:

```sh
curl -fsSLO https://github.com/shiftstage/rendezvous/releases/latest/download/rendezvous-linux-amd64
curl -fsSLO https://github.com/shiftstage/rendezvous/releases/latest/download/SHA256SUMS
sha256sum -c --ignore-missing SHA256SUMS
install -m 755 rendezvous-linux-amd64 /usr/local/bin/rendezvous

rendezvous keygen --name my-node --out /etc/rendezvous
rendezvous --tls-certificate /etc/rendezvous/cert.pem --tls-key /etc/rendezvous/key.pem \
           --advertise your.server.address --roles signaling,relay
```

Use `rendezvous-linux-arm64` on arm64. The binary is extracted from the published image,
so the two cannot differ. All flags: `rendezvous --help`.

### As a systemd service

```ini
# /etc/systemd/system/rendezvous.service
[Unit]
Description=ShiftStage Rendezvous node
After=network-online.target

[Service]
ExecStart=/usr/local/bin/rendezvous \
  --tls-certificate /etc/rendezvous/cert.pem \
  --tls-key /etc/rendezvous/key.pem \
  --advertise your.server.address \
  --roles signaling,relay
Restart=always
RestartSec=2
DynamicUser=yes
LimitNOFILE=16384
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
NoNewPrivileges=yes
ReadOnlyPaths=/etc/rendezvous

[Install]
WantedBy=multi-user.target
```

`DynamicUser=yes` runs the node as a user that does not exist before start, so both PEM
files need mode `644`; `keygen` sets it.

## Check it

```sh
rendezvous --version                    # which build this is
curl -s http://127.0.0.1:9090/healthz   # a running node, started with --metrics-port 9090
```

If signalling answers but relay does not work, UDP is almost always closed: some hosting
providers filter it by default.

## Keeping it current

The signalling exchange is versioned, and an unknown field is ignored rather than
rejected: a node one version behind keeps working. The ShiftStage agent on your Mac tells
you when your node is older than it expects. Updating is pulling a newer image and
recreating the container with the same `docker run` command:

```sh
docker pull ghcr.io/shiftstage/rendezvous:latest
docker rm -f rendezvous
# then the docker run command from above
```

## Support

This repository only publishes releases. Write to
[support@shiftstage.app](mailto:support@shiftstage.app) for questions and bug reports.
