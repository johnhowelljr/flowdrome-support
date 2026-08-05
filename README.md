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

**Start with the Nucleus alone.** It carries a built-in host, so one container or one CT is a
complete, working Flowdrome: you can build a workflow, deploy it and call it. Add the separate Host
artifact only when you want workloads running on *more* machines.

Ports: **Nucleus 4800**, its **built-in host 4820**, a standalone **Host 4801**.

### Docker

```bash
docker load -i flowdrome-nucleus_<version>.tar.gz
docker run -d --name flowdrome-nucleus --restart unless-stopped \
  -p 4800:4800 -p 4820:4820 -v nucleus-data:/data flowdrome/nucleus:<version>
```

Open <http://localhost:4800> and sign in with **admin / admin**. Change that password before the box
is reachable by anyone else.

**Hosts** already lists "Built-in host" — no join token, no configuration — so you can deploy right
away. Publishing `4820` above is what lets you call those deployed workflows from outside the
container; leave it off if you only ever call them from within.

To add another machine:

```bash
docker load -i flowdrome-host-agent_<version>.tar.gz
docker run -d --name flowdrome-host --restart unless-stopped \
  -p 4801:4801 -v host-data:/data flowdrome/host-agent:<version>
```

In the Nucleus go to **Hosts → Add host**, mint a join token, and give it to that host — either paste
it into the host's own dashboard at <http://its-address:4801>, or start the host with
`-e FLOWDROME_JOIN_URL=http://<nucleus>:4800 -e FLOWDROME_JOIN_TOKEN=<token>` and it enrols itself
on boot.

### Proxmox (LXC)

```bash
scp flowdrome-nucleus-lxc_<version>_amd64.tar.gz root@pve:/var/lib/vz/template/cache/

pct create 200 local:vztmpl/flowdrome-nucleus-lxc_<version>_amd64.tar.gz \
  --hostname flowdrome-nucleus --cores 2 --memory 2048 --swap 512 --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp --unprivileged 1 --onboot 1 --start 1
```

That CT is a complete install — the built-in host is reachable on the container's own address at
4820, nothing to publish. For workloads on a further machine, add a Host CT too:

```bash
scp flowdrome-host-lxc_<version>_amd64.tar.gz root@pve:/var/lib/vz/template/cache/

pct create 201 local:vztmpl/flowdrome-host-lxc_<version>_amd64.tar.gz \
  --hostname flowdrome-host --cores 2 --memory 2048 --swap 512 --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp --unprivileged 1 --onboot 1 --start 1
```

Both run as a systemd service and start on boot. `pct exec <vmid> -- journalctl -u flowdrome -f`
follows the log.

### Turning the built-in host off

`-e FLOWDROME_BUILTIN_HOST=0` if you would rather run only external hosts.
`FLOWDROME_BUILTIN_HOST_PORT` moves it off 4820; `FLOWDROME_BUILTIN_HOST_BIND` pins its interface.

### Verify a download

```bash
sha256sum -c SHA256SUMS.txt
```

## Versions

Releases are numbered **Mark X Mod Y**, written **Mk X Mod Y**, and the version appears in the
browser tab of every Flowdrome screen. Artifacts are tagged with the dotted equivalent —
Mk 2 Mod 1 → `2.1.0`.

A Mod **is** the release — there are no release candidates and no pre-release tags. If it is
published here, it is released.

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
