## Part 0. Pre flight sanity check

Run:

```bash
uname -m
free -h
df -h /
````

If:

* `uname -m` is not `x86_64`, or
* `/` has almost no free space

then either use a fresh VM or fix storage using your storage / LVM lab.

Create a notes file:

```bash
nano clab-notes.txt
```

Record:

* Architecture from `uname -m`
* Free space on `/` from `df -h /`

Save the file.

## Part 1. Check Containerlab and Docker

### 1.1 Containerlab version

Run:

```bash
containerlab version
```

If you get a version string, you are fine.

If you get `command not found`, reinstall Containerlab the way you did in class.

Add the exact version string to `clab-notes.txt`.

### 1.2 Docker status

Check Docker:

```bash
sudo systemctl status docker
```

If it is not active, start it:

```bash
sudo systemctl start docker
```

Then:

```bash
docker ps
```

It is ok if nothing is running yet.

In `clab-notes.txt` note whether Docker was already running or you had to start it.

## Part 2. Check Nokia and Arista images

List Docker images:

```bash
docker images
```

Filter if the list is long:

```bash
docker images | grep -i srl
docker images | grep -i ceos
```

You are looking for something like:

* Nokia SR Linux image, for example: `ghcr.io/nokia/srlinux   <tag>`
* Arista cEOS image, for example: `ceos   4.35`

In `clab-notes.txt` record:

* Nokia image repository and tag
* Arista image repository and tag

If either image is missing, re import it before continuing.

## Part 3. Create the Containerlab topology

You will build a two node lab:

* `nokia1` runs SR Linux (SSH access)
* `ceos1` runs cEOS (CLI via Docker)

### 3.1 Working directory

```bash
mkdir -p ~/clab/week11-recap
cd ~/clab/week11-recap
```

### 3.2 Topology file

Create `week11-recap.clab.yaml`:

```bash
nano week11-recap.clab.yaml
```

Paste this, then edit the `image:` lines to match what you saw in Part 2:

```yaml
name: week11-recap

topology:
  nodes:
    nokia1:
      kind: srl
      image: ghcr.io/nokia/srlinux:latest    # change to your exact Nokia image:tag
    ceos1:
      kind: ceos
      image: ceos:4.35                       # change to your exact cEOS tag
```

Save and exit.

Key points:

* `kind: srl` tells Containerlab this is a Nokia SR Linux node
* `kind: ceos` tells it this is an Arista cEOS node
* `image:` must match a Docker image you already have

## Part 4. Deploy the lab

From the same directory:

```bash
cd ~/clab/week11-recap
containerlab deploy -t week11-recap.clab.yaml
```

You should see Containerlab:

* Resolving images
* Creating `nokia1` and `ceos1`
* Finishing without errors

Confirm containers:

```bash
docker ps
```

You should see containers with names like:

* `clab-week11-recap-nokia1`
* `clab-week11-recap-ceos1`

Record the exact names in `clab-notes.txt`.

## Part 5. SSH into Nokia

### 5.1 Find the management IP

Use `inspect`:

```bash
containerlab inspect -t week11-recap.clab.yaml
```

Look for the `mgmt` section for `nokia1`. You should see an IPv4 address, for example `172.20.20.5`.

Quick filter option:

```bash
containerlab inspect -t week11-recap.clab.yaml | grep -A4 nokia1
```

Record the IPv4 address of `nokia1` in `clab-notes.txt`.

### 5.2 SSH to the node

From the host VM:

```bash
ssh admin@<NOKIA_MGMT_IP>
```

Replace `<NOKIA_MGMT_IP>` with the address from 5.1.

Use the credentials configured for your SR Linux image.

Once logged in, run for example:

```text
info
show interface brief
exit
```

In `clab-notes.txt` write:

* The IP address used
* Confirmation that `info` and `show interface brief` worked

## Part 6. Enter Arista CLI using Docker

For Arista, you attach to the container and run the CLI process.

### 6.1 Attach to the container

Confirm the name:

```bash
docker ps
```

Then:

```bash
docker exec -it clab-week11-recap-ceos1 Cli
```

Adjust the container name if yours differs.

You should land at a cEOS prompt, something like:

```text
localhost>
```

Run a few commands:

```text
show version
show ip interface brief
enable
show running-config | section hostname
exit
```

If `Cli` fails, drop into the shell then run `Cli`:

```bash
docker exec -it clab-week11-recap-ceos1 bash
Cli
```

In `clab-notes.txt` record:

* The prompt you saw
* One thing that looked familiar from Cisco IOS

## Part 7. Destroy the lab

From `~/clab/week11-recap`:

```bash
containerlab destroy -t week11-recap.clab.yaml
docker ps
```

The `clab-week11-recap-*` containers should be gone.

Add a short note to `clab-notes.txt`:

* One or two sentences on why the pattern `deploy → work → destroy` is safer than leaving labs running

## Deliverables

Submit:

1. `week11-recap.clab.yaml`
2. `clab-notes.txt` containing:

   * Architecture and free space from Part 0
   * Containerlab version
   * Docker status (running or started)
   * Nokia and Arista image names and tags
   * Container names from `docker ps`
   * Nokia management IP and SSH confirmation
   * Arista CLI prompt and one familiar feature
   * Final “why we destroy labs” note

```

ex=0}
```
