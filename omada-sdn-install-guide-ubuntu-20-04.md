# Installing the TP-Link Omada SDN Controller (v6.3.0.45) in a Proxmox LXC Container

Tested and working — November 17, 2026. Written up because the video guide I followed
assumed Debian 12, and this container runs Ubuntu 20.04. Every error below actually
happened during this install, with the exact fix that resolved it.

## Environment

| Item | Detail |
|---|---|
| Host | Proxmox VE 8.4.11, node `proxmox` |
| Container | LXC CT 104, named `OmadaSDN` |
| Container OS | **Ubuntu 20.04 LTS (Focal Fossa)** — not Debian 12 |
| Controller | Omada Network Application v6.3.0.45 (`Omada_Network_Application_v6.3.0.45_linux_x64_20260903171910.deb`, ~307 MB) |
| Controller dependencies | MongoDB 7.0 (server < 8.1.0) and Java 17 |

## Step-by-step install

### 1. Download the controller

Download the Linux x64 `.deb` (v6.3.0.45) from TP-Link's official Omada Software
Controller download page and place it in the container (e.g. `/root`).

### 2. Install prerequisites

```bash
apt update && apt install -y gnupg curl gdebi
```

(`gdebi` resolves the `.deb`'s dependencies for you; install it if it is not present.)

### 3. Add MongoDB's official repository — Ubuntu 20.04 version

MongoDB is **not** in Ubuntu's (or Debian's) own repositories anymore, so the Omada
installer cannot satisfy its MongoDB dependency without this step. Use the repo that
matches your actual OS — this is where the video guide goes wrong (see below).

```bash
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/7.0 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-7.0.list
apt update
```

Two things to get right here:

- **Distribution:** `focal` (Ubuntu 20.04). The Debian 12 (`bookworm`) repo line
  pulls packages built against newer libraries and will not install on 20.04.
- **Component:** `multiverse`, **not** `main`. MongoDB's Ubuntu repository uses the
  `multiverse` component — verified against MongoDB's own `InRelease` file.
  With `main`, apt fetches the list and then reports that the repository
  "doesn't have the component 'main'" and "Unable to locate package mongodb-org-server".

(If your container runs Debian 12 instead, use the `bookworm` repo line with the
`main` component — but confirm your OS first with `cat /etc/os-release`.)

### 4. Remove Ubuntu's old MongoDB 3.6 first

Ubuntu 20.04 ships MongoDB 3.6, and this container already had it installed
(`mongodb`, `mongodb-clients`, `mongodb-server`, `mongodb-server-core`).
The 7.0 install will fail with a dpkg overwrite error unless the old one is fully gone:

```bash
apt remove -y mongodb-server-core
```

Check nothing old remains:

```bash
dpkg -l | grep -i mongo
```

### 5. Install MongoDB 7.0 and start it

```bash
apt install -y mongodb-org-server
systemctl enable --now mongod
systemctl status mongod
```

Installed version: **7.0.48**. Note that MongoDB does **not** start on its own after
install — the service shows as loaded but `inactive (dead)` until you start it, and
the Omada installer needs the database running. `enable --now` both starts it and
makes it start on boot.

### 6. Install Java 17 — and make it the default Java

Omada 6.3 requires Java 17:

```bash
apt install -y openjdk-17-jre-headless
```

On Ubuntu 20.04, Java 17 lives in the **`universe`** repository (`main` only has
Java 11). If apt reports "Unable to locate package openjdk-17-jre-headless",
enable universe first:

```bash
apt install -y software-properties-common
add-apt-repository -y universe
apt update
apt install -y openjdk-17-jre-headless
```

Installed version: **17.0.15+6~us1-0ubuntu1~20.04**.

**This is the sneaky one:** installing the Java 17 package is not enough. This
container also had Java 8 installed, and Java 8 was still the default `java`
(`java -version` showed `1.8.0_452`). Omada's installer checks the Java *version*,
so it kept failing even with the Java 17 package present. Switch the default:

```bash
update-alternatives --set java /usr/lib/jvm/java-17-openjdk-amd64/bin/java
java -version
```

`java -version` must show **17** before you continue. (Do not remove Java 8 —
just change the default.)

### 7. Install the Omada controller

```bash
gdebi Omada_Network_Application_v6.3.0.45_linux_x64_20260903171910.deb
```

Answer `y` when asked. The install should now complete without errors.

### 8. Verify and reach the controller

```bash
systemctl status mongod
systemctl status omadac
```

Then open the web UI from your LAN:

```
https://<container-IP>:8043
```

Find the container's IP with `hostname -I` inside the container, or in the
Proxmox UI under the container's Network tab. Create your admin account on first
login, then adopt your devices.

## Errors encountered and their fixes

| # | Exact error | Cause | Fix |
|---|---|---|---|
| 1 | `Dependency is not satisfiable: mongodb-server (< 8.1.0)\|mongodb-org-server (< 8.1.0)\|mongodb-10gen (< 8.1.0)` | MongoDB is no longer in Ubuntu/Debian repos | Add MongoDB's official 7.0 repo (step 3) |
| 2 | `mongodb-org-server : Depends: libc6 (>= 2.34) ... / Depends: libssl3 (>= 3.0.0) but it is not installable` | Used the Debian 12 (`bookworm`) repo line on an Ubuntu 20.04 container | Use the `focal` repo line instead |
| 3 | `Skipping acquire of configured file 'main/binary-amd64/Packages' ... doesn't have the component 'main'` / `E: Unable to locate package mongodb-org-server` | MongoDB's Ubuntu repo uses the `multiverse` component, not `main` | End the repo line with `multiverse` |
| 4 | `dpkg: error ... trying to overwrite '/usr/bin/mongod', which is also in package mongodb-server-core 1:3.6.9+really3.6.8+90~g8e540c0b6d-0ubuntu5.3` | Old Ubuntu MongoDB 3.6 still installed | `apt remove -y mongodb-server-core` before installing 7.0 |
| 5 | `Error: openjdk-17-jre-headless required. Please install openjdk-17-jre-headless before installing this package.` (from the Omada pre-install script) | Java 8 was the default `java` even though the Java 17 package was installed | `update-alternatives --set java /usr/lib/jvm/java-17-openjdk-amd64/bin/java`, then confirm `java -version` shows 17 |
| 6 | `E: The package omadac needs to be reinstalled, but I can't find an archive for it.` (on *every* apt command after a failed install) | A failed install leaves `omadac` half-registered in dpkg | `dpkg --remove --force-remove-reinstreq omadac`, then continue |

## Corrections to the video guide

If you are following a guide that installs Omada in a Proxmox LXC on Debian 12,
check your container's OS first (`cat /etc/os-release`). If it is Ubuntu 20.04:

1. Do **not** use the Debian `bookworm` MongoDB repo line — use the `focal` one.
2. The MongoDB repo component for Ubuntu is **`multiverse`**, not `main`.
3. Remove Ubuntu's pre-installed MongoDB 3.6 completely (including
   `mongodb-server-core`) before installing MongoDB 7.0.
4. Start MongoDB yourself after install — `systemctl enable --now mongod`.
5. Java 17 is in `universe` on 20.04, and installing it is not enough if Java 8
   is the default — switch the default with `update-alternatives` and verify
   with `java -version`.
6. If an install attempt fails halfway, clear the broken package state with
   `dpkg --remove --force-remove-reinstreq omadac` before running apt again.

## Short version (for a YouTube comment)

> Great guide, but a few corrections for anyone whose LXC is Ubuntu 20.04 instead
> of Debian 12: (1) use the MongoDB 7.0 repo for `focal`, not `bookworm` — the
> bookworm packages need libc6 >= 2.34 / libssl3, which 20.04 doesn't have.
> (2) The MongoDB Ubuntu repo component is `multiverse`, not `main` — with `main`,
> apt can't find the package. (3) Remove Ubuntu's old MongoDB 3.6 completely
> (`apt remove -y mongodb-server-core`) or the 7.0 install fails on
> `/usr/bin/mongod`. (4) Start MongoDB yourself: `systemctl enable --now mongod`.
> (5) Omada 6.3 needs Java 17 (in `universe` on 20.04), and if Java 8 is also
> installed you must switch the default with `update-alternatives --set java
> /usr/lib/jvm/java-17-openjdk-amd64/bin/java` — the installer checks the Java
> version, not just the package. (6) If an install fails halfway, clear it with
> `dpkg --remove --force-remove-reinstreq omadac` or every apt command breaks.
> Full write-up with every error and fix is on my GitHub lab.

## What's next

- Adopt the EAP650 access point and the ES205G switch into the controller.
  The controller sits on a different VLAN from the devices, so adopt by IP
  (inform URL), not L2 discovery.
- The ER605 router can also be adopted, but adopting it migrates its VLAN and
  access-control configuration to controller management — a separate decision,
  not automatic.
