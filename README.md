# Kimai in podman

Container files for running [Kimai](https://www.kimai.org/) with rootless podman
under systemd. They start an empty Kimai instance with a MariaDB database.
Outgoing mail is not set up.

The directory `quadlet/` holds Quadlet files: descriptions of containers,
volumes and a network, from which podman generates systemd units. systemd then
starts the containers in order, restarts one that fails and starts them at boot.
This needs podman on a Linux machine with systemd; the files do not work with
Docker. For Docker, Kimai documents a [compose
file](https://www.kimai.org/documentation/docker-compose.html).

Two containers are started:

- `kimai` runs the image `kimai/kimai2` of the Kimai project, with Apache and
  PHP. Kimai is tested with version 2.69.
- `kimai-db` runs the MariaDB image of Docker Hub.

The data is kept in three named volumes: `kimai-db` for the database,
`kimai-data` for the files that Kimai stores, such as invoices, and
`kimai-plugins` for plugins. The containers share the network `kimai`.

## Requirements

- podman with the `podman quadlet` commands, set up for rootless use. The files
  were tested with podman 6.1.
- systemd with a user instance, which every login session has
- `openssl`, to generate the passwords

## Passwords

The database needs two passwords: one for the user `kimai`, which Kimai logs in
with, and one for the MariaDB user `root`. Each is a podman secret. Create both
once, before the first start.

```bash
openssl rand -hex 32 | podman secret create kimai-db-password -
```

```bash
openssl rand -hex 32 | podman secret create kimai-db-root-password -
```

The password of `kimai` becomes a part of the address with which Kimai connects
to the database. Use letters and digits only, as the commands above do: other
characters would have to be encoded for the address.

podman stores both passwords without encryption, in a file in its storage
directory that only your user can read, and mounts them as files in the
containers. Inside the `kimai` container the password of `kimai` is also in the
environment variable `DATABASE_URL` of the Kimai processes. `podman inspect`
does not show it.

MariaDB reads the passwords only when it initialises an empty `kimai-db` volume.
Replacing a secret afterwards does not change the password in the database, and
Kimai can then no longer log in.

## Installation

Run this once, in the root of the repository. It copies the content of
`quadlet/` to `~/.config/containers/systemd/kimai/` and lets systemd generate
the units.

```bash
podman quadlet install --application=kimai quadlet
```

The units cannot be enabled or disabled with `systemctl enable`. `kimai` is
started at every start of your systemd user instance, because
`quadlet/kimai.container` contains `WantedBy=default.target`.

## Settings

| Setting | Default |
|---|---|
| `PublishPort` | `127.0.0.1:8001:8001` |
| `DEFAULT_URI` | `http://localhost:8001/` |
| `TRUSTED_HOSTS` | `^localhost$` |
| `TRUSTED_PROXIES` | `127.0.0.1` |

`PublishPort` is the address and the port on the host. `DEFAULT_URI` is the
address for browsers, which Kimai uses in the links of its emails.
`TRUSTED_HOSTS` is a regular expression for the host names under which Kimai may
be requested, without the port. `TRUSTED_PROXIES` lists the addresses of the
[reverse proxies](#reverse-proxy) in front of the container.

Settings are changed in a file of your own, which systemd merges with the
installed `kimai.container`. The installed files stay unchanged, so a later
installation does not overwrite the settings. Create the directory and copy the
example, once:

```bash
mkdir -p ~/.config/containers/systemd/kimai.container.d
```

```bash
cp settings.conf.example ~/.config/containers/systemd/kimai.container.d/settings.conf
```

Edit the copy, and after every change let systemd read it:

```bash
systemctl --user daemon-reload
```

A running container keeps its old settings until it is [started again](#start).

Kimai answers a request for a host name that `TRUSTED_HOSTS` does not match with
the status 400. The container checks its own health with a request for the host
name of `DEFAULT_URI`, so the two settings have to agree; otherwise the start
fails. For the address `https://kimai.example.com/` the lines are:

```ini
Environment=DEFAULT_URI=https://kimai.example.com/
Environment=TRUSTED_HOSTS=^kimai\.example\.com$$
```

systemd reads a single `$` as the start of a variable and a single `%` as the
start of a specifier. Write each of them twice in the regular expression.

The default of `PublishPort` accepts connections from the local machine only.
The container serves HTTP without TLS.

### Reverse proxy

To reach Kimai over HTTPS, put a reverse proxy that ends the TLS connection in
front of the port of `PublishPort`, and set `DEFAULT_URI` and `TRUSTED_HOSTS` to
the address of the proxy. The proxy has to pass on the `Host` header of the
request and add the headers `X-Forwarded-For` and `X-Forwarded-Proto`. For
nginx:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

Kimai reads these headers only in requests that come from an address in
`TRUSTED_PROXIES`. Without that, it treats every request as HTTP and redirects
the browser to an `http` address. The setting accepts addresses and ranges such
as `10.0.0.0/8`, separated by commas, and no host names. The address is the one
from which the requests arrive inside the container. A request through the port
of `PublishPort` arrives from an address of the network `kimai`, not from
`127.0.0.1`. The log of the `kimai` container shows the address in each request
line, after the host name:

```bash
journalctl --user -u kimai -g ' "GET '
```

That address changes when the container is started again. Use the range of the
network, which this command prints:

```bash
podman network inspect kimai --format '{{range .Subnets}}{{.Subnet}}{{end}}'
```

Set `TRUSTED_PROXIES` only when the port of `PublishPort` cannot be reached
without passing the proxy. A client that connects from a listed address can put
any address in the headers.

## Start

```bash
systemctl --user restart kimai
```

The command starts what is not running: it creates the network and the volumes,
starts `kimai-db`, waits until the database accepts connections and then starts
`kimai`. It returns when Kimai answers requests. The first start downloads the
two images and creates the database schema, which takes several minutes.

Follow the progress in a second terminal:

```bash
journalctl --user -f -u kimai-db -u kimai
```

At every start the container brings the database schema to the version of the
image and builds the cache of Kimai, before the web server starts.

systemd gives `kimai` 900 seconds to answer requests. After that it stops the
container and starts it again, also when the update of the schema is still
running. When an update of a large database needs more time, add a section
`[Service]` with a higher `TimeoutStartSec` to `settings.conf`.

While the start script runs, it does not react to a stop:
`systemctl --user stop kimai` then takes 90 seconds, after which systemd ends
the container. Once the web server runs, a stop takes a few seconds.

## Start at boot

Your systemd user instance, and with it Kimai, starts at your first login and
stops when your last session ends. This command starts the user instance at boot
and keeps it running after you log out. Run it once.

```bash
loginctl enable-linger
```

## First user

A new instance has no user. This command creates an administrator and asks for
the password. Replace `USERNAME` and `EMAIL`.

```bash
podman exec -it kimai sh /usr/local/bin/with-database-url /opt/kimai/bin/console kimai:user:create USERNAME EMAIL ROLE_SUPER_ADMIN
```

Then open the address of `DEFAULT_URI` in a browser and log in.

## Commands of Kimai

`bin/console` of Kimai needs the address of the database in the environment
variable `DATABASE_URL`. A command that `podman exec` starts does not have that
variable. The script `quadlet/container/with-database-url`, which is mounted in
the container, sets it and runs the command that follows, as in the command
above. This lists the commands of Kimai:

```bash
podman exec kimai sh /usr/local/bin/with-database-url /opt/kimai/bin/console list kimai
```

## Processes

Inside the `kimai` container, the start script and the main process of Apache
run as root. The processes that handle requests run as `www-data`. With rootless
podman, root inside a container is the user who started the container, not root
on the host.

Kimai keeps a key for its sessions and login links in the file `.appsecret` of
the `kimai-data` volume. The start script creates it at the first start.

## Stop

```bash
systemctl --user stop kimai kimai-db
```

systemd removes the two containers. The volumes are kept, so the next start
continues with the same data.

## Update of Kimai

`quadlet/kimai.container` uses the image tag `2`, which the Kimai project moves
to each new release of Kimai 2. podman downloads the image only when it is
missing. To update, download the current image and start Kimai again:

```bash
podman pull docker.io/kimai/kimai2:2
```

```bash
systemctl --user restart kimai
```

The start updates the database schema. Make a backup of the database before: the
previous image does not work with the new schema.

The MariaDB series is written in two files: as the image tag in
`quadlet/kimai-db.container`, and as `serverVersion` in
`quadlet/container/with-database-url`. Change both together.

## Update of the files

After a change to a file in `quadlet/`, for example when a newer commit of this
repository is checked out, install the files again and start Kimai again.

```bash
podman quadlet install --replace --application=kimai quadlet
```

```bash
systemctl --user restart kimai
```

## Removal

Stop every unit first. Besides the containers, this ends the units that created
the network and the volumes, which systemd otherwise keeps until you log out.

```bash
systemctl --user stop 'kimai*'
```

Remove the installed files:

```bash
podman quadlet rm --recursive kimai
```

Your settings in `~/.config/containers/systemd/kimai.container.d/` are not
removed by this command.

The next command deletes the database and all files that Kimai stored.

```bash
podman volume rm kimai-db kimai-data kimai-plugins
```

Remove the network, the passwords and the images:

```bash
podman network rm kimai
```

```bash
podman secret rm kimai-db-password kimai-db-root-password
```

```bash
podman rmi docker.io/kimai/kimai2:2 docker.io/library/mariadb:12.3
```

## Prior work

Kimai documents Docker and Docker Compose, and has no documentation for podman.
These sources were compared before this setup was written:

- [Docker](https://www.kimai.org/documentation/docker.html) and [Docker
  Compose](https://www.kimai.org/documentation/docker-compose.html) in the
  documentation of Kimai
- [kimai/kimai#5218](https://github.com/kimai/kimai/issues/5218), a Quadlet
  fragment that passes the database password as a podman secret
- [kimai/kimai#5932](https://github.com/kimai/kimai/pull/5932), a change to the
  start script for a container that does not run as root
- [Tarow/nix-podman-stacks](https://github.com/Tarow/nix-podman-stacks), a Home
  Manager module that generates Quadlet units for Kimai

## Licence

GNU Affero General Public License, version 3 or later, the licence of Kimai. See
[LICENSE](LICENSE).
