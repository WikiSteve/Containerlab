# Containerlab “Zero to Ping” Setup Guide

This guide takes you from a bare VM to a working Containerlab environment with Nokia and Arista images ready to use. It focuses on infrastructure only. No routing, no BGP, just getting the lab platform stable.

## 1. Host VM prerequisites

Don’t starve your VM. If you try to run this on default tiny settings, things will crash in ways that look “mysterious” but are just resource limits.

### 1.1 RAM

Set your VM to **8 GB RAM** or more.

2 GB is not enough for:

* Linux
* Docker
* Containerlab
* Multiple NOS containers

### 1.2 Disk space

Docker is greedy in two different places:

* You download NOS images into **your home directory** (usually on `/home`).
* When you import them, Docker expands and stores them under **`/var/lib/docker` on `/`**.

If you gave most of your disk to `/home` and left `/` tiny, you can end up in this situation:

* The download to `/home` works.
* The `docker import` fails because `/` runs out of space while unpacking.

You need **several gigabytes free in both**:

* `/home` for the raw downloaded files
* `/` for Docker’s internal image storage

Quick checks:

```bash
df -h /
df -h /home
```

If either of those has only a couple of gigabytes free, fix it now.
Do not soldier on and then blame Containerlab when imports fail halfway.

Typical fix (high level):

1. Shut down the VM.
2. Add a new virtual disk in your hypervisor (for example 40 GB).
3. Boot the VM.
4. Use LVM to:

   * Create a new physical volume on the new disk.
   * Extend your existing volume group with it.
   * Extend the logical volume that backs `/`.
   * Optionally extend `/home` too if that is also tight.
   * Grow the filesystem.

You can document the exact LVM commands separately as an **“LVM Emergency Expansion”** cheat sheet.

---

## 2. Install core tools

We are not using nested virtualization. We are running **containers** on top of the Linux host (using cgroups and namespaces) instead of spinning VMs inside VMs.

You need two main tools:

* Docker (container engine)
* Containerlab (topology orchestrator)

### 2.1 Install Docker

On a Debian or Ubuntu style system:

```bash
sudo apt update
sudo apt install docker.io
```

Then verify:

```bash
docker version
sudo systemctl status docker
```

Make sure Docker is **active (running)**.

### 2.2 Install Containerlab

Use the official install script:

```bash
bash -c "$(curl -sL https://get.containerlab.dev)"
```

If you care about security, read the script before running it.

Verify:

```bash
containerlab version
```

If that prints a version string, the install worked.

---

## 3. Get the Arista cEOS image

Nokia SR Linux images can be pulled automatically from a registry. Arista cEOS usually needs a manual download and import.

### 3.1 Register on Arista’s portal

1. Go to the Arista software download page.
2. Create an account with a valid email.
3. Log in and locate the **cEOS-lab** image. You want **cEOS-lab** (container), not vEOS (VM).

### 3.2 Download the correct image

When you pick the file:

* Choose the **64 bit** build for your platform.
* Pay attention to the filename. Sometimes it ends up with odd extensions like `.tar.tar`.

If the filename looks wrong:

```bash
ls
file cEOS*.tar*
```

If `file` says it is an `XZ compressed` archive but the name is weird, fix it:

```bash
mv cEOS64-lab-4.32.0F.tar.tar cEOS64-lab-4.32.0F.tar.xz
```

Adjust names to match your actual file.

Make sure you have:

* Enough free space on `/home` for the downloaded file
* Enough free space on `/` for Docker to unpack it

### 3.3 Import cEOS into Docker

From the directory where the file lives:

```bash
docker import cEOS64-lab-4.32.0F.tar.xz ceos:4.32.0F
```

Replace `4.32.0F` with whatever version you downloaded. The part after the space (`ceos:4.32.0F`) is the **repository:tag** you will use in your Containerlab YAML.

Check:

```bash
docker images | grep -i ceos
```

You should see something like:

```text
ceos   4.32.0F   <image-id>   ...
```

---

## 4. Define the topology

Create a minimal Containerlab topology file that uses Nokia and Arista together.

### 4.1 Create a working directory

```bash
mkdir -p ~/clab/zero-to-ping
cd ~/clab/zero-to-ping
```

### 4.2 Create `clab.yaml`

Open the file:

```bash
nano clab.yaml
```

Example minimal topology:

```yaml
name: zero-to-ping

topology:
  nodes:
    nokia1:
      kind: srl
      image: ghcr.io/nokia/srlinux:latest   # or the exact tag you want
    ceos1:
      kind: ceos
      image: ceos:4.32.0F                   # must match your docker import tag
```

Two key gotchas:

* **Version trap:** Docs might show `ceos:4.32`. Your imported tag might be `ceos:4.32.0F` or `ceos:4.35.0F`. The tag in `image:` must match exactly.
* **Image names:** The Nokia image string (`ghcr.io/nokia/srlinux:latest`) must be something Docker can pull. If you use a different tag, adjust it.

Save the file.

---

## 5. Deploy and verify

### 5.1 Deploy the lab

From the same directory:

```bash
containerlab deploy --topo clab.yaml
```

You should see Containerlab:

* Pull or reuse the Nokia and Arista images
* Create two nodes (`nokia1` and `ceos1`)
* Report success at the end

If you get “image not found” errors, check:

* The output of `docker images`
* The `image:` lines in `clab.yaml`

They must match.

### 5.2 Verify with Docker

List running containers:

```bash
docker ps
```

You should see containers named something like:

* `clab-zero-to-ping-nokia1`
* `clab-zero-to-ping-ceos1`

If they are up, the infrastructure side is working.

---

## 6. Access the nodes

### 6.1 Nokia (SSH)

Find the management IP:

```bash
containerlab inspect --topo clab.yaml | grep -A4 nokia1
```

Look for an `IPv4` address in the `mgmt` section.

SSH in:

```bash
ssh admin@<NOKIA_MGMT_IP>
```

Replace `<NOKIA_MGMT_IP>` with the address you found. Use the credentials for your SR Linux demo image.

Once in, try:

```text
info
show interface brief
exit
```

If that works, Nokia is good.

### 6.2 Arista (Docker exec + CLI)

For Arista cEOS, access is via Docker plus the Arista CLI process.

Attach:

```bash
docker exec -it clab-zero-to-ping-ceos1 Cli
```

Adjust the container name if yours differs.

You should see an EOS prompt like:

```text
localhost>
```

Try:

```text
show version
show ip interface brief
enable
show running-config | section hostname
exit
```

If `Cli` is not found, drop into the shell and run it there:

```bash
docker exec -it clab-zero-to-ping-ceos1 bash
Cli
```

At this point, you have:

* Nokia accessible over SSH
* Arista accessible via Docker exec and CLI

That is enough for the “Zero to Ping” foundation.

---

## 7. Cleanup

When you are done:

```bash
containerlab destroy --topo clab.yaml
docker ps
```

The `clab-zero-to-ping-*` containers should be gone.

This `deploy → test → destroy` pattern keeps your VM clean and makes later labs reproducible. Leaving stray labs running is a good way to burn RAM and then hit week 13 problems caused by ghosts from week 10.
