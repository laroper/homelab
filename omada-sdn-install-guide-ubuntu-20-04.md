# Installing the TP-Link Omada SDN Controller (v6.3.0.45) in a Fresh Proxmox LXC Container (Ubuntu 20.04)

A clean, start-to-finish install. Tested November 17, 2026 — every step below was
verified during a real install on this exact setup, with the pitfalls already
worked out so the install succeeds the first time.

## Environment

| Item | Detail |
|---|---|
| Host | Proxmox VE 8.4.11, node `proxmox` |
| Container | LXC, Ubuntu 20.04 LTS (Focal Fossa) |
| Controller | Omada Network Application v6.3.0.45 (`Omada_Network_Application_v6.3.0.45_linux_x64_20260903171910.deb`, ~307 MB) |
| Controller needs | MongoDB 7.0 (server < 8.1.0) and Java 17 |

## 1. Create the container

In Proxmox, **Create CT** and choose:

- Template: Ubuntu 20.04 standard
- Resources: 2 CPU cores, 4 GB RAM, 16 GB disk (the controller is light; this leaves headroom for MongoDB)
- Network: DHCP, or a static address on the VLAN the controller will live on
- An unprivileged container is fine

Start the container and open its console (or SSH in as root).

## 2. Update the container and install prerequisites

```bash
apt update && apt upgrade -y
apt install -y gnupg curl gdebi software-properties-common
```

## 3. Install MongoDB 7.0

The controller needs MongoDB, but Ubuntu 20.04 no longer ships it — you must add
MongoDB's own repository. Use the repo that matches the container's OS exactly.

```bash
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/7.0 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-7.0.list
apt update
```

Two details that break this step if wrong:

- **`focal`** is the Ubuntu 20.04 release name. Do not use the Debian `bookworm`
  repo line — those packages need libc6 >= 2.34 and libssl3, which 20.04 does
  not have. Confirm your OS first with `cat /etc/os-release`.
- The component is **`multiverse`**, not `main`. With `main`, apt fetches the
  list and then says the repository "doesn't have the component 'main'" and the
  package cannot be found. (Verified against MongoDB's own `InRelease` file.)

If the container already has Ubuntu's old MongoDB 3.6 installed, remove it
first — it owns `/usr/bin/mongod` and will block the 7.0 install:

```bash
dpkg -l | grep -i mongo
apt remove -y mongodb-server-core mongodb-server mongodb-clients mongodb
```

Then install and start MongoDB:

```bash
apt install -y mongodb-org-server
systemctl enable --now mongod
systemctl status mongod
```

Installed version: **7.0.48**. MongoDB does **not** start by itself after install —
the service shows as loaded but `inactive (dead)` until you start it, and the
Omada installer needs the database running. `enable --now` both starts it and
makes it start on boot.

> CPU note: MongoDB 7.0 needs AVX support from the host CPU. If `mongod` fails
> to start with an illegal-instruction error, the host CPU is too old for
> MongoDB 7.

## 4. Install Java 17 and make it the default

Omada 6.3 requires Java 17. On Ubuntu 20.04 it lives in the `universe`
repository (`main` only has Java 11), so enable universe first:

```bash
add-apt-repository -y universe
apt update
apt install -y openjdk-17-jre-headless
java -version
```

`java -version` must show **17**. If it shows 1.8.x, Java 8 is also installed
and is still the default — the Omada installer checks the Java *version*, so it
will fail even though the Java 17 package is present. Switch the default:

```bash
update-alternatives --set java /usr/lib/jvm/java-17-openjdk-amd64/bin/java
java -version
```

Installed version: **17.0.15+6~us1-0ubuntu1~20.04**. Leave Java 8 installed; only
the default changes.

## 5. Download and install the Omada controller

Download the Linux x64 `.deb` (v6.3.0.45) from TP-Link's official Omada Software
Controller download page and place it in the container (e.g. `/root`), then:

```bash
gdebi Omada_Network_Application_v6.3.0.45_linux_x64_20260903171910.deb
```

Answer `y` when asked. With MongoDB running and Java 17 as the default, the
install completes without errors.

## 6. Verify and open the controller

```bash
systemctl status mongod
systemctl status omadac
```

Both services should show `active (running)`. Then open the web UI from your LAN:

```
https://<container-IP>:8043
```

Find the container's IP with `hostname -I` inside the container, or in the
Proxmox UI under the container's Network tab. Create your admin account on first
login.

## 7. Adopt your devices

- Adopt the access point and switch from the controller's Devices page. If the
  controller sits on a different VLAN from the devices, adopt by IP (set the
  inform URL on each device to the controller's address) — L2 discovery will not
  cross VLANs.
- The ER605 router can also be adopted, but adopting it migrates its VLAN and
  access-control configuration to controller management — treat that as a
  separate decision, not part of the install.

## If something fails

| Symptom | Fix |
|---|---|
| `Dependency is not satisfiable: mongodb-server (< 8.1.0)...` | MongoDB repo not added — do step 3 |
| `Depends: libc6 (>= 2.34)` / `libssl3 ... not installable` | Wrong repo line (Debian bookworm on Ubuntu 20.04) — use the `focal` line in step 3 |
| `doesn't have the component 'main'` / `Unable to locate package mongodb-org-server` | Repo component must be `multiverse`, not `main` |
| `trying to overwrite '/usr/bin/mongod'` | Old MongoDB 3.6 still installed — remove it (step 3) |
| `Error: openjdk-17-jre-headless required` during install | Default `java` is still 8 — switch it (step 4) and re-run gdebi |
| `E: The package omadac needs to be reinstalled, but I can't find an archive for it` (every apt command fails) | A failed install left omadac half-registered — `dpkg --remove --force-remove-reinstreq omadac`, then continue |

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
> Full clean-install walkthrough is on my GitHub lab.
