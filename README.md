# ds-irods-rs-build

A program for creating an iRODS resource server that runs in a Docker container and is configured
for the CyVerse Data Store.

## Design

All of the sensitive information that would normally be set in the iRODS configuration files as
well as the clerver password have been removed. They must be provided in a file named
`cyverse-secrets.env` that will be loaded at run time.

docker-compose was chosen as the tool to manage the building of the top level image as well as
starting and stopping its container instance.

The program `prep-docker` was created to simplify the generation of the top level image's
`Dockerfile` file and the `docker-compose.yml` file. As its input, it takes an environment file that
should provide the resource server specific configuration values. In most cases, the generated files
can be used without modification.

There are three deployment artifacts: `Dockerfile`, `docker-compose.yml`, and `cyverse-secrets.env`.
Once the correct configuration values have been defined, these files will rarely need to be
modified. All changes to Data Store business logic will be made to the base image. This means that
only the following commands need to be executed to upgrade the resource server.

```bash
prompt> docker-compose build --pull
prompt> docker-compose up -d
```

If for some reason a base image upgrade doesn't work, the resource server can be reverted to the
last good base image by modifying the Dockerfile to use the tag of the good image. Used the commands
above to redeploy the reverted resource server.

## Setting up the Resource Server

This section describes what needs to be done to run a containerized iRODS resource server as part of
the CyVerse Data Store.

### Host Machine

The server hosting the containerized resource server needs to have docker-compose version 1.8 or
new installed.

There needs to be a user on the host machine that the container will use to run the iRODS resource
server.

Two directories on the hosting server's file system need to be setup. One directory will be used by
the container to store the files managed by the resource server. The other will be used to store
the generated log files. Both of these directories will need to be writable by the user running
iRODS.

For the rest of the CyVerse iRODS grid to be able to communicate with this resource server, the host
needs a public FQDN or IP address. This doesn't necessarily need to be the host's actually name or
IP address. DNS aliases and/or NAT can be used.  If NAT is used, there needs to be a static IP that
can be used to access the host.

The CyVerse iRODS grid will used the IP ports 1247-1248/TCP, 20000-20009/TCP, and 20000-20009/UDP to
communicate with this resource server. They need to be accessible from the IP address range
206.207.252.0/25. Please ensure that all local or institutional firewalls and router ACLs allow
access to these ports for that address range.

Any clients that need to connect directly to this resource server, will need to potentially use
1247/TCP, 20000-20009/TCP, and 20000-20009/UDP. The firewalls and router ACLs will need to allow
access from these clients as well.

Here are the minimum reasonable requirements for the host server. It should be running a 64-bit
operating system that has stable support for Docker, like CentOS 7. It should have at least two
cores and 8 GiB of memory as well. The file system hosting the vault should have at least 256 GiB of
storage, and the one hosting the logs should have at least 16 GiB.

### Preparing CyVerse's iRODS Zone

Before this generated image can be used, there are a few things that need to be done.

First, the resource server's UNIX file system resource needs to be defined within the CyVerse Data
Store. The vault path within the container will be a subdirectory of `/irods_vault` with the same
name as resource being served. For example, if the resource is to be named _demo_, then the vault
path will be `/irods_vault/demo`. If the hosting server's public name is _rs.domain.net_, then
defining the resource can be done with commands like the following.

```bash
prompt> iadmin mkresc demo 'unix file system' rs.domain.net:/irods_vault/demo
prompt> iadmin modresc demo status down
```

Next, the corresponding passthru resource needs to be created for the UNIX file system resource. The
name of the passthru resource needs to be the name of the UNIX file system resource with _Res_
appended. For example, if the UNIX file system resource is to be named _demo_, then the passthru
resource will be named _demoRes_. This will be the default resource served by the resource server.
This can be done with a set of commands like the following.

```bash
prompt> iadmin mkresc demoRes passthru
prompt> iadmin addchildtoresc demoRes demo
```

Finally, create a rodsadmin user for the resource server to use when connecting to other servers in
the grid as a client. If the chosen user name is `demo-admin`, the following commands can be used to
create the user with the password `SECRET_PASSWORD`.

```bash
prompt> iadmin mkuser demo-admin rodsadmin
prompt> iadmin moduser demo-admin password SECRET_PASSWORD
prompt> iadmin atg rodsadmin demo-admin
```

### Generating the Docker Source Files

The `prep-docker` program can be used to create a `Dockerfile` file and a `docker-compose.yml` file
that can be use to build and run a container hosting an iRODS resource server that is configured to
serve a given resource within the CyVerse Data Store.

As its first command line argument, `prep-docker` expects the name of a file defining a set of
expected environment variables. It also accepts an optional second argument specifying the directory
where created files will be written. If this isn't provided, the files will be written to the
current working directory.

The `prep-docker` expects several environment variables to be defined in an environment file when it
is executed.

Environment Variable | Required | Default       | Description
-------------------- | -------- | ------------- | -----------
`IRODS_CLERVER_USER` | no       | ipc_admin     | the name of the rodsadmin user representing the resource server within the zone
`IRODS_HOST_UID`     | no       |               | the UID of the hosting server to run iRODS as instead of the default user defined in the container
`IRODS_LOG_DIR`      | no       | `$HOME`/log   | the host directory where the container will mount the iRODS log directory (`/var/lib/irods/log`), `$HOME` is evaluated at container start time
`IRODS_RES_SERVER`   | yes      |               | the FQDN or address used by the rest of the grid to communicate with this server
`IRODS_RES_VAULT`    | no       | `$HOME`/vault | the host directory where the container will mount the vault, for the default, `$HOME` is evaluated at container start time
`IRODS_STORAGE_RES`  | yes      |               | the name of the unix file system resource that will be served

Here's an example.

```bash
prompt> cat build.env
IRODS_STORAGE_RES=demo
IRODS_RES_SERVER=rs.domain.net

prompt> prep-docker build.env project

prompt> ls project
docker-compose.yml  Dockerfile
```

### Running the Resource Server

docker-compose is used to run the iRODS resource server. The `docker-compose.yml` file assumes there
is a file named `cyverse-secrets.env` in the same directory. It should have the following
environment variables defined in it.

Environment Variable      | Description
------------------------- | -----------
`IRODS_CLERVER_PASSWORD`  | the password used to authenticate `IRODS_CLERVER_USER`
`IRODS_CONTROL_PLANE_KEY` | the encryption key required for communicating over the relevant iRODS grid control plane
`IRODS_NEGOTIATION_KEY`   | the encryption key shared by the iplant zone for advanced negotiation during client connections
`IRODS_ZONE_KEY`          | the shared secret used during server-to-server communication

Here's an example.

```bash
prompt> ls
docker-compose.yml  Dockerfile  cyverse-secrets.env

prompt> cat cyverse-secrets.env
###
# *** DO NOT SHARE THIS FILE ***
#
# THIS FILE CONTAINS SECRET INFORMATION THAT COULD BE USED TO GAIN PRIVILEGED
# ACCESS TO THE CYVERSE DATA STORE. PLEASE KEEP THIS FILE IN A SECURE PLACE.
#
###
IRODS_CLERVER_PASSWORD=SECRET_PASSWORD
IRODS_CONTROL_PLANE_KEY=SECRET_____32_byte_ctrl_plane_key
IRODS_NEGOTIATION_KEY=SECRET____32_byte_negotiation_key
IRODS_ZONE_KEY=SECRET_zone_key

prompt> docker-compose up -d --build
```
