# Redis vs Postgres (Logged & Unlogged tables)

## Setting up
1. Run the docker compose file. It will initialize both Postgres and Redis containers

## Benchmarking
### Postgres
1. Create the two tables and populate them with random data:
```sql
DROP TABLE IF EXISTS logged_benchmark_table;
CREATE TABLE logged_benchmark_table (
                                                   id SERIAL PRIMARY KEY,
                                                   data TEXT NOT NULL
);
INSERT INTO unlogged_benchmark_table (data)
SELECT md5(random()::text)
FROM generate_series(1, 200000);

DROP TABLE IF EXISTS unlogged_benchmark_table;
CREATE UNLOGGED TABLE unlogged_benchmark_table (
                                                   id SERIAL PRIMARY KEY,
                                                   data TEXT NOT NULL
);
INSERT INTO unlogged_benchmark_table (data)
SELECT md5(random()::text)
FROM generate_series(1, 200000);
```

2. Now, let's access our Postgres container:
`docker exec -i -t redis-vs-postgres-postgres-1 /bin/bash`

3. And create two SQL scripts that will be the basis of our benchmark:
```shell
cat > /select_unlogged.sql <<EOF
\set id random(1, 100000)
SELECT * FROM unlogged_benchmark_table WHERE id = :id;
EOF
```
```shell
cat > /select_logged.sql <<EOF
\set id random(1, 100000)
SELECT * FROM logged_benchmark_table WHERE id = :id;
EOF
```

4. Run the benchmark for the unlogged table:
```shell
 pgbench -c 10 -j 2 -T 10 -f /select_unlogged.sql -U postgres -d postgres
```

You will see the result as something like:
```shell
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 150718
number of failed transactions: 0 (0.000%)
latency average = 0.679 ms
initial connection time = 5.752 ms
tps = 14724.204238 (without initial connection time)
```

5. Run the benchmark for the logged table:
```shell
 pgbench -c 10 -j 2 -T 10 -f /select_logged.sql -U postgres -d postgres
```

You will see the result as something like:
```shell
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 159293
number of failed transactions: 0 (0.000%)
latency average = 0.627 ms
initial connection time = 11.574 ms
tps = 15946.025786 (without initial connection time)
```

### Redis
1. Access your Redis container: `docker exec -i -t redis-vs-postgres-redis-1 /bin/bash`

2. Generate the random data:
```shell
for i in {1..100000}; do
  redis-cli -h redis SET key$i value$i
done
```
3. Run the benchmark:
```shell
redis-benchmark -q -n 100000 -c 10 -P 10 GET key:rand
```

4. You will see the result as something like: 
```shell
GET key:rand: 892857.12 requests per second, p50=0.095 msec      
```

