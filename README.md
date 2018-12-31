# postgresql-ram
## What is this?
Allow running a PostgreSQL database server from a container in RAM (working example).

This requires support for docker-compose tmpfs, specifically file format 3.6+.

This especially useful for running tests, and I wanted to avoid configuring a local
postgresql server on the host, or setting up and mounting an external RAMDisk, which is an
inherently non-portable task and has a different life cycle than a container.
This is a vastly better option than using sqlite :memory for running tests where your
production server will be postgresql.

tmpfs is almost as good as a native RAM based filesystem, and makes building and destroying
a database extremely fast. The data, of course, does not persist, and the optimal way to
reset everything is to simply restart the container. The server self initialises. See
the section below on creating your own database(s) on startup.

I found very little on the web with a working example, so publish this in the hope that it
might help someone out looking for exactly what this does.
It provides all the necessary speed for testing code on a fast replica of the same database
software actually used in production.

## Tweaks
- Scripts can be added (sql or shell) to the scripts sub-folder to run intial database setup.
  Shell scripts must end in .sh: exe-bit scripts are run, non-exec-bit scripts are sourced.
  SQL scripts must end in .sql.
- The amount of memory allocated is by default 100mB and can be tweaked in docker-compose.yml.
  This is likely a little excessive for a simple application where you're not likely to ingest
  or generate a lot of fixture data.
- This container is derived from the official PostgreSQL docker container, and may be configured
  further according to [instructions there](https://hub.docker.com/_/postgres/).
- Running a different version of postgresql is easily done by modifying the FROM
  line in Dockerfile to nominate an alternative base.
  Some changes may be required in arguments for the CMD, however - the existing ones will
  work with postgresql 9 through 11. Non-official images may work if they provide a similar
  protocol for customising database initialisation.

## Testing with Django
Running postgresql on tmpfs is useful for testing.
However Django seems to only allow overriding the database name and none of the other connection
details for the test database.
I therefore took the following (hackish) approach to get the django test module to
use the test database running on port 5433.
Snippet from the Django settings module:
```python
import sys
DATABASES = {
    'default': {    # postgresql://rjf_web@db/rjf
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'USER': 'username',
        'HOST': os.environ.get('DBHOST', 'localhost'),
        'NAME': os.environ.get('DBNAME', 'defaultdb'),
        'PORT': 5433 if 'test' in sys.argv else 5432,
        'TEST': {   # sqlite://:memory
            'NAME': os.environ.get('TEST_DBNAME', 'defaultdb_test'),
        }
    }
}
```
and changing the ports section for the service in docker-compose.yml as follows:
```yaml
  postgres-ram:
    build:
      context: ./postgres-ram
    image: postgres-ram
    volumes:
        - type: tmpfs
          target: /var/lib/postgresql/data
          volume:
            nocopy: true
          tmpfs:
            size: 104857600
    networks:
        - internal
    expose:
        - "5432"
    ports:
        - "127.0.0.1:5433:5432"
```

## Credits
Many thanks to [Manni Wood's blog article](https://www.manniwood.com/postgresql_94_in_ram/index.html)
which provided some handy insight into ramdisk-friendly postgresql configuration tweaks.
