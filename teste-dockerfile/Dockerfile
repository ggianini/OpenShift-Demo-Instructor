FROM registry.redhat.io/rhel8/mysql-80:1-152

ENV SUMMARY="MySQL 8.0 SQL database server POPULATED" \
    DESCRIPTION="Image based on the official MYSQL image curated by Red Hat and should be used for demonstration purposes."

LABEL summary="$SUMMARY" \
    description="$DESCRIPTION" \
    io.k8s.description="$DESCRIPTION" \
    io.k8s.display-name="MySQL 8.0 Populated" \
    io.openshift.expose-services="3306:mysql" \
    io.openshift.tags="database,mysql,mysql80,mysql-80" \
    com.redhat.component="mysql-80-container" \
    name="rhel8/mysql-80" \
    version="1" \
    com.redhat.license_terms="https://www.redhat.com/en/about/red-hat-end-user-license-agreements#rhel" \
    usage="podman run -d -p 3306:3306 rhel8/mysql-80" \
    maintainer="Hugo Pfeffer <hpfeffer@redhat.com>"



ENV MYSQL_USER=redhat
ENV MYSQL_PASSWORD=redhat
ENV MYSQL_ROOT_PASSWORD=redhat
ENV MYSQL_DATABASE=sampledb

COPY ./populate.sh /opt/app-root/src
RUN sleep 60 
CMD ["/bin/bash", "/opt/app-root/src/populate.sh"]

