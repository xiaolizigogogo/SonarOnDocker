# Sonar On Docker

Complete Docker Compose guide to configure and run **SonarQube** + **MySQL** docker applications.

[![](https://img.shields.io/badge/Docker%20Hub-info-blue.svg)](https://hub.docker.com/r/thyrlian/sonar/)
[![Build Status](https://travis-ci.org/thyrlian/SonarOnDocker.svg?branch=master)](https://travis-ci.org/thyrlian/SonarOnDocker)

<img src="https://github.com/thyrlian/SonarOnDocker/blob/master/Banner.png">

```
                        ##         .
                  ## ## ##        ==
               ## ## ## ## ##    ===
           /"""""""""""""""""\___/ ===
      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~~ ~ /  ===- ~~~
           \______ o           __/
             \    \         __/
              \____\_______/
```

## Pitfalls

Orchestrating Docker with compose sounds easy, but sometimes could be full of pitfalls in practice.  If you just wanna use it, jump directly to [**Getting Started**](https://github.com/thyrlian/SonarOnDocker/blob/master/README.md#getting-started).

Running both SonarQube and MySQL containers together by compose could encounter such error:

```console
Can not connect to database. Please check connectivity and settings (see the properties prefixed by 'sonar.jdbc.').
```

It's because MySQL database initialization takes a bit longer than SonarQube's boot time, especially when there is no persisted database yet.

Failed Attempts:

* [**`depends_on`**](https://docs.docker.com/compose/compose-file/#/dependson) **option**: it will start services in dependency order, but won't wait for the dependent service to be ready.

* **wait script**: [wait-for-it](https://github.com/vishnubob/wait-for-it) recommended by Docker's [Controlling startup order in Compose](https://docs.docker.com/compose/startup-order/), which checks the database port and waits.  Unluckily, this doesn't help.  The reason is that the port will be available right after the database container starts, while it doesn't mean that the database connection is ready.  Just forget about `nc -v -n -z -w1 $HOST $PORT`.

* **Database logs**: the readiness status of MySQL is written in its logs, could be read by `grep 'ready for connections'`.  Well, keep in mind, the logs are only accessible with inside the MySQL container, or from the host machine, but not from the SonarQube container.  Maybe you wanna try to persist MySQL logs to the host directory by adding `command: bash -c "mkdir -p /var/log/mysql && mysqld 2>&1 | tee /var/log/mysql/mysql.log"` and `volumes: ./data/mysql:/var/log/mysql`, and then mount it to (share the volume with) the SonarQube container, so that it could be gotten from there.  Hmm, I don't want to mess it up with adding `command` and `volumes` configurations at both services' sides.  Is there any better idea?

* **Database command**: how about simply executing `mysql -e "select 1"` to check the database availability?  Yep, but wait a second, the SonarQube container doesn't have mysql client installed, and we have no control of the official SonarQube docker image.

* **Web Server**: yet another hack - setting up a minimal (one-liner) web server at the MySQL container, responds with the database status, just like `while true; do echo -e 'HTTP/1.1 200 OK\n\n $(db_status)' | nc -l -p 9999; done`.  Unfortunately again, netcat is not installed by the MySQL container.

* [**`HEALTHCHECK`**](https://docs.docker.com/engine/reference/builder/#/healthcheck) **instruction**: new feature since Docker v1.12, but not for docker-compose yet.  Usage: `HEALTHCHECK [OPTIONS] CMD command`.  Still, you have to write the command on your own, to tell docker what to check.

* **JDBC**: finally, here comes an easy solution - creating a [java file](https://github.com/thyrlian/SonarOnDocker/blob/master/data/sonarqube/docker/com/basgeekball/db/Detector.java), which has some code utilizing JDBC to check the database availability (Java environment and JDBC jar are both available within the SonarQube container).  Just override the entrypoint of the SonarQube container, first check the database availability via this java code, then run the default entrypoint shell script when the database is ready.  Pretty slick and it works great!

## Getting Started

### Setup

1. Make sure that you've cloned the whole project, particularly the ***Detector.java*** under [`data/sonarqube/docker/com/basgeekball/db`](https://github.com/thyrlian/SonarOnDocker/blob/master/data/sonarqube/docker/com/basgeekball/db/Detector.java).

2. Pull the latest docker images for [**SonarQube**](https://hub.docker.com/_/sonarqube/) and [**MySQL**](https://hub.docker.com/_/mysql/):

    ```console
    docker pull sonarqube
    docker pull mysql
    ```

3. (Optional - Mac only) There is a permission problem when mount a host directory in MySQL container using `boot2docker`.

    ```console
    [ERROR] InnoDB: Operating system error number 13 in a file operation.
    [ERROR] InnoDB: The error means mysqld does not have the access rights to the directory.
    ```

    **Solution**:

    * Build a custom MySQL image for Mac:

        ```console
        docker build -t mysql_mac [PATH_OF_THIS_REPO_ON_YOUR_DISK]/mysql_mac/
        ```

    * Edit ***docker-compose.yml***, replace `image: mysql` by `image: mysql_mac`.

4. In order to persist data, you need to setup mounting data volumes: replace two mounting points under volumes in ***docker-compose.yml*** file.

    ```
    - [path_to_persist_sonar_data_on_host]:/opt/sonarqube/extensions
    - [path_to_persist_mysql_data_on_host]:/var/lib/mysql
    ```

    Note: the path to persist data on host could be a relative path, e.g.: `./data/xyz`

### Play

```console
docker-compose -f [PATH_OF_THIS_REPO_ON_YOUR_DISK]/docker-compose.yml up
```

## Persist Data

* All historical analysis data, imported rules, changed settings are saved here.

    ```
    /var/lib/mysql
    ```

* SonarQube's plugins.

    ```
    /opt/sonarqube/extensions
    ```

Don't persist ElasticSearch indices (which is located at `/opt/sonarqube/data/es`), let it rebuild by itself (otherwise could cause problem during upgrading).

## Upgrading

⚠ Always keep a **backed up database** in case upgrade fails and roll back is needed.

### MySQL

1. Perform a logical backup on the old version of MySQL

    ```console
    mysqldump -u sonar -p --opt sonar > [path_to_mysql_backup]/sonar.sql
    ```

2. Start a MySQL docker container (new version of MySQL)

    ```console
    docker run -i -t -v [path_to_mysql_backup]:/tmp -v [path_to_persist_db]:/var/lib/mysql mysql /bin/bash
    ```

3. Start MySQL server

    ```console
    /etc/init.d/mysql start
    ```

4. Start MySQL client

    ```console
    mysql
    ```

5. Create and use the database

    ```sql
    create database sonar;
    use sonar;
    ```

6. Grant privileges to user

    ```sql
    grant all on sonar.* to 'sonar'@'%' identified by 'sonar';
    grant all on sonar.* to 'sonar'@'localhost' identified by 'sonar';
    grant usage on *.* to sonar@localhost identified by 'sonar';
    grant all privileges on sonar.* to sonar@localhost;
    ```

7. Restore the backup file (by executing SQL script)

    ```sql
    source /tmp/sonar.sql
    ```

8. Quit MySQL client

    ```sql
    exit
    ```

9. Stop MySQL server

    ```console
    /etc/init.d/mysql stop
    ```

Now you have successfully restored the database on the new version of MySQL.  The database data are stored in ***path_to_persist_db*** of your host.

### SonarQube

SonarQube Server upgrade process is automated, you have nothing to manually change in the SonarQube Database.

**Migration path**: `[your_version] -> LTS (if exists) -> [expected_version]`

[Upgrading guide by SonarQube](http://docs.sonarqube.org/display/SONAR/Upgrading) (just for reference, please follow below steps.)

Don't try to stop the SonarQube server, if you kill the process, the SonarQube container exits immediately.  So you can't really upgrade SonarQube by hand within its container.  Don't worry, just try below steps.

Steps:

1. Use the new sonarqube image in ***docker-compose.yml***;
2. Run `docker-compose up`;
3. Wait until sonarqube is up.

For big SonarQube upgrading, it also requires database upgrading, but this happens automatically.

1. After the new SonarQube container is up, open its web page, you'll be redirected to a maintenance page;

    ```console
    sonarqube_1  | 2099.12.31 12:00:00 WARN  web[o.s.s.p.DatabaseServerCompatibility] Database must be upgraded. Please backup database and browse /setup
    sonarqube_1  | 2099.12.31 12:00:00 INFO  web[o.s.s.p.Platform] DB needs migration, entering safe mode
    ```

2. Open ***http://[your_sonarqube_url]:9000/setup***;

3. Click the **Upgrade** button.

The database upgrade can take several minutes.  When the DB migration ends successfully, the page will display "Database is up-to-date", then redirect you to home page.

##License
Copyright (c) 2016 Jing Li. **SonarOnDocker** is released under the GNU General Public License version 3. See the [LICENSE](https://github.com/thyrlian/SonarOnDocker/blob/master/LICENSE) file for details.
