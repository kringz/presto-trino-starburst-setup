# Requirements 
- Docker
- Docker-Compose 
- About 16GB memory 
- At least 20GB local storage

# Configuration
### Memory Management
Inside the `presto` directory, you can tune the `config.properties` and `jvm.config` files for the coordinator and worker nodes. `config.properties` handles internal Presto memory management, while `jvm.config` manages the Java virtual machines that each Presto node runs in. All memory management is relative to the memory values provided in JVM configuration. 

### Catalogs
A catalog is an implementation of a Presto *connector*. A connector's sole responsibility is to connect to a particular data source. You can have multiple catalogs, i.e. `mysql.properties` and `mysql2.properties` –– each of these implement the MySQL connector. 

### Other Stuff
There's a bunch of info on Presto all over the place. I will of course recommend the [Starburst docs](https://docs.starburstdata.com/latest/index.html), but you can also join the [PrestoSQL Slack channel](https://prestosql.io/slack.html) and ask questions on StackOverflow – an impressively active community resides there. 

There's also a book that was just released on Presto. The environment is incredibly extensible, so keep that in mind when you have petabytes of user data to analyze. 

Catalog files have one requirements: `connector.name`. The rest of the required and optional properties can be found in the [Starburst connector documentation](https://docs.starburstdata.com/latest/connector.html). 

# Running 
Once you have installed the repository on a machine, you will need to create a Docker volume. The default Postgres volume for Hive metastore persistence is:

```sh
docker volume create postgres-data
```

You can inspect this volume: 

```sh
docker volume inspect postgres-data
```

Without a volume, data cannot be persisted. It is recommended that the volume data is backed up regularly to ensure that the metastore can persist indefinitely. 

To run the environment, run the following from the root directory of the repository:

```sh
docker-compose up -d
```

If you make changes to any of the resources (config files, Docker files, etc.), you can recompile your containers/images via:

```sh
docker-compose down 
docker-compose up -d --build
```

To view refreshing statuses of your Docker containers, you can run:

```sh
watch -n 5 sudo docker container ls
```

If you want to monitor system memory and have it refreshed periodically, you can run:

```sh
watch -n 5 free -m
```

# Services & Docker
The environment consists of five docker containers: three Presto nodes, a Hive metastore service, and a Postgres database for use by the Hive metastore. You can shell into any of these containers and use their respective command line tools as follows:

```sh
# Shell into a Presto node 
docker exec -it <container-name> bash 
presto-cli # Access the Presto CLI
presto-cli --debug # Access the Presto CLI in debug mode
/usr/lib/presto/bin/launcher run # Manually start the Presto service
/usr/lib/presto/bin/launcher stop # Manually stop the Presto service

# Shell into Postgres 
docker exec -it postgres bash 
psql --username=use database # Access the Postgres CLI 

# Shell into the Hive metastore service
docker exec -it metastore bash 
hive # Access the hive CLI (aka "Beeline")
```

Other useful Docker commands:

```sh
# List all images
docker images

# List all containers
docker container ls -a

# Remove a container or image by ID
docker rm <container-or-image-id>

# Tail logs of a running container 
docker logs <container-name>

# Restart a single container 
docker restart <container-name>
```

# Queries
It's time to query some data. Here are some tips:

[Read this article](https://eng.uber.com/engineering-sql-support-on-apache-pinot/). If you want to get the most out of Presto, you need to use some of its magic sugar like predicate pushdown.

```
Predicates are Boolean-valued functions in the WHERE clause of the query. Predicates represent the allowed value range for specific columns. We implemented predicate pushdown, which means that the Presto coordinator would push the predicate down to Presto workers to do best-effort filtering when fetching data from Pinot. When Presto workers fetch records from Pinot servers, our system preserves the predicates of the query the workers are operating with. By applying the predicates in the user’s query in the Presto workers, our system fetches only the necessary data from Pinot. For example, if the predicate of the Presto query is WHERE city_id = 1, utilizing predicate pushdown would ensure that workers only fetch records from Pinot segments where city_id = 1. Without predicate pushdown, they will fetch all data from Pinot. 
```

This is an example of predicate pushdown:

```
SELECT * FROM hive.benchmarks.urls WHERE id = 56225
```

The `WHERE id = 56225` evaluates to `True` or `False` – using table statistics, Presto can make smarter decisions about what data it actually needs to pay attention to during the query.

------------------

Presto is broken down into three distinct levels: catalogs, schemas, and tables/views. A catalog is essentially a data source; it is synonymous with the associated `catalog.properties` file. Schemas are equivalent to databases in most cases. Schemas can hold multiple tables and views (an implementation of a SQL query expressed as a logical table).

Here are a variety of useful queries:

```sh
# Show catalogs - run at root level
SHOW catalogs;

# Show schemas from a catalog
SHOW schemas FROM catalog;

# Show tables fro a schema
SHOW tables FROM catalog.schema;

# Assume a schema, then query a table name directly w/out the three-part name convention
USE catalog.schema;
SELECT COUNT(*) FROM table;

# View Presto node details
SELECT * FROM system.runtime.nodes;

# View executed queries in the current session
SELECT * FROM system.runtime.queries;

# Create a Hive schema
# This creates a "managed" location in Hive
CREATE SCHEMA hive.benchmarks 
WITH (location = 's3a://bucket/folder');

# Create a Hive table via CTAS operation
# This effectively ETLs your data from MySQL to Hive/ORC distributed file storage
CREATE TABLE hive.benchmarks.male_names AS (
   SELECT * FROM mysql.domain_fish.male_names
);

# Drop a table
DROP TABLE catalog.schema.table;

# Drop a schema
DROP SCHEMA catalog.schema;

# Federate between data sources
# This example joins MySQL data with Hive data (which is crazy)
USE mysql.domain_fish;
SELECT mn.first_name FROM male_names mn
    INNER JOIN hive.example.people p
ON mn.person_id = p.person_id;

# View file locations and sizes for Hive tables
SELECT DISTINCT "$path", "$file_size" FROM <table>;
```

# Benchmarks 
Below are the benchmarks that are used:

```sh
CREATE SCHEMA hive.persistence_test
WITH (location = 's3a://domainfish1/persistence_test');

CREATE TABLE hive.persistence_test.small AS (
   SELECT * FROM mysql.domain_fish.male_names
);

CREATE TABLE hive.persistence_test.big AS (
   SELECT * FROM mysql.domain_fish.dmoz_1m_com_net_only_ahrefs_reports_all_batch_combined_no_id_sor
);

CREATE TABLE hive.persistence_test.huge AS (
   SELECT 
      id,
      linked_domains,
      domain_rating,
      ahrefs_rank,
      total_links_count,
      links_for_domain,
      dofollow_links_count_desc,
      dofollow_backlinks_for_domain_percent,
      referring_domains,
      linked_domains_number,
      organic_traffic
   FROM mysql.domain_fish.urls
);
```