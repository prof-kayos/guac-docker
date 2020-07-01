# Install Guacamole via Docker

## Install Docker

1. Ensure system is up to date

    ```bash
    sudo apt update && sudo apt upgrade -y
    ```

1. Install Docker and add current user

    ```bash
    curl -sSL https://get.docker.com | sh
    ```

1. Install Docker Compose

    ```bash
    sudo apt install -y docker-compose
    ```

1. Test Docker

    ```bash
    docker run hello-world
    ```

1. Create guacamole user

    ```bash
    sudo adduser guacamole
    sudo usermod -aG docker guacamole
    sudo passd guacamole <guacamole password>
    sudo su - guacamole
    ```

1. Validate you are a member of the docker group

    ```bash
    groups | grep docker
    ```

1. Test Docker

    ```bash
    docker run hello-world
    ```

## Install/Configure Guacamole

1. Pull Guacamole

    ```bash
    sudo docker pull guacamole/guacd
    sudo docker pull guacamole/guacamole
    sudo docker pull mysql
    ```

1. Initialize MySQL

    ```bash
    docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --mysql > initdb.sql

    mkdir /tmp/scripts
    cp initdb.sql /tmp/scripts

    docker run --name guac-mysql --restart unless-stopped -v /tmp/scripts:/tmp/scripts -e MYSQL_ROOT_PASSWORD=<MySQL Password> -d mysql:latest
    history -c
    ```

1. Get Schema/Admin sql files and upload to MySQL container

    ```bash
    wget https://raw.githubusercontent.com/glyptodon/guacamole-client/master/extensions/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-mysql/schema/001-create-schema.sql
    wget https://raw.githubusercontent.com/glyptodon/guacamole-client/master/extensions/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-mysql/schema/002-create-admin-user.sql
    docker cp 001-create-schema.sql guac-mysql:/home/
    docker cp 002-create-admin-user.sql guac-mysql:/home/
    docker exec -it guac-mysql /bin/bash
    ```

1. Connect to MySQL container and update configuration

    ```bash
    docker exec -it guac-mysql /bin/bash

    mysql -u root -p'<MySQL Password>'

    CREATE DATABASE guacamole;
    CREATE USER 'guacamole' IDENTIFIED BY '<guac user password>';
    GRANT SELECT,INSERT,UPDATE,DELETE ON guacamole.* TO 'guacamole';
    FLUSH PRIVILEGES;
    quit

    cd /home
    mysql -u root -p'<MySQL Password>' -D guacamole < 001-create-schema.sql
    mysql -u root -p'<MySQL Password>' -D guacamole < 002-create-admin-user.sql
    ```

1. Exit container via <Ctrl+d>
1. Create index.html file for redirect

    ```bash
    vi index.html
    ```

    ```html
    <!DOCTYPE html>
    <html>
        <head>
            <meta http-equiv="Refresh" content="0; url=http://<FQDN for site>/guacamole/#/" />
        </head>
    </html>
    ```
1. Obtain guacamole-auth-totp jar from Apache (http://guacamole.apache.org/releases).
1. Copy guacamole-auth-totp-1.1.0.jar to $HOME directory.
1. Start Guacamole containers

    ```bash
    docker run --name guacd --restart unless-stopped -d guacamole/guacd

    Exmaple with new MySQL instance:

    docker run --name guacamole --restart unless-stopped --link guacd:guacd --link guac-mysql:mysql \
    -e MYSQL_DATABASE='guacamole' \
    -e MYSQL_USER='guacamole' \
    -e MYSQL_PASSWORD='<MySQL Password>' \
    -v $(echo $HOME)/index.html:/usr/local/tomcat/webapps/ROOT/index.html \
    -v $(echo $HOME)/guacamole-auth-totp-1.1.0.jar:/opt/guacamole/mysql/guacamole-auth-totp-1.1.0.jar \
    -d -p 8080:8080 guacamole/guacamole

    Example with exiting MySQL database:

    docker run --name guacamole --restart unless-stopped --link guacd:guacd \
    -e MYSQL_HOSTNAME='192.168.16.202' \
    -e MYSQL_DATABASE='guacamole_db' \
    -e MYSQL_USER='guacamole_user' \
    -e MYSQL_PASSWORD='<PASSWORD>' \
    -v $(echo $HOME)/index.html:/usr/local/tomcat/webapps/ROOT/index.html \
    -v $(echo $HOME)/guacamole-auth-totp-1.1.0.jar:/opt/guacamole/mysql/guacamole-auth-totp-1.1.0.jar \
    -d -p 8080:8080 guacamole/guacamole
    ```

1. Tune Tomcat

    ```bash
    sudo docker exec -it guacamole /bin/bash

    sed -i 's/redirectPort="8443"/redirectPort="8443" server="" secure="true"/g' /usr/local/tomcat/conf/server.xml

    sed -i 's/<Server port="8005" shutdown="SHUTDOWN">/<Server port="-1" shutdown="SHUTDOWN">/g' /usr/local/tomcat/conf/server.xml

    rm -Rf /usr/local/tomcat/webapps/docs/
    rm -Rf /usr/local/tomcat/webapps/examples/
    rm -Rf /usr/local/tomcat/webapps/manager/
    rm -Rf /usr/local/tomcat/webapps/host-manager/

    chmod -R 400 /usr/local/tomcat/conf
    ```

1. Exit container via <Ctrl+d>
1. Browse to <http://localhost:8080/guacamole/> and login using the credentials guacadmin/guacadmin.
