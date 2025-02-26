# Hadoop-Based Web Log Analysis

## Overview
This project focuses on analyzing web server logs using Hadoop and Hive. It processes large-scale log data, partitions it for efficient querying, and extracts insights such as request frequency, error distributions, and user behavior trends. The main objectives include:
- Storing and processing web logs in HDFS.
- Using Hive for structured querying and partitioning.
- Performing analytics on web traffic patterns and error occurrences.

## Implementation Strategy
### **Mapper and Reducer Functions**
- **Mapper:** Parses log entries to extract key details like IP addresses, timestamps, URLs, status codes, and user agents.
- **Reducer:** Aggregates data to compute insights such as most visited pages, common HTTP errors, and user agent statistics.
- **Partitioning:** Data is partitioned based on HTTP status codes to enhance query performance.

## Execution Workflow
### **Setting Up Docker and Copying Files**
```sh
docker cp web_server_logs.csv resourcemanager:/opt/hadoop-3.2.1/share/hadoop/mapreduce/
docker cp web_server_logs.csv namenode:/tmp/
```

### **Uploading Data to HDFS**
```sh
docker exec -it namenode /bin/bash
hdfs dfs -mkdir -p /user/hive/warehouse/web_logs
hdfs dfs -put /tmp/web_server_logs.csv /user/hive/warehouse/web_logs/
```

### **Creating Hive Tables**
```sql
CREATE TABLE web_server_logs_partitioned (
    ip STRING,
    timestamp_ STRING,
    url STRING,
    user_agent STRING
) PARTITIONED BY (status INT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;
```

### **Enabling Dynamic Partitioning**
```sql
SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;
```

### **Inserting Data into Partitioned Table**
```sql
INSERT INTO TABLE web_server_logs_partitioned PARTITION (status)
SELECT ip, timestamp_, url, user_agent, status FROM web_server_logs;
```

### **Checking Table Structure**
```sql
DESCRIBE web_server_logs;
```

### **Exporting Processed Data**
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output/web_logs_analysis' 
SELECT * FROM web_server_logs;
```

### **Defining External Table for Logs**
```sql
CREATE DATABASE web_logs;
USE web_logs;

CREATE EXTERNAL TABLE web_server_logs (
    ip STRING,
    timestamp_ STRING,
    url STRING,
    status INT,
    user_agent STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/warehouse/web_logs/';
```

### **Loading Data into Hive Table**
```sql
LOAD DATA INPATH '/user/hive/warehouse/web_logs/web_server_logs.csv' INTO TABLE web_server_logs;
```

### **Analyzing Web Log Data**
#### **1. Total Request Count**
```sql
SELECT COUNT(*) AS total_requests FROM web_server_logs;
```
#### **2. Distribution of HTTP Status Codes**
```sql
SELECT status, COUNT(*) AS count FROM web_server_logs GROUP BY status;
```
#### **3. Most Popular Pages**
```sql
SELECT url, COUNT(*) AS visits FROM web_server_logs GROUP BY url ORDER BY visits DESC LIMIT 3;
```
#### **4. Most Common User Agents**
```sql
SELECT user_agent, COUNT(*) AS count FROM web_server_logs GROUP BY user_agent ORDER BY count DESC;
```
#### **5. IPs with Multiple Failed Requests**
```sql
SELECT ip, COUNT(*) AS failed_requests 
FROM web_server_logs 
WHERE status IN (404, 500) 
GROUP BY ip 
HAVING COUNT(*) > 3;
```
#### **6. Request Volume Over Time**
```sql
SELECT SUBSTR(timestamp_, 1, 16) AS time_slot, COUNT(*) AS request_count 
FROM web_server_logs 
GROUP BY SUBSTR(timestamp_, 1, 16) 
ORDER BY time_slot;
```

### **Storing Results in HDFS**
```sql
INSERT OVERWRITE DIRECTORY '/user/hadoop/web_logs/output'
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
SELECT * FROM web_server_logs;
```

### **Checking Output in HDFS**
```sh
hdfs dfs -ls /user/hadoop/web_logs/output
exit
```

### **Transferring Output to Local System**
```sh
docker cp namenode:/opt/output output/output
```

## Challenges & Solutions
### **1. CSV Parsing Issues**
- **Problem:** Incorrect field delimiters caused errors while loading data.
- **Solution:** Used `FIELDS TERMINATED BY ','` in Hive table schema to ensure proper parsing.

### **2. Dynamic Partitioning Errors**
- **Problem:** Insert queries into partitioned tables failed initially.
- **Solution:** Enabled dynamic partitioning using:
  ```sql
  SET hive.exec.dynamic.partition = true;
  SET hive.exec.dynamic.partition.mode = nonstrict;
  ```

### **3. Copying HDFS Data to Host System**
- **Problem:** `docker cp` failed since data was stored in HDFS, not `/opt/output`.
- **Solution:** Used:
  ```sh
  hdfs dfs -copyToLocal /user/hadoop/web_logs/output /opt/output
  ```
  before running `docker cp`.

## Example Input and Output
### **Sample Log File (CSV Format)**
```
192.168.0.1,2025-02-25 13:00:15,/index,200,Firefox/98.0
192.168.0.2,2025-02-25 13:01:20,/contact,500,Edge/99.0
192.168.0.3,2025-02-25 13:02:05,/blog,404,Safari/15.3
```

### **Sample Query Result (Most Visited Pages)**
```
/index     |  1800
/contact   |  950
/blog      |  800
```

## Conclusion
This project effectively utilizes Hadoop and Hive for large-scale log analysis. By structuring log data, partitioning tables, and executing analytical queries, we gain valuable insights into web traffic trends and system performance. The approach ensures efficient data processing and scalable analysis for massive log datasets.

---
