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
