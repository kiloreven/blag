+++
title = 'Improving Django testing with seed database'
date = 2024-02-05T19:30:00+01:00
draft = false
+++

> The code used as example in this post can be found in [this git repo](https://github.com/kiloreven/demo-django-seed-db)

At FOSDEM24, [Denny Biasiolli](https://www.dennybiasiolli.com/) gave an excellent [talk on optimizing Django migrations for testing](https://fosdem.org/2024/schedule/event/fosdem-2024-1894-django-migrations-friend-or-foe-optimize-them-for-testing/). This talk has a great run-through of the migration flow in Django, and different ways to improve the time and resources spent applying Django migrations. This can be further improved by using a seed database, especially in CI/CD pipelines.

# Seed databases
A seed database contains an initial set of data, for instance a set of migrations and/or fixtures. Using seed databases in a Django workflow can be really useful when doing iterative development that requires flushing the DB, like migrations or CI pipeline setup.

This is also a powerful tool to use for the CI runs themselves, saving time on migrating from scratch.

# How-to
_This example is based on Poetry, Docker Compose, PostgreSQL and Github Actions -- your setup might be different so your milage may vary_

---
TL;DR:
1. Run migrations against empty database
2. Dump contents of database to file
3. Add file to code tree
4. Use the file as initial database in CI

---

## Run migrations with empty DB
It's important that your seed database is generated using an empty database. The DB dump will include all parts of the database, including data. This method can also be used to deploy your app for production, for example if you run a multi-instance environment. To avoid shipping testing- or user data, it's important to start off with a clean database locally.

If you're using `docker-compose` for local development, you can stop the database service, remove the data volume, and bring it back up.


`📁 Commands for flushing a Docker volume`
``` bash
$ docker-compose down
$ docker volume rm demo-django-seed-db_database
$ docker-compose up -d
```

Once your clean database is up, you run your migrations as usual.

`📁 Running Django migrations through Poetry`
``` bash
# Example using poetry
$ poetry run ./app/manage.py migrate
```

## Dump contents of database to file
There are multiple ways extract the contents of the database to a file, and for PostgreSQL, the easiest one is to use `pg_dump`. This command can be called from within the database container, and the output can be piped directly to a file.

To make this process painless in the future, the command should be documented, for instance in a README, Makefile, or a bash script, like the one below.

`📁 dump-db.sh`
``` bash
#!/usr/bin/bash
# Dump the database to a local SQL file named `seed-db.sql`
# Password, docker-compose service and username are all `postgres` in this example
docker-compose exec -e PGPASSWD=postgres postgres pg_dump -U postgres > seed-db.sql
```

This makes the process simple and reproducible.

`📁 Running DB dumping script`
``` bash
$ chmod +x dump-db.sh
$ ./dump-db.sh
```

## Add file to code tree
The database dump should be committed to the code tree, and versioned along the rest of the codebase. It represents the state of your database model at a given point in time, so it makes sense for the file to follow the rest of the codebase.

`📁 Committing seed database`
``` bash
git add seed-db.sql
git commit -m '🗃️ chore(db): Update seed database'
git push
```

### Use the file as initial database in CI
To import the database in your CI job, you can use a Github action to load the the dump into the database, through a pre-built Postgres image.

`📁 .github/workflows/ci.yml (extracted snippets)`
``` yaml
# [...]
jobs:
  build:
    name: Build
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres
        env:
          POSTGRES_PASSWORD: postgres

    # [...]

    steps:
    - name: Check out repository code
      uses: actions/checkout@v4

    # [...]

    - name: Restore database
      uses: tj-actions/pg-restore@v6
      with:
        database_url: "postgres://postgres:postgres@localhost:5432/postgres"
        backup_file: "seed-db.sql"
        postgresql_version: "16"

    - name: Run migrations
      shell: bash
      run: poetry run ./app/manage.py migrate
```

The demo app contains 8 migrations. If the Django migration command was starting from a completely empty database, we would see our 8 migrations listed, as well as the migrations for Django's default contrib apps. However, the output from Github Actions shows that no migrations needed to be applied.


`📁 Output from Github Actions`
```
Run poetry run ./app/manage.py migrate
  
Operations to perform:
  Apply all migrations: admin, app, auth, contenttypes, sessions
Running migrations:
  No migrations to apply.
```

# Conclusion
Setting up a CI pipeline to use a seed database can significantly cut down on time and resource requirements for CI workflows, especially for large codebases with many migrations. The initial investment to set up a good pipeline and maintenance processes can quickly pay off, and reduce iteration time for developers.

There are many variations for all parts of the architecture (i.e. MySQL or SQLite for database; Jekyll or Gitlab Runner for CI; bare metal or shared server for development). Covering all of these would take a lot more than just one blog post.

# Addendum: Considerations

## Load file on service startup when possible

The best way to load the database dump into Postgres is to make the file available to the container when it first starts up. This removes the need for a separate process/container with the `pg_load` executable.

The [official Docker image for Postgres](https://hub.docker.com/_/postgres) will automatically run any SQL file (`.sql` or gzip-compressed `.sql.gz`) against the database when it starts up. Just make sure the file is available at `/docker-entrypoint-initdb.d` inside the container when it first starts up, and the contents will be executed automagically. This only happens the first time the DB starts up (empty data volume), so it's safe to keep this configuration in `docker-compose` or similar scripts for persistent databases.

Some CI systems, like Github Actions, do not support mounting files from the repo directly into the service, meaning that we have to execute a separate SQL transaction against the database from our CI steps.

## Maintenance

A seed database only gives a snapshot of the migrations that exist at the time of commit, and should be maintained regularely. If your database version changes, you should also update your seed database.

## CI step ordering
The seed database should be loaded as late as possible in your script, just before the migrations. The reason for this is that the process of restoring the database can be non-reproducible, meaning that the end state is different between runs. This can be caused by things like automatic timestamping at the time of import, logs etc.

Caching tools in CI often use a hash of the current state to decide whether anything has changed or not. They only cache the steps leading up to the first one that alters the state. If the state differs between runs, this makes the cache miss, which can lead to degraded perfomance in CI. By running the database import just before the migration, you increase the chances of hitting the cache for the previous steps.

## Automatic seeding with docker-compose
If you're iterating over migrations in local development, or want to reduce the time to get your app up and running from scratch, you can configure `docker-compose` to use your seed database when starting up for the first time (without data).

In this example, the file `seed-db.sql` in the root of the app repo.

`📁 docker-compose.yaml`
``` yaml
version: "3.8"
services:
    postgres:
        image: "postgres:16"
        ports:
            - "5432:5432"
        volumes:
            - database:/var/lib/postgresql/data
            - ./seed-db.sql:/docker-entrypoint-initdb.d/seed-db.sql
        environment:
            POSTGRES_PASSWORD: "postgres"

volumes:
    database:
```

Bringing the project up the empty database (`docker-compose up -d`) will load the seed database, and no migrations need to be applied.

`📁 Running Django-migrations locally`
``` bash
$ poetry run ./app/manage.py migrate

Operations to perform:
  Apply all migrations: admin, app, auth, contenttypes, sessions
Running migrations:
```

---

# Changelog
2024-02-05: Initial version
