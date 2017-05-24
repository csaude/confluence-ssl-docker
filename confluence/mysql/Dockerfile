FROM alpine:3.4

ENV MYSQL_ROOT_PASSWORD=root-password
ENV MYSQL_DATABASE=confluencedb
ENV MYSQL_USER=confluence
ENV MYSQL_PASSWORD=confluence-user-password
ENV MYSQL_DATA_DIRECTORY=/opt/data/system/mysql

ADD confluence-mysql.cnf /root/.my.cnf
ADD backup /usr/bin/backup
ADD localtime /etc/localtime

RUN apk add --update mariadb mariadb-client; \
    rm /usr/bin/mysql_client_test_embedded; \
    rm /usr/bin/mysqltest; \
    rm /usr/bin/mysqltest_embedded;

COPY run.sh /run.sh

VOLUME /opt/data

EXPOSE 3306

CMD ["/run.sh"]
