# Flowdrome — support and releases

Flowdrome is a workflow automation platform — in the family of n8n, Zapier and Power Automate —
where your workflows run on your own machines instead of somebody else's queue. You build a flow
visually in **Studio**, test it against real services, and deploy the exact document you tested to a
**host** you control, where its triggers serve continuously.

- **Documentation** — <https://flowdrome.com/docs/>
- **Downloads** — [Releases](../../releases)
- **Questions and bug reports** — [open an issue](../../issues)

This repository is the support tracker and the download point. The product source lives elsewhere.

## Install

Every release ships four artifacts: the **Nucleus** (control plane — Studio, the workflow store,
users and credentials) and the **Host** (data plane — where deployed workflows actually run), each
as a Docker image and as a Proxmox LXC template.

You need both. Install the Nucleus first, then join one or more Hosts to it.

Ports: **Nucleus 4800**, **Host 4801**.

### Docker

```bash
# Control plane
docker load -i flowdrome-nucleus_<version>.tar.gz
docker run -d --name flowdrome-nucleus --restart unless-stopped \
  -p 4800:4800 -v nucleus-data:/data flowdrome/nucleus:<version>

# Data plane
docker load -i flowdrome-host-agent_<version>.tar.gz
docker run -d --name flowdrome-host --restart unless-stopped \
  -p 4801:4801 -v host-data:/data flowdrome/host-agent:<version>
```

Open <http://localhost:4800> and sign in with **admin / admin**. Change that password before the box
is reachable by anyone else.

To connect the host: in the Nucleus go to **Hosts → Add host**, mint a join token, and give it to the
host — either paste it into the host's own dashboard at <http://localhost:4801>, or start the host
with `-e FLOWDROME_JOIN_URL=http://<nucleus>:4800 -e FLOWDROME_JOIN_TOKEN=<token>` and it enrols
itself on boot.

### Proxmox (LXC)

```bash
scp flowdrome-nucleus-lxc_<version>_amd64.tar.gz root@pve:/var/lib/vz/template/cache/
scp flowdrome-host-lxc_<version>_amd64.tar.gz    root@pve:/var/lib/vz/template/cache/

pct create 200 local:vztmpl/flowdrome-nucleus-lxc_<version>_amd64.tar.gz \
  --hostname flowdrome-nucleus --cores 2 --memory 2048 --swap 512 --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp --unprivileged 1 --onboot 1 --start 1

pct create 201 local:vztmpl/flowdrome-host-lxc_<version>_amd64.tar.gz \
  --hostname flowdrome-host --cores 2 --memory 2048 --swap 512 --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp --unprivileged 1 --onboot 1 --start 1
```

Both run as a systemd service and start on boot. `pct exec <vmid> -- journalctl -u flowdrome -f`
follows the log.

### Verify a download

```bash
sha256sum -c SHA256SUMS.txt
```

## Versions

Releases are numbered **Mark X Mod Y**, written **Mk X Mod Y**, and the version appears in the
browser tab of every Flowdrome screen. Artifacts are tagged with the dotted equivalent:
Mk 2 Mod 1 → `2.1.0`, and a release candidate → `2.1.0-rc.1`.

A release candidate is exactly that — installable and tested end to end, but published for you to
try before it is called final. Anything tagged `-rc.N` is marked as a pre-release here.

## Upgrading

Data lives in the volume (`/data`), never in the image, so an upgrade is: pull or load the new
image, stop the old container, start the new one against the **same** volume.

```bash
docker load -i flowdrome-nucleus_<new-version>.tar.gz
docker rm -f flowdrome-nucleus
docker run -d --name flowdrome-nucleus --restart unless-stopped \
  -p 4800:4800 -v nucleus-data:/data flowdrome/nucleus:<new-version>
```

Upgrade the Nucleus first, then the hosts. Take a copy of the volume before a major move.

## Reporting a problem

[Open an issue](../../issues) and include: what you ran, what happened, and what you expected. For a
workflow that misbehaves, the run id and the node it stopped at are the two most useful things —
both are on the run in the Runs view.
