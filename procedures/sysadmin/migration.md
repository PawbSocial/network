# Migrating Mastodon to a new server

## Prerequisites

### Hardware

- 4GB RAM per instance (Postgres and Mastodon)
- 2 vCPUs (Must be a modern / recent CPU for performance)
- 16GB SSD / NVMe
- 1 Gbps uplink

## Installation

The target server must have Mastodon installed using the instructions in the [Official Documentation](https://docs.joinmastodon.org/admin/install/).

When you reach **Checking out the code**, instead of using the official repository, use the https://github.com/PawbSocial/mastodon repo.

**DO NOT** run the interactive setup wizard (`mastodon:setup`).

## Migrating data

### Enable maintenance mode

Through Cloudflare, go into each of the domains, click `Workers Routes`, and add a route with the following settings:

| Field | Value |
|-------|-------|
| Route | `[domain]/*` |
| Service | `pawbsocial-maintenance` |
| Environment | `production` |

The site should begin redirecting to the maintenance page after about 10-30 seconds, ensure this has been completed before migrating.

### Stop all services

Exit back to the root user and gracefully stop all of the Mastodon services.

```
service mastodon-* stop
```

### Back up old database and config

On the old server, login, sudo, and `su - mastodon` to change to the the `mastodon` user, and run the following:

```
pg_dump -Fc -U [user] [database] > /tmp/mastodon_[instancename]_[date].dump
cp /home/mastodon/live/.env.production /tmp/.env.production
chmod 777 /tmp/mastodon_[instancename]_[date].dump /tmp/.env.production
```

### Copy the files to the new server

I'd recommend doing this with `sftp`, copying the file locally, then uploading it. Alternatively, you can set up an SSH key on the old server and authorize it on the new one to copy the file directly.

### Create and restore the database

#### Create the database

If you already created the database following the official guide, ensure the db user has permissions for the database.

1. Open a Postgres shell with `sudo -u postgres psql`
2. Run the following
```
GRANT ALL PRIVILEGES ON DATABASE [database] TO [user]
\c [database] postgres
GRANT ALL ON SCHEMA public TO [user]
\q
```

#### Restore the database

With the .dump file on the new server, login, sudo, and `su - mastodon` to change to the mastodon user, and ensure the .dump file is readable by the mastodon user.

```
pg_restore -Fc -U [user] -n public --no-owner --role=[user]  -d [database] -v --exit-on-error mastodon_[instancename]_[date].dump
```

Postgres can have some *issues* when it comes to restoring the .dump file, so here's some things I've had to do:

#### `auth: ...` errors

Typically, this will be that the database name either doesn't match your current user (e.g. if the db is `mastodon_pawb` and the user is `mastodon`).

1. Open `/etc/postgresql/15/main/pg_hba.conf` in the editor of your choice.
2. Scroll down to the line that reads `local` `all` `all` `peer` and change `peer` to `md5`. This changes Postgres to use password-based authentication, instead of just checking the current user.
3. Open a Postgres shell with `sudo -u postgres psql`
4. `\password [user]` (check the `.env.production` db_pass line)
5. `\q` to exit the Postgres shell
6. `service postgresql restart`

### Set up nginx

In Cloudflare, create a new EC Origin cert + key pair, and save those to a directory on the server. For this guide, we're using `/etc/ssl/mastodon`.

Copy the configuration from the Mastodon install:

```
cp /home/mastodon/live/dist/nginx.conf /etc/nginx/sites-available/mastodon
ln -s /etc/nginx/sites-available/mastodon /etc/nginx/sites-enabled/mastodon
```

Open the `/etc/nginx/sites-enabled/mastodon` configuration file in your chosen editor, and update `server_name` to be the domain, then uncomment the `ssl_certificate` and `ssl_certificate_key` lines and point them to the location of the origin cert and key files.

Reload nginx for the changes to take effect:

```
systemctl reload nginx
```

### Copy the systemd templates

Copy the templates from the Mastodon install folder:

```
cp /home/mastodon/live/dist/mastodon-*.service /etc/systemd/system/
```

Finally, enable and start the new services:

```
systemctl daemon-reload
systemctl enable --now mastodon-web mastodon-sidekiq mastodon-streaming
```

### Update Cloudflare

Once you've verified the service is working, update the A (or AAAA) records on Cloudflare to point to the new IP(s), and remove the Workers Route.
