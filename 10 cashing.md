Exercise 
1. Setup Redis locally
2. put and get some data
3. Measure time taken
4. compare it with database

Solution

# 🚀 Exercise: Redis vs Database on Ubuntu

## **1. Setup Redis locally**

```bash
sudo apt update
sudo apt install redis-server -y
```

Enable and start Redis:

```bash
sudo systemctl enable redis-server
sudo systemctl start redis-server
```

Check status:

```bash
redis-cli ping
# should return PONG
```

---

## **2. Put and Get Some Data (Redis)**

```bash
redis-cli
127.0.0.1:6379> SET user:1 "Keshav"
OK
127.0.0.1:6379> GET user:1
"Keshav"
```

Or using Python:

```python
import redis

r = redis.Redis(host='localhost', port=6379, db=0)

r.set("user:1", "Keshav")
print(r.get("user:1").decode())
```

---

## **3. Measure Time Taken**

### Redis Benchmark (built-in tool)

```bash
redis-benchmark -q -n 100000 -c 50 SET foo bar
redis-benchmark -q -n 100000 -c 50 GET foo
```

* `-n` → number of requests
* `-c` → concurrent clients

### Python Benchmark

```python
import redis, time

r = redis.Redis(host='localhost', port=6379, db=0)

# Write benchmark
start = time.time()
for i in range(10000):
    r.set(f"user:{i}", f"value-{i}")
print("Redis SET time:", time.time() - start)

# Read benchmark
start = time.time()
for i in range(10000):
    r.get(f"user:{i}")
print("Redis GET time:", time.time() - start)
```

---

## **4. Compare with Database (MySQL Example)**

### Install MySQL

```bash
sudo apt install mysql-server -y
sudo mysql_secure_installation
```

### Create Table

```sql
CREATE DATABASE testdb;
USE testdb;
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100)
);
```

### Python Benchmark for MySQL

```python
import mysql.connector, time

db = mysql.connector.connect(
    host="localhost",
    user="root",      # adjust if you created another user
    password="yourpassword",
    database="testdb"
)
cursor = db.cursor()

# Insert benchmark
start = time.time()
for i in range(10000):
    cursor.execute("INSERT INTO users (name) VALUES (%s)", (f"value-{i}",))
db.commit()
print("MySQL INSERT time:", time.time() - start)

# Read benchmark
start = time.time()
cursor.execute("SELECT name FROM users")
cursor.fetchall()
print("MySQL SELECT time:", time.time() - start)
```

---

## **Expected Results**

* **Redis**: microseconds per operation, \~100k+ ops/sec.
* **MySQL/Postgres**: milliseconds per operation, \~1k–10k ops/sec depending on setup.
* Redis is much faster because it works **in-memory**, while databases involve **disk I/O and transactions**.


<img width="717" height="1000" alt="image" src="https://github.com/user-attachments/assets/c88658e1-41e9-47f2-a53b-68b6062dbc5a" />

<img width="595" height="902" alt="image" src="https://github.com/user-attachments/assets/fd48e71a-a646-4949-a0e2-4c8c0ecece4f" />

<img width="1215" height="680" alt="image" src="https://github.com/user-attachments/assets/d4b45329-27e1-44e1-9773-61c34cfd4797" />


Ah, now it makes perfect sense why your **MySQL GET (\~1.34s for 100k queries)** looks almost the same as **Redis GET (\~1.44s)**. Let me break it down clearly:

---

## 1️⃣ MySQL is hitting a cached row

* Your query is always:

```sql
SELECT name FROM users WHERE id=1;
```

* `id` is the **primary key**, so MySQL uses the **primary key index** (B-tree lookup, very fast).
* After the first fetch, MySQL caches the row in the **InnoDB buffer pool** (memory).
* Therefore, almost all 100k queries are served **from memory**, not disk.
* Result: near in-memory speed → comparable to Redis.

---

## 2️⃣ Redis is always in-memory

* Redis **stores all data in RAM**.
* `GET foo` fetches the value directly from memory → very low latency.
* That’s why Redis still shows slightly faster **throughput (69k+ RPS)** and lower latency distribution than MySQL, but the difference is small for this trivial query.

---

## 3️⃣ Your current benchmark is **best-case for MySQL**

* Single-row query
* Indexed by primary key
* Cached in memory
* No joins, no aggregations
* Localhost, no network latency

✅ In real-world applications, MySQL will be **slower than Redis** when:

* Query multiple rows or ranges (`SELECT * FROM users WHERE id > 50000`)
* Dataset exceeds RAM → disk reads occur
* Complex queries (joins, GROUP BY, ORDER BY)
* Multiple concurrent writes and reads → locks/contention

---

### 4️⃣ How to see the difference clearly

To stress MySQL more like Redis under realistic conditions:

**Random row GET (not cached single ID):**

```bash
mysqlslap --user=keshav --password=keshav \
  --concurrency=50 --iterations=1 \
  --number-of-queries=100000 \
  --create-schema=sbtest \
  --query="SELECT name FROM users WHERE id=FLOOR(1 + RAND()*100000);"
```

* This forces MySQL to fetch **different rows**, cache misses may occur, throughput drops.

**Range scan / large dataset:**

```bash
mysqlslap --user=keshav --password=keshav \
  --concurrency=50 --iterations=1 \
  --number-of-queries=10000 \
  --create-schema=sbtest \
  --query="SELECT * FROM users WHERE id BETWEEN 50000 AND 60000;"
```

---

### ✅ Summary

* **Single cached row** → MySQL can approach Redis speed.
* **Redis** → consistently fast for all keys because always in RAM.
* **MySQL** → fast only for trivial, cached queries; slows down with disk access, large ranges, or complex queries.

---

If you want, I can make a **side-by-side comparison table** with **RPS and latency** for Redis vs MySQL using:

1. Single cached key
2. Random key
3. Range query


Perfect ✅ Let’s make a clear **side-by-side comparison** for Redis vs MySQL under different query scenarios. I’ll use your benchmark numbers and realistic estimates for stressed cases.

| Scenario                      | Redis            | MySQL (cached single row)   | MySQL (random row, some cache miss) | MySQL (range query, disk involved)  |
| ----------------------------- | ---------------- | --------------------------- | ----------------------------------- | ----------------------------------- |
| **Operation**                 | GET / SET        | SELECT by PK (id=1)         | SELECT random id                    | SELECT range of 10k rows            |
| **Total requests**            | 100,000          | 100,000                     | 100,000                             | 10,000                              |
| **Concurrency**               | 50               | 50                          | 50                                  | 50                                  |
| **Time taken**                | 1.44 s           | 1.345 s                     | \~4–6 s (depends on buffer pool)    | \~10–15 s (depends on disk)         |
| **Requests per second (RPS)** | 69,396           | 74,400                      | 16,700–25,000                       | 700–1,000                           |
| **Latency avg**               | 0.473 ms         | \~0.45 ms                   | \~2–5 ms                            | \~10–20 ms                          |
| **Latency min**               | 0.104 ms         | 0.104 ms                    | 0.5 ms                              | 5 ms                                |
| **Latency max**               | 2.967 ms         | 2–3 ms                      | 20 ms                               | 50+ ms                              |
| **Notes**                     | Always in-memory | Trivial query, fully cached | Cache misses start to appear        | Disk I/O dominates; complex queries |

---

### 🔑 Insights

1. **Redis is consistently fast**

   * Always RAM-based, no indexing overhead, negligible latency for both SET and GET.

2. **MySQL can look similar to Redis**

   * Only when: single-row query + primary key index + cached in memory.
   * That’s why your benchmark for `id=1` showed almost the same time as Redis.

3. **Realistic MySQL workloads are slower**

   * Random keys, ranges, or big datasets cause cache misses → disk reads → higher latency.
   * Throughput drops dramatically compared to Redis.

4. **Parallelism effect**

   * Both Redis and MySQL use concurrency (`-c 50` / `--concurrency=50`).
   * Redis handles parallel clients extremely efficiently; MySQL has locking and transactional overheads, especially with writes.


Perfect ✅ you’ve now successfully created the procedure and called it.

A few important points:

---

### 1️⃣ Inserting 1 million rows will take **a long time**

* MySQL is inserting **row by row** inside a loop → this is **slow** for 1M rows.
* On a typical laptop, it may take **several minutes or longer**.

---

### 2️⃣ Faster alternatives for large inserts

Instead of looping, you can do a **bulk insert using a single query**:

```sql
-- Example: insert 100,000 rows at once
INSERT INTO users (name)
SELECT CONCAT('user-', n)
FROM (
    SELECT a.N + b.N*10000 + 1 AS n
    FROM (SELECT 0 AS N UNION ALL SELECT 1 UNION ALL SELECT 2 ... UNION ALL SELECT 9999) a
    CROSS JOIN (SELECT 0 AS N UNION ALL SELECT 1 UNION ALL SELECT 2 ... UNION ALL SELECT 99) b
) nums
WHERE n <= 1000000;
```

> This avoids the row-by-row loop and is **much faster**.

---

### 3️⃣ After the 1M rows are inserted

You can now run **realistic benchmarks**:

**Random row SELECT:**

```bash
mysqlslap --user=keshav --password=keshav \
  --concurrency=50 --iterations=1 \
  --number-of-queries=100000 \
  --create-schema=sbtest \
  --query="SELECT name FROM users WHERE id=FLOOR(1 + RAND()*1000000);"
```

**Range query SELECT (many rows at once):**

```bash
mysqlslap --user=keshav --password=keshav \
  --concurrency=50 --iterations=1 \
  --number-of-queries=10000 \
  --create-schema=sbtest \
  --query="SELECT * FROM users WHERE id BETWEEN FLOOR(1 + RAND()*990000) AND FLOOR(1 + RAND()*990000 + 10000);"
```

* Random row → will now include **cache misses** for rows not in buffer pool.
* Range query → heavy workload, **slower than Redis**.

---

Do you want me to **guide you to run these benchmarks and then produce the Redis vs MySQL comparison graph** next?




mysql>
mysql> -- Insert 1 million rows
mysql> DELIMITER $$
mysql> CREATE PROCEDURE fill_users()
    -> BEGIN
    ->   DECLARE i INT DEFAULT 1;
    ->   WHILE i <= 1000000 DO
    ->     INSERT INTO users (name) VALUES (CONCAT('user-', i));
    ->     SET i = i + 1;
    ->   END WHILE;
    -> END$$
Query OK, 0 rows affected (0.01 sec)

mysql> DELIMITER ;
mysql>
mysql> CALL fill_users();

^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> mysqlslap --user=keshav --password=keshav \
-iterations=1 \
    ->   --concurrency=50 --iterations=1 \
    ->   --number-of-queries=100000 \
    ->   --create-schema=sbtest \
ery="SELECT name    ->   --query="SELECT name FROM users WHERE id=FLOOR(1 + RAND()*1000000);"
    ->
    -> exit;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'mysqlslap --user=keshav --password=keshav
  --concurrency=50 --iterations=1
  ' at line 1
mysql> exit;
Bye
keshav@Breakoutpickr:~$ mysqlslap --user=keshav --password=keshav \
  --concurrency=50 --iterations=1 \
  --number-of-queries=100000 \
  --create-schema=sbtest \
  --query="SELECT name FROM users WHERE id=FLOOR(1 + RAND()*1000000);"
mysqlslap: [Warning] Using a password on the command line interface can be insecure.
Benchmark
        Average number of seconds to run all queries: 41.187 seconds
        Minimum number of seconds to run all queries: 41.187 seconds
        Maximum number of seconds to run all queries: 41.187 seconds
        Number of clients running queries: 50
        Average number of queries per client: 2000

keshav@Breakoutpickr:~$ mysqlslap --user=keshav --password=keshav \
  --concurrency=50 --iterations=1 \
  --number-of-queries=10000 \
  --create-schema=sbtest \
  --query="SELECT * FROM users WHERE id BETWEEN FLOOR(1 + RAND()*990000) AND FLOOR(1 + RAND()*990000 + 10000);"
mysqlslap: [Warning] Using a password on the command line interface can be insecure.
Benchmark
        Average number of seconds to run all queries: 4.940 seconds
        Minimum number of seconds to run all queries: 4.940 seconds
        Maximum number of seconds to run all queries: 4.940 seconds
        Number of clients running queries: 50
        Average number of queries per client: 200




<img width="1615" height="683" alt="image" src="https://github.com/user-attachments/assets/75cfa836-9633-43d9-8f6d-c51a394a34d9" />

keshav@Breakoutpickr:~$ redis-cli --pipe

redis-benchmark -n 100000 -c 50 SET randomkey:value:1 1^C
keshav@Breakoutpickr:~$ redis-benchmark -n 100000 -c 50 SET randomkey:value:1 1
====== SET randomkey:value:1 1 ======
  100000 requests completed in 1.57 seconds
  50 parallel clients
  44 bytes payload
  keep alive: 1
  host configuration "save": 3600 1 300 100 60 10000
  host configuration "appendonly": no
  multi-thread: no

Latency by percentile distribution:
0.000% <= 0.119 milliseconds (cumulative count 1)
50.000% <= 0.391 milliseconds (cumulative count 50576)
75.000% <= 0.607 milliseconds (cumulative count 75448)
87.500% <= 0.927 milliseconds (cumulative count 87573)
93.750% <= 1.199 milliseconds (cumulative count 93822)
96.875% <= 1.423 milliseconds (cumulative count 96938)
98.438% <= 1.607 milliseconds (cumulative count 98468)
99.219% <= 1.767 milliseconds (cumulative count 99244)
99.609% <= 1.927 milliseconds (cumulative count 99614)
99.805% <= 2.111 milliseconds (cumulative count 99810)
99.902% <= 2.287 milliseconds (cumulative count 99903)
99.951% <= 7.831 milliseconds (cumulative count 99952)
99.976% <= 8.455 milliseconds (cumulative count 99976)
99.988% <= 8.967 milliseconds (cumulative count 99988)
99.994% <= 9.383 milliseconds (cumulative count 99994)
99.997% <= 9.615 milliseconds (cumulative count 99997)
99.998% <= 9.711 milliseconds (cumulative count 99999)
99.999% <= 9.767 milliseconds (cumulative count 100000)
100.000% <= 9.767 milliseconds (cumulative count 100000)

Cumulative distribution of latencies:
0.000% <= 0.103 milliseconds (cumulative count 0)
0.320% <= 0.207 milliseconds (cumulative count 320)
16.099% <= 0.303 milliseconds (cumulative count 16099)
53.980% <= 0.407 milliseconds (cumulative count 53980)
67.515% <= 0.503 milliseconds (cumulative count 67515)
75.448% <= 0.607 milliseconds (cumulative count 75448)
80.355% <= 0.703 milliseconds (cumulative count 80355)
84.171% <= 0.807 milliseconds (cumulative count 84171)
86.919% <= 0.903 milliseconds (cumulative count 86919)
89.810% <= 1.007 milliseconds (cumulative count 89810)
92.024% <= 1.103 milliseconds (cumulative count 92024)
93.961% <= 1.207 milliseconds (cumulative count 93961)
95.452% <= 1.303 milliseconds (cumulative count 95452)
96.738% <= 1.407 milliseconds (cumulative count 96738)
97.723% <= 1.503 milliseconds (cumulative count 97723)
98.468% <= 1.607 milliseconds (cumulative count 98468)
98.969% <= 1.703 milliseconds (cumulative count 98969)
99.357% <= 1.807 milliseconds (cumulative count 99357)
99.585% <= 1.903 milliseconds (cumulative count 99585)
99.719% <= 2.007 milliseconds (cumulative count 99719)
99.804% <= 2.103 milliseconds (cumulative count 99804)
99.950% <= 3.103 milliseconds (cumulative count 99950)
99.966% <= 8.103 milliseconds (cumulative count 99966)
99.990% <= 9.103 milliseconds (cumulative count 99990)
100.000% <= 10.103 milliseconds (cumulative count 100000)

Summary:
  throughput summary: 63572.79 requests per second
  latency summary (msec):
          avg       min       p50       p95       p99       max
        0.532     0.112     0.391     1.279     1.711     9.767
keshav@Breakoutpickr:~$ redis-benchmark -n 100000 -c 50 GET randomkey:value:1
====== GET randomkey:value:1 ======
  100000 requests completed in 1.54 seconds
  50 parallel clients
  37 bytes payload
  keep alive: 1
  host configuration "save": 3600 1 300 100 60 10000
  host configuration "appendonly": no
  multi-thread: no

Latency by percentile distribution:
0.000% <= 0.119 milliseconds (cumulative count 1)
50.000% <= 0.391 milliseconds (cumulative count 50580)
75.000% <= 0.567 milliseconds (cumulative count 75074)
87.500% <= 0.855 milliseconds (cumulative count 87504)
93.750% <= 1.119 milliseconds (cumulative count 93829)
96.875% <= 1.327 milliseconds (cumulative count 96912)
98.438% <= 1.511 milliseconds (cumulative count 98469)
99.219% <= 1.679 milliseconds (cumulative count 99247)
99.609% <= 1.839 milliseconds (cumulative count 99619)
99.805% <= 1.983 milliseconds (cumulative count 99807)
99.902% <= 2.143 milliseconds (cumulative count 99908)
99.951% <= 2.311 milliseconds (cumulative count 99953)
99.976% <= 2.487 milliseconds (cumulative count 99977)
99.988% <= 2.567 milliseconds (cumulative count 99989)
99.994% <= 2.671 milliseconds (cumulative count 99994)
99.997% <= 2.743 milliseconds (cumulative count 99997)
99.998% <= 2.767 milliseconds (cumulative count 99999)
99.999% <= 2.791 milliseconds (cumulative count 100000)
100.000% <= 2.791 milliseconds (cumulative count 100000)

Cumulative distribution of latencies:
0.000% <= 0.103 milliseconds (cumulative count 0)
0.318% <= 0.207 milliseconds (cumulative count 318)
15.082% <= 0.303 milliseconds (cumulative count 15082)
54.408% <= 0.407 milliseconds (cumulative count 54408)
69.390% <= 0.503 milliseconds (cumulative count 69390)
77.790% <= 0.607 milliseconds (cumulative count 77790)
82.542% <= 0.703 milliseconds (cumulative count 82542)
86.115% <= 0.807 milliseconds (cumulative count 86115)
88.809% <= 0.903 milliseconds (cumulative count 88809)
91.410% <= 1.007 milliseconds (cumulative count 91410)
93.502% <= 1.103 milliseconds (cumulative count 93502)
95.386% <= 1.207 milliseconds (cumulative count 95386)
96.634% <= 1.303 milliseconds (cumulative count 96634)
97.688% <= 1.407 milliseconds (cumulative count 97688)
98.428% <= 1.503 milliseconds (cumulative count 98428)
98.984% <= 1.607 milliseconds (cumulative count 98984)
99.310% <= 1.703 milliseconds (cumulative count 99310)
99.561% <= 1.807 milliseconds (cumulative count 99561)
99.718% <= 1.903 milliseconds (cumulative count 99718)
99.823% <= 2.007 milliseconds (cumulative count 99823)
99.883% <= 2.103 milliseconds (cumulative count 99883)
100.000% <= 3.103 milliseconds (cumulative count 100000)

Summary:
  throughput summary: 64808.82 requests per second
  latency summary (msec):
          avg       min       p50       p95       p99       max
        0.510     0.112     0.391     1.183     1.615     2.791






