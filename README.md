# ds-irods-rs-build

A program for creating an iRODS resource server configured for the CyVerse Data Store that runs
inside a Docker container.


## Design

The sensitive information that would be in the iRODS configuration files as well as the clerver
password have been removed. They must be provided in a file named `cyverse-secrets.env` that will
be loaded at run time.

docker-compose was chosen as the tool to manage the building of the top level image as well as
starting and stopping its container instance.

The program `prep-docker` was created to simplify the generation of the top level image's
`Dockerfile` file and the `docker-compose.yml` file. As its input, it takes an environment file
that should provide the resource server specific configuration values. In most cases, the
responsible party can use the generated files without modification.

The responsible party needs three artifacts to run the resource server: `Dockerfile`,
`docker-compose.yml`, and `cyverse-secrets.env`. After defining the correct configuration values,
these files will rarely need to be modified. All changes to Data Store business logic will be made
to the base image. This means that executing the following commands is enough to upgrade the
resource server.

```bash
prompt> docker-compose build --pull
prompt> docker-compose up -d
```

If for some reason a base image upgrade doesn't work, the responsibe party can revert the resource
server to the last good base image by modifying the Dockerfile to use the tag of the good image.
Use the commands above to redeploy the reverted resource server.


## Setting up the Resource Server

This section describes what needs to be done to run a containerized iRODS resource server as part
of the CyVerse Data Store.

### Host Machine

The server hosting the containerized resource server needs to have docker-compose version 1.8 or
newer installed.

There needs to be a user on the host machine that the container will use to run the iRODS resource
server.

Two directories on the hosting server's file system need to be setup. One directory will be the
vault. The other will store the generated log files. Both of these directories need to be writable
by the user running iRODS.

For the rest of the CyVerse iRODS grid to be able to communicate with this resource server, the
host needs a public FQDN or IP address. This doesn't necessarily need to be the host's actually
name or IP address. DNS aliases and/or NAT are acceptable. If using NAT, there needs to be a static
IP address that can be used to access the host.

The iRODS servers communicated over TLS encrypted connections. This server requires two files, one
for the private key and one for the certificate chain (the server's certificate plus the chain of
certificates to the certificate authority's root certificate). The certificate should be for the
host's public FQDN. Wildcard certificates are acceptable.

The CyVerse iRODS grid will used the IP ports 1247-1248/TCP, 20000-20009/TCP, and 20000-20009/UDP
to communicate with this resource server. They need to be accessible from the IP address range
206.207.252.0/25. Please ensure that all local or institutional firewalls and router ACLs allow
access to these ports for that address range.

Any clients that need to connect directly to this resource server, will need to potentially use
1247/TCP, 20000-20009/TCP, and 20000-20009/UDP. The firewalls and router ACLs will need to allow
access from these clients as well.

Here are the minimum, reasonable requirements for the host server. It should be running a 64-bit
operating system that has stable support for Docker, like CentOS 7. It should have at least two
cores and 8 GiB of memory as well. The file system hosting the vault should have at least 256 GiB
of storage, and the one hosting the logs should have at least 16 GiB.

### Preparing CyVerse's iRODS Zone

Before using the generated image, there is some preparation that needs to be done.

First, define the resource server's UNIX file system resource within the CyVerse Data Store. The
vault path within the container will be a subdirectory of `/irods_vault` with the same name as the
server's resource. For example, if _demo_ will be the resource name, then the vault path will be
`/irods_vault/demo`. If the hosting server's public name is _rs.domain.net_, then the following
commands will define the resource.

```bash
prompt> iadmin mkresc demo 'unix file system' rs.domain.net:/irods_vault/demo
prompt> iadmin modresc demo status down
```

Next, create the corresponding passthru resource to the UNIX file system resource. The name of the
passthru resource needs to be the name of the UNIX file system resource with _Res_ appended. For
example, if _demo_ is the UNIX file system resource, then _demoRes_ will be the passthru resource.
This server will use this by default. The following commands will do this.

```bash
prompt> iadmin mkresc demoRes passthru
prompt> iadmin addchildtoresc demoRes demo
```

Finally, create a rodsadmin user for the resource server to use when connecting to other servers
in the grid as a client. If the chosen user name is `demo-admin`, the following commands will
create the user with the password `SECRET_PASSWORD`.

```bash
prompt> iadmin mkuser demo-admin rodsadmin
prompt> iadmin moduser demo-admin password SECRET_PASSWORD
prompt> iadmin atg rodsadmin demo-admin
```

### Generating the Docker Source Files

Use the `prep-docker` program to create `Dockerfile` and `docker-compose.yml` files for building
and running a container hosting an iRODS resource server in CyVerse's Data Store.

As its first command line argument, `prep-docker` expects the name of a file defining a set of
expected environment variables. It accepts an optional second argument specifying the directory to
store the created files. If this isn't provided, the files will be written to the current working
directory.

The `prep-docker` expects the following environment variables to be defined in the environment
file.

Environment Variable        | Required | Default       | Description
--------------------------- | -------- | ------------- | -----------
`IRODS_CLERVER_USER`        | no       | ipc_admin     | the name of the rodsadmin user representing the resource server within the zone
`IRODS_HOST_UID`            | yes      |               | the UID of the hosting server to run iRODS
`IRODS_LOG_DIR`             | no       | `$HOME`/log   | the host directory where the container will mount the iRODS log directory (`/var/lib/irods/iRODS/server/log`), `$HOME` is evaluated at container start time
`IRODS_RES_SERVER`          | yes      |               | the FQDN or address used by the rest of the grid to communicate with this server
`IRODS_RES_VAULT`           | no       | `$HOME`/vault | the host directory where the container will mount the vault, for the default, `$HOME` is evaluated at container start time
`IRODS_STORAGE_RES`         | yes      |               | the name of the unix file system resource that will be served
`IRODS_TLS_CERT_CHAIN_FILE` | yes      |               | the absolute path to the TLS certificate chain file
`IRODS_TLS_KEY_FILE`        | yes      |               | the absolute path to the TLS key file

Here's an example.

```bash
prompt> cat build.env
IRODS_RES_SERVER=rs.domain.net
IRODS_STORAGE_RES=demo
IRODS_TLS_CERT_CHAIN_FILE=/etc/pki/tls/certs/rs.domain.net-chain.crt
IRODS_TLS_KEY_FILE=/etc/pki/tls/private/rs.domain.net.key

prompt> prep-docker build.env project

prompt> ls project
docker-compose.yml  Dockerfile
```

### Running the Resource Server

docker-compose manages the iRODS resource server. The `docker-compose.yml` file assumes there is a
file named `cyverse-secrets.env` in the same directory. It should have the following environment
variables defined in it.

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
