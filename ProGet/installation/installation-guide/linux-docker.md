---
title: Linux Installation Guide
sequence: 100
show-headings-in-nav: true
keywords: proget, installation, linux
---

ProGet for Linux is available using our official Docker image.

## Prerequisites {#prerequisites data-title="Prerequisites"}

### Docker

Docker must be installed and the docker daemon running on your server. If you don't already have Docker installed, you can get [installation instructions for your specific Linux distribution from Docker](https://docs.docker.com/engine/installation/#installation).

Once Docker is up and running, you are ready to continue. Note that Docker commands generally have to be issued by members of the docker group or with root/sudo privileges, so if you encounter errors with these commands make sure your account is in the docker group (`adduser myusername docker` and then log out and back in) or you are issuing them with the appropriate sudo/su depending on your configuration.

### Network

First, you'll need to create a network for the SQL Server and ProGet containers

```
docker network create proget
```

### SQL Server Database

ProGet requires a SQL Server database. You can either host this database externally or simply [use a SQL Server Docker image](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-configure-docker); it doesn't matter how it's hosted, as long as your ProGet instance can access it. For example, to start up a SQL Server container that you only intend to use with ProGet, use the command:

```
docker run --name proget-sql \
      -e 'ACCEPT_EULA=Y' -e 'MSSQL_SA_PASSWORD=‹YourStrong!Passw0rd›' \
      -e 'MSSQL_PID=Express' --net=proget --restart=unless-stopped \
      -d mcr.microsoft.com/mssql/server:2017-latest
```

**Note:** This example specifies the free SQL Express edition. This is adequate for most ProGet installations, but you can use any other edition as well if you have the license for it.

Once you have a SQL Server instance up and running, you'll need to create an empty database. For example, to create a database called **ProGet** on the SQL Server instance running in the **proget-sql** container:

```
docker exec -it proget-sql /opt/mssql-tools/bin/sqlcmd \
   -S localhost -U SA -P '‹YourStrong!Passw0rd›' \
   -Q 'CREATE DATABASE [ProGet] COLLATE SQL_Latin1_General_CP1_CI_AS'
```

**Note:** You can create the database however you want, but to avoid issues make sure you specify its collation as **SQL_Latin1_General_CP1_CI_AS**.

## Licensing

You'll need to have a valid license key once you get ProGet up and running. You are welcome to use either a license key that you already have or request a new one from [my.inedo.com](https://my.inedo.com).

## Starting the ProGet Docker Image {#starting data-title="Starting the Container"}

The ProGet Docker image contains a web server and a background service. To get this image and start it, use the `docker run` command. If you'd like to just get started right away with our defaults, you can just use the command below as-is, or continue reading for an explanation on the arguments and how to provide additional configuration values.

```
docker run -d -v proget-packages:/var/proget/packages -p 80:80 --net=proget \
    --name=proget --restart=unless-stopped -e PROGET_DB_TYPE=SqlServer \
    -e PROGET_DATABASE='Data Source=proget-sql; Initial Catalog=ProGet; User ID=sa; Password=‹YourStrong!Passw0rd›' \
    inedo/proget:latest
```

<table style="margin-top: 10px; margin-bottom: 10px;">
    <tbody>
        <tr>
            <td style="width: 400px;"><code>-d</code></td>
            <td>
                Starts the container in detached mode. Without this argument,
                Docker will block your current terminal session and output
                ProGet's logs to your terminal.
            </td>
        </tr>
        <tr>
            <td style="width: 400px;"><code>--net=proget</code></td>
            <td>
                Putting the containers into a Docker network lets them see each
                other and prevents other Docker containers from accessing them.
            </td>
        </tr>
        <tr>
            <td style="width: 400px;"><code>--restart=unless-stopped</code></td>
            <td>
                Tells Docker to restart the container unless it is explicitly
                stopped using <code>docker stop</code>. This makes the container
                automatically restart after the host reboots.
            </td>
        </tr>
        <tr>
            <td style="width: 400px;"><code>-e PROGET_DB_TYPE=SqlServer</code></td>
            <td>
                This is a required variable, and indicates that ProGet is connecting to SQL Server. Without this, ProGet will
                assume it is connecting to a since-deprecated PostgreSQL instance instead.
            </td>
        </tr>
        <tr>
            <td style="width: 400px;"><code>-e PROGET_DATABASE=...</code></td>
            <td>
                The connection string for SQL Server. The value specified in the example here
                assumes you are using the <code>proget-sql</code> container. To connect to a
                different instance, just change this connection string to the appropriate value.
            </td>
        </tr>
        <tr>
            <td style="width: 400px;"><code>-v proget-packages:/var/proget/packages</code></td>
            <td>
                Persists ProGet's packages in the <code>proget-packages</code> Docker volume.
            </td>
        </tr>
        <tr>
            <td style="width: 400px;"><code>-p 80:80</code></td>
            <td>
                Exposes TCP port 80 of the container to port 80 of the host, so that
                browsers can access the ProGet web application. If you don't want to
                use port 80, you can change the first port number to whatever you
                would like; make sure to change the BasedUrl in Admin > Advanced Settings if the ports aren't the same.
            </td>
        </tr>
        <tr>
            <td style="width: 400px;"><code>--name=proget</code></td>
            <td>
                Names the container <code>proget</code> so it can be easily referenced
                using other Docker commands. If you don't specify a name, Docker
                will generate one for you.
            </td>
        </tr>
        <tr>
            <td style="width: 400px;"><code>inedo/proget:latest</code></td>
            <td>
                This is the repository and tag for ProGet on Docker Hub. If you want
                to run a specific version instead of the latest, just replace <code>latest</code>
                with the appropriate ProGet release number. Note that downgrades will only
                work if there have been no database schema changes.
            </td>
        </tr>
    </tbody>
</table>

When the container is running, you should be able to browse to the ProGet web UI on the exposed port.

## Upgrading to a new version {#upgrading data-title="Upgrading"}

```
docker stop proget

docker rename proget proget-old

docker pull inedo/proget:latest

docker run -d --volumes-from=proget-old \
    -p 80:80 --net=proget \
    --name=proget --restart=unless-stopped \
    -e PROGET_DB_TYPE=SqlServer -e PROGET_DATABASE='Data Source=proget-sql; Initial Catalog=ProGet; User ID=sa; Password=‹YourStrong!Passw0rd›' \
    inedo/proget:latest

docker rm proget-old
```

## Troubleshooting {#troubleshooting data-title="Troubleshooting"}

If you aren't able to browse to the website, here's a few troubleshooting steps you can try:

1. Run `docker ps` - this will display all of the currently running Docker containers. If you don't see ProGet listed, then the container isn't running and you can issue a `docker logs proget` command to display the log messages from when it last ran; there was most likely some kind of initialization error that prevented it from starting up.
2. Run `docker logs proget` - this will display ProGet's log messages. Look for any error or warning messages for clues as to what's wrong.
3. Make sure a firewall on the host server isn't blocking requests on the specified port.
4. Run `docker restart proget` to stop the container and then restart it.

## Stopping the Container

If you need to stop the ProGet container, the `docker stop proget` command should do it. You can determine if it is running with the `docker ps` command. To start the container after it has been created, use the `docker start proget` command.

## HTTPS Support {#tls data-title="SSL/TLS Support"}

ProGet does not handle TLS termination. It is recommended that you use a third party reverse proxy to terminate TLS and forward requests to ProGet. This has the added benefit of allowing multiple websites and applications to be hosted on the same IP address.

Any HTTP reverse proxy can work, but four specific examples are provided below:

<table>
<tbody>
<tr>
<th>Caddy</th>
<td>[Proxy](https://github.com/caddyserver/caddy/wiki/v2:-Documentation#reverse_proxy)</td>
<td>[TLS](https://github.com/caddyserver/caddy/wiki/v2:-Documentation#tls)</td>
</tr>
<tr>
<th>nginx</th>
<td>[Proxy](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)</td>
<td>[TLS](https://docs.nginx.com/nginx/admin-guide/security-controls/terminating-ssl-http/)</td>
</tr>
<tr>
<th>Apache</th>
<td>[Proxy](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html)</td>
<td>[TLS](https://httpd.apache.org/docs/2.4/mod/mod_ssl.html)</td>
</tr>
<tr>
<th>HAProxy</th>
<td>[Proxy](https://www.haproxy.com/blog/the-four-essential-sections-of-an-haproxy-configuration/)</td>
<td>[TLS](https://www.haproxy.com/blog/haproxy-ssl-termination/)</td>
</tr>
</tbody>
</table>

## Docker Compose
Alternatively, you can use docker compose to instantiate your ProGet environment.  Create a docker-compose.yaml file, replacing all values denoted with curly braces in the below script.
```
version: '3.3'

services:
  server:
    image: inedo/proget:latest
    restart: unless-stopped
    environment:
      - PROGET_DB_TYPE=SqlServer 
      - "PROGET_DATABASE=Server=db;Database={your ProGet database name};User Id={your ProGet user name};Password={your ProGet user passwrod}"
    volumes:
      - /srv/proget/data/packages:/var/proget/packages
      - /srv/proget/data/extensions:/var/proget/extensions
    ports:
      - {your ProGet port}:80

  db:
    image: mcr.microsoft.com/mssql/server:2017-latest
    restart: unless-stopped
    volumes:
      - {your path to mssql data}:/var/opt/mssql/data
      - {your path to mssql logs}:/var/opt/mssql/log
      - {your path to mssql secrets}:/var/opt/mssql/secrets
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_PID=Express
      # This line can be removed once you have created your sql database and user
      - MSSQL_SA_PASSWORD={your sa password}
    volumes:
      - /srv/mssql/data:/var/opt/mssql
    # Ports can be removed once you have created your sql database and user
    ports:
      - {your ProGet port}:80
```
Create your containers by running the following bash command
```
docker-compose up -d
```

## Limitations and Known Issues {#limitations data-title="Limitations"}

Although ProGet's codebase is largely platform-independent, there are a few platform-specific features from the Windows version that had to be omitted:

{.docs}
- **Extension Management** - Extensions can be installed from the ProGet web application just like on Windows installations, but the hosting Docker container must be manually restarted after the extensions are installed using a `docker restart proget` command.
- **NuGet Source Server** - This requires the pdbstr.exe Windows binary, which will not run on Linux short of some kind of emulation layer like Wine.
