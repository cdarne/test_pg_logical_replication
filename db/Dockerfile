FROM postgres:12

ADD scripts/1_init.sql /docker-entrypoint-initdb.d
ADD scripts/2_copy.sql /docker-entrypoint-initdb.d
ADD config/postgresql.conf /etc/postgresql/postgresql.conf

RUN chmod a+r /docker-entrypoint-initdb.d/*
RUN chmod a+r /etc/postgresql/postgresql.conf