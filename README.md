# 🚀 Practical 3: AWS RDS + Redis (ElastiCache) Side Cache Implementation

---

## 📌 Objective

To reduce database load and improve response time by implementing a **Side Cache (Lazy Loading) Pattern** using:

* Amazon RDS (MySQL)
* Amazon ElastiCache (Redis)

---

## 🏗️ Step-by-Step Implementation

### 1️⃣ Created VPC

* Created custom VPC
* CIDR: `10.0.0.0/16`
  
<img width="1915" height="927" alt="Screenshot 2026-03-27 225409" src="https://github.com/user-attachments/assets/f628f449-070c-44c5-b91b-4d39c45f8a01" />

---

### 2️⃣ Created Subnets

* Public Subnet → for EC2
  <img width="1903" height="897" alt="Screenshot 2026-03-27 230139" src="https://github.com/user-attachments/assets/0916115d-c807-4efd-967c-ffd8a892710f" />

  <img width="1918" height="910" alt="Screenshot 2026-03-27 230311" src="https://github.com/user-attachments/assets/b25dac97-54f7-4fdd-8e4e-3251a30f3ca6" />

* Private Subnet 1 → for RDS
* Private Subnet 2 → for Redis
* 
<img width="1917" height="890" alt="Screenshot 2026-03-27 230922" src="https://github.com/user-attachments/assets/f69951f9-a8e8-472e-a6b6-d350af2a50d8" />

---

### 3️⃣ Configured Internet Gateway

* Created Internet Gateway
* Attached to VPC
* Added route:
  
<img width="1919" height="916" alt="Screenshot 2026-03-27 234138" src="https://github.com/user-attachments/assets/4da01ee8-b741-4d40-81fe-4cea544e0bfe" />


```
0.0.0.0/0 → Internet Gateway
```

<img width="1918" height="914" alt="Screenshot 2026-03-28 000338" src="https://github.com/user-attachments/assets/79d7801d-5989-4a17-9fe9-f850979126e3" />


---

### 4️⃣ Created NAT Gateway

* Created NAT Gateway in Public Subnet
* Attached Elastic IP
  
<img width="1919" height="918" alt="Screenshot 2026-03-27 231827" src="https://github.com/user-attachments/assets/6298cbeb-4e0d-43af-bbfd-481789d12034" />

  
* Updated Private Route Table:

```
0.0.0.0/0 → NAT Gateway
```

<img width="1917" height="924" alt="Screenshot 2026-03-27 232713" src="https://github.com/user-attachments/assets/6f89768b-eaf3-43e6-bdc1-eb28dfb24246" />


---

### 5️⃣ Created RDS (MySQL)

* Engine: MySQL
* Instance: db.t3.micro
* Public Access: NO
* Subnet Group: Private subnets
* Encryption: Enabled
  
<img width="1914" height="917" alt="Screenshot 2026-03-28 000633" src="https://github.com/user-attachments/assets/96bfdcc2-c67e-4d20-858d-611c947c43bf" />

<img width="1883" height="627" alt="Screenshot 2026-03-28 001853" src="https://github.com/user-attachments/assets/28d606d5-da8c-4221-a756-0d9767115c39" />

  

---

### 6️⃣ Created Redis (ElastiCache)

* Engine: Redis
* Node type: cache.t3.micro
* Subnet Group: Private subnets
* Encryption: Enabled
  
<img width="1919" height="904" alt="Screenshot 2026-03-28 121253" src="https://github.com/user-attachments/assets/e3a9853d-277f-4e10-b170-b7ca2449a089" />

<img width="1919" height="913" alt="Screenshot 2026-03-28 121305" src="https://github.com/user-attachments/assets/ef7f5ac8-a45e-41b0-aab5-7db305769ffd" />

---

### 7️⃣ Configured Security Groups

* App Server SG → attached to EC2
  
<img width="1917" height="922" alt="Screenshot 2026-03-28 002824" src="https://github.com/user-attachments/assets/fa7438c9-a652-4239-af24-717ebce34f3f" />

  
* RDS SG:

  * Port 3306 → Source: App SG
* Redis SG:

  <img width="1919" height="896" alt="Screenshot 2026-03-28 122839" src="https://github.com/user-attachments/assets/416ef4d2-1d3a-459f-975d-25ac06552393" />


  * Port 6379 → Source: App SG

<img width="1916" height="847" alt="Screenshot 2026-03-28 122940" src="https://github.com/user-attachments/assets/b0c802e1-a9a1-483c-af24-1421c96323ce" />

---

### 8️⃣ Launched EC2 Instance

* OS: Amazon Linux 2023
* Placed in Public Subnet
* Installed:

  <img width="1223" height="265" alt="2" src="https://github.com/user-attachments/assets/3a172576-4b54-4ed3-bd30-3ff11562082d" />




```
Node.js
mariadb (mysql client)
redis-cli
```



---

### 9️⃣ Tested Connectivity

#### RDS Connection

```
mysql -h <RDS-ENDPOINT> -u admin -p
```

CREATE DATABASE testdb;
USE testdb;

CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    price INT,
    description TEXT
);

INSERT INTO products VALUES 
(1, 'Laptop', 50000, 'Gaming laptop'),
(2, 'Phone', 20000, 'Smartphone'),
(3, 'Headphones', 3000, 'Wireless headphones');

<img width="1920" height="1020" alt="14" src="https://github.com/user-attachments/assets/afec5dd7-2263-4554-aca9-30bb93d45955" />

#### Redis Connection

```
redis-cli -h <REDIS-ENDPOINT> -p 6379
PING
```

<img width="861" height="92" alt="image" src="https://github.com/user-attachments/assets/b644d35f-d3a8-49eb-a1ad-b2e149f8ac40" />


---

### 🔟 Implemented Lazy Loading (Side Cache)

Workflow:

1. Check Redis for data
2. If found → return (Cache HIT ⚡)
3. If not → fetch from RDS (Cache MISS 🐢)
4. Store data in Redis
5. Return response

import redis
import pymysql
import json

cache = redis.Redis(
    host='my-cache-ok9oou.serverless.aps1.cache.amazonaws.com',
    port=6379,
    ssl=True,
    decode_responses=True
)

db = pymysql.connect(
    host='database-1-prctical-3.cni6c2y2yq5z.ap-south-1.rds.amazonaws.com',
    user='admin',
    password='my-password',
    database='testdb'
)

def get_product(product_id):
    key = f"product:{product_id}"

    cached = cache.get(key)
    if cached:
        print("Cache Hit")
        return json.loads(cached)

    print("Cache Miss")

    cursor = db.cursor()
    cursor.execute("SELECT * FROM products WHERE id=%s", (product_id,))
    row = cursor.fetchone()

    product = {
        "id": row[0],
        "name": row[1],
        "price": row[2],
        "description": row[3]
    }

    cache.setex(key, 60, json.dumps(product))

    return product

print(get_product(1))
print(get_product(1))

---

### 1️⃣1️⃣ TTL Implementation

* Cache expiry set to 60 seconds
* Prevents stale data

<img width="843" height="81" alt="image" src="https://github.com/user-attachments/assets/e7b90c8f-3050-4063-886f-578eb1906a93" />

---

### 1️⃣2️⃣ Cache Invalidation

* On update:

```
redisClient.del(key)
```

---

## 🧪 Testing

* First request → Cache MISS
* Second request → Cache HIT
* Observed faster response time

  <img width="840" height="297" alt="image" src="https://github.com/user-attachments/assets/f6213613-58cd-4ff1-9e8f-a6cf13181dbd" />


---

## 🔐 Security

* RDS & Redis are in private subnets
* No public access
* Only EC2 can access them

---

## 🎯 Result

* Reduced database load
* Faster response time
* Efficient caching using Redis

---

## 📚 Conclusion

The Side Cache pattern using Redis significantly improves performance by reducing repeated database queries and enabling faster data retrieval.

---

### Author : Suyash Dahitule
