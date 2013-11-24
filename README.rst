docker-powerdns-authoritive
===========================
Runs an authoritive (master) PowerDNS server

This assumes you use a PostgreSQL backend.

Requirements
------------

- Start with a link for the Postgres database (name of the link should be
  ``pgsql``)
- Database details: ``PGSQL_USERNAME`` (default: ``powerdns``),
  ``PGSQL_PASSWORD`` (default: ``powerdns``), ``PGSQL_DATABASE`` (default:
  ``powerdns``)
- ``PDNS_AXFR_IPS``: the IP range that allows AXFR. Default: ``127.0.0.1/8``
  (loopback IPs).
- ``WEBPASSWD``: the password for the webserver

Start up
--------
- Generate password::

    export PG_ROOT_USER=powerdns_root
    export PG_ROOT_PASS=$(pwgen -c -n -1 -s 12)
    export PG_USER=powerdns
    export PG_PASS=$(pwgen -c -n -1 -s 12)

- PostgreSQL::

    PG_CONTAINER=$(docker run -d -name pgsql_server -e POSTGRESQL_USER=$PG_ROOT_USER -e POSTGRESQL_PASS=$PG_ROOT_PASS orchardup/postgresql)

  Create the database schema::

    PG_IP=$(docker inspect ${PG_CONTAINER} | grep IPAddress | cut -d '"' -f 4)

    createdb -U $PG_ROOT_USER -h $PG_IP powerdns
    createuser -E -P -U $PG_ROOT_USER -h $PG_IP $PG_USER

    # Fix schema files for correct users
    sed -i -e "s/__PGSQL_USER__/$PG_USER/" sql_schema/*.sql
    # Import schema
    psql -U $PG_ROOT_USER -h $PG_IP powerdns < sql_schema/00-*.sql
    psql -U $PG_ROOT_USER -h $PG_IP powerdns < sql_schema/01-*.sql

- Run::

    docker run -link pgsql_server:pgsql -e PDNS_AXFR_IPS=172.16.0.0/12 -e WEBPASSWD=changethispasswd -e PGSQL_USERNAME=$PG_USER -e PGSQL_PASSWORD=$PG_PASS -p 53:53 -p 8053:8053 -name pdns_master -d gvangool/powerdns-authoritive
