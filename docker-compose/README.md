# Running Hub in Docker (Using Docker Compose)

This is the bundle for running with Docker Compose. 

## Contents

Here are the descriptions of the files in this distribution:

1. docker-compose.yml - This is the primary docker-compose file.
2. docker-compose.dbmigrate.yml - Docker-compose file *used one time only* for migrating DB data from another Hub instance.
3. docker-compose.externaldb.yml - Docker-compose file to start Hub using an external PostgreSQL instance.
4. hub-webserver.env - This contains an env. entry to set the host name of the main server so that the certificate name will match as well as port definitions.
5. hub-proxy.env - This file container environment settings to to setup the proxy.
6. hub-postgres.env - Contains database connection parameters when using an external PostgreSQL instance.

## Requirements

Hub has been tested on Docker 17.03.x (ce/ee). The minimum version of docker-compose to use this bundle must be able to read Docker Compose 2.1 files.

## Migrating DB Data from Hub/AppMgr
----

This section will describe the process of migrating DB data from a Hub instance installed with AppMgr to this new version of Hub. There are a couple of steps.

NOTE: Before running this restore process it's important that only a subset of the containers are initially started. Sections below will walk you through this.

### Prerequisites

Before beginning the database migration, you'll need to have a PostgreSQL Dump file containing the data from the previous Hub instance.

#### Making the PostgreSQL Dump File from an AppMgr Installation

These instructions require being on the same server that the Hub in installed.
Instructions can be found in the Hub install guide in Chapter 4, Installing the Hub AppMgr.

#### Making the PostgreSQL Dump File from an an Existing Hub Container

```
./bin/hub_create_data_dump.sh <local destination path for the dump>
```

This creates a dump in the container, and copies over to the local destination directory.

### Restoring the Data
----

#### Starting PostgreSQL

There is a separate compose file that will start PostgreSQL for this restore process. You can run this:

```
docker-compose -f docker-compose.dbmigrate.yml -p hub up -d 
```

Once this has brought up the DB container the next step is to restore the data.

#### Restoring the DB Data

There is a script in './bin' that will restore the data from an existing DB Dump file.

```
./bin/hub_db_migrate.sh <path to dump file>
```

Once you run this, you'll be able to stop the existing containers and then run the full compose file.

##### Possible Errors

When an dump file is restored from an AppMgr version of Hub, you might see a couple of errors like:

```
 ERROR:  role "blckdck" does not exist
```

Along with a few surrounding errors. At the end of the migration you might also see:

```
WARNING: errors ignored on restore: 7
```

This is OK and should not affect the data restoration.

#### Stopping the Containers

```
docker-compose -f docker-compose.dbmigrate.yml -p hub stop
```

## Running 

Note: These command might require being run as either a root user, a user in the docker group, or with 'sudo'.

```
docker-compose -f docker-compose.yml -p hub up -d 
```


## Running with External PostgreSQL

Hub can be run using a PostgreSQL instance other than the provided hub-postgres docker image.
```
     $ docker-compose -f docker-compose.externaldb.yml -p hub up -d 
```
This assumes that the external PostgreSQL instance has already been configured (see External PostgreSQL Settings below).


## Configuration

There are a couple of options that can be configured in this compose file. This section will convert these things:

Note: Any of the steps below will require the containers to be restarted before the changes take effect.

### Web Server Settings
----

#### Host Name Modification

When the web server starts up, if it does not have certificates configured it will generate an HTTPS certificate.  

Configuration is needed to tell the web server which real host name it will listening on so that the host names can match. Otherwise the certificate will only have the service name to use as the host name.

To modify the real host name, edit the hub-webserver.env file to update the desired host name value.

#### Port Modification

The web server is configured with a host to container port mapping.  If a port change is desired, the port mapping should be modified along with the associated configuration.

To modify the host port, edit the port mapping as well as the hub-webserver.env file to update the desired host and/or container port value.

If the container port is modified, any healthcheck URL references should also be modified using the updated container port value.

### Proxy Settings

There are currently three containers that need access to services hosted by Black Duck Software:

* registration
* jobrunner
* webapp

If a proxy is required for external internet access you'll need to configure it. 

#### Steps

1. Edit the file hub-proxy.env
2. Add any of the required parameters for your proxy setup

#### Authenticated Proxy Password

There are two methods for specifying a proxy password when using Docker Compose.

* Mount a directory that contains a text file called 'HUB_PROXY_PASSWORD_FILE' to /run/secrets 
* Specify an environment variable called 'HUB_PROXY_PASSWORD' that contains the proxy password

There are the services that will require the proxy password:

* webapp
* registration
* jobrunner

### External PostgreSQL Settings

The external PostgreSQL instance needs to initialized by creating users, databases, etc., and connection information must be provided to the _webapp_ and _jobrunner_ containers.

#### Steps

1. Create a database user named _blackduck_ with admisitrator privileges.  (On Amazon RDS, do this by setting the "Master User" to "blackduck" when creating the RDS instance.)
2. Run the _external-postgres-init.pgsql_ script to create users, databases, etc.; for example,
   ```
   psql -U blackduck -h <hostname> -p <port> -f external_postgres_init.pgsql postgres
   ```
3. Using your preferred PostgreSQL administration tool, set passwords for the *blackduck* and *blackduck_user* database users (which were created by step #2 above).
4. Edit _hub-postgres.env_ to specify database connection parameters.
5. Create a file named 'HUB_POSTGRES_USER_PASSWORD_FILE' with the password for the *blackduck_user* user.
6. Create a file named 'HUB_POSTGRES_ADMIN_PASSWORD_FILE' with the password for the *blackduck* user.
7. Mount the directory containing 'HUB_POSTGRES_USER_PASSWORD_FILE' and 'HUB_POSTGRES_ADMIN_PASSWORD_FILE' to /run/secrets in both the _hub-webapp_ and _hub-jobrunner_ containers.

#### Secure LDAP Trust Store Password

There are two methods for specifying an LDAP trust store password when using Docker Compose.

* Mount a directory that contains a text file called 'LDAP_TRUST_STORE_PASSWORD_FILE' to /run/secrets
* Specify an environment variable called 'LDAP_TRUST_STORE_PASSWORD' that contains the LDAP trust store password.

This configuration is only needed when adding a custom Hub web application trust store.

# Connecting to Hub

Once all of the containers for Hub are up the web application for hub will be exposed on port 443 to the docker host. You'll be able to get to hub using:

```
https://hub.example.com/
```

## Using Custom webserver certificate-key pair
*For the upgrading users from version < 4.0 : 'hub_webserver_use_custom_cert_key.sh' no longer exists so please follow the updated instruction below if you wish to use the custom webserver certificate.*
----

Hub allows users to use their own webserver certificate-key pairs for establishing ssl connection.
* Mount a directory that contains the custom certificate and key file each as 'WEBSERVER_CUSTOM_CERT_FILE' and 'WEBSERVER_CUSTOM_KEY_FILE' to /run/secrets 

In your docker-compose.yml, you can mount by adding to the volumes section:
```
webserver:
    image: blackducksoftware/hub-nginx:4.0.0-SNAPSHOT
    ports: ['443:443']
    env_file: hub-webserver.env
    links: [webapp, cfssl]
    volumes: ['webserver-volume:/opt/blackduck/hub/webserver/security', '/directory/where/the/cert-key/is:/run/secrets']
```
* Start the webserver container

## Hub Reporting Database
----

Hub ships with a reporting database. The database port will be exposed to the docker host for connections to the reporting user and reporting database.

Details:

* Exposed Port: 55436
* Reporting User Name: blackduck_reporter
* Reporting Database: bds_hub_report
* Reporting User Password: initially unset

Before connecting to the reporting database you'll need to set the password for the reporting user. There is a script included in './bin' of the docker-compose directory called 'hub_reportdb_changepassword.sh'. 

To run this script you must:

* Be on the docker host that is running the PostgreSQL database container
* Be able to run 'docker' commands. This might require being 'root' or in the 'docker' group depending on your docker setup.

To run the change password command:

```
./bin/hub_reportdb_changepassword.sh blackduck
```

Where 'blackduck' is the new password. This script can also be used to change the password for the reporting user after it has been set.

Once the password is set you should now be able to connect to the reporting database. An example of this with 'psql' is:

```
psql -U blackduck_reporter -p 55436 -h localhost -W bds_hub_report
```

This should also work for external connections to the database.

# Scaling Hub

The Job Runner in the only container that is scalable. Job Runners can be scaled using:

```
docker-compose -p hub scale jobrunner=2
```

This example will add a second Job Runner container. It is also possible to remove Job Runners by specifying a lower number than the current number of Job Runners. To return back to a single Job Runner:

```
docker-compose -p hub scale jobrunner=1
```
