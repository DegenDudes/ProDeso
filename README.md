# ProDeso

Simple docker compose stack running postgres DB with Deso Backend.

This is for local development, testing and data analysis only. Dont expose this stack to the public internet.

## Dependencies

If you are on a server, install [Docker](https://docs.docker.com/engine/install/).

If you are on a desktop, install [Docker Desktop](https://www.docker.com/products/docker-desktop).

Make sure you give docker access to atleast 16gb of memory, and 300GB of discspace. On Docker Desktop you can do this in Settings > Resources > Advanced.

I presume your system has git installed already.

## Quick Start

1. Clone this repo locally: 
`git clone https://github.com/DeSoDev/ProDeso.git`

2. Open the folder:
`cd ProDeso`

3. Edit the `.env` file and change `POSTGRES_PASSWORD` value from `change_me` to a password you want to use

4. Bring up the containers:
`docker compose up -d`

5. Watch the logs for sync progress: `docker logs -tf backend 2>&1 | egrep "Received block|Received header bundle"`

## Sync Time

IMPORTANT: It can take 24 hours for the full blockchain to be synced. To minimise disruptions and issues, the backend is forced to connect to the primary node `deso-seed-4.io` using the `CONNECT_IPS` option in the `.env` file. 

## How to query the DB

You can use any postgres compatible tool or GUI on your local machine to query the DB. Use the values from the `.env` file if you changed them.

* HOST: `localhost`
* PORT: `5432`
* USER: `postgres`
* PASSWORD: `change_me`
* DB: `deso`

Here are some sample queries to run.

### What's the latest synced block & time?

```sql
SELECT to_timestamp(max(timestamp)), max(height) FROM pg_blocks;
```

### Get details about a user

Note: that the user may not yet exist if the chain is not fully synced.

```sql
SELECT * FROM pg_profiles WHERE LOWER(username)='tijn'
```

### Get hex public key for a user

You cant use Deso Base58 Public Key in Postgres easily. Instead you can get the hex encoded public key and use that.

```sql
SELECT encode(pkid,'hex') FROM pg_profiles WHERE LOWER(username)='tijn';

```

### Get the top posts for a user

Note: Using a hex public key (`02a9192d6d3fdf38347606e58e96960b4126b6f3c1fb53244032700cdc97a00b18`) from the previous query with `\x` prepended.

```sql
SELECT encode(post_hash,'hex'),body,comment_count,like_count,repost_count,quote_repost_count,diamond_count
FROM pg_posts 
WHERE poster_public_key='\x02a9192d6d3fdf38347606e58e96960b4126b6f3c1fb53244032700cdc97a00b18'
ORDER BY diamond_count DESC 
LIMIT 10;
```

### Get a users posts in the last day

Note: This uses a sub query to lookup the public key of a username.

```sql
SELECT * 
FROM pg_posts 
WHERE 
	to_timestamp(timestamp/1e9) >= current_timestamp - interval '1 day'
    AND poster_public_key IN (select public_key FROM pg_profiles WHERE LOWER(username)='tijn')
    AND parent_post_hash IS NULL
    AND reclouted_post_hash IS NULL
ORDER BY timestamp ASC
```



## Some gotchas

* The default db does not come with many indexes setup - so if you are running large queries with several joins it can be rather slow

* Postgres does not include a BASE58 function, so you cant copy a public key from a deso node, and use it in your queries. Instead look up the hex pub key for a user, or use a subquery with their username as highlighted in the SQL examples above.

## Useful commands

Get backend logs from last 10min:

```shell
docker logs --since 10m backend
```

Show last 100 log lines & follow:

```shell
docker logs -n 100 -f backend
```

Update the images & restart affected containers:

```shell
docker compose pull
docker compose up -d
```

To stop the containers: 

```shell
docker compose down
```

To wipe the container volumes (this deletes synced blockchain data): 

```shell
docker compose down -v --remove-orphans
```

Check blockchain sync status: either [open in your browser](http://localhost:17001/api/v0/health-check) or `curl http://localhost:17001/api/v0/health-check`. This shows 200 if the full blockchain has been downloaded.

## Limitations

This Deso Backend uses Postgres for data storage, and excludes the usual DESO frontend.

This means:

* It is read only - you can not yet post transactions to ANY postgres node

* It does not have notifications

* There is no frontend, but you can access the backend api at port 17001, eg http://localhost:17001/api/v0/get-exchange-rate

* Deso may be migrating to MySQL as Postgres is not performant enough.

* The Postgres backend code is Beta and has not had much development over the last few months. 

* If the backend crashes or panics for whatever reason, you may need to wipe volumes and resync the blockchain and db.

## Troubleshooting

* The first time you bring up this container stack, it can take some time (5-10s) for the database connection to come up. The docker compose config is set to wait for 6 seconds before starting the backend. In most instances this is ok. If the backend container exits complaining of the db connection not being available, just rerun `docker compose up -d`.

## Todo

- [ ] Add redash or another good data analysis web frontend to the stack.