# Hadoop on GCP with Python
[GCP Hadoop Dataproc Example](https://github.com/GoogleCloudDataproc/initialization-actions/tree/master/cloud-sql-proxy)

### Setup environment with Dataproc
- set variables
```
export REGION=us-central1
export ZONE=us-central1-a
gcloud config set compute/zone $ZONE
export PROJECT=$(gcloud info --format='value(config.project)')
export HIVE_DATA_BUCKET=${PROJECT}-warehouse5
export INSTANCE_NAME=hive-metastore5
export CLUSTER_NAME=hive-cluster5
gcloud services enable dataproc.googleapis.com sqladmin.googleapis.com 
```

- make bucket
```
gsutil mb -l ${REGION} gs://${HIVE_DATA_BUCKET}
```
- make database
```
gcloud sql instances create ${INSTANCE_NAME} \
    --tier db-n1-standard-1 \
    --activation-policy=ALWAYS \
    --region $REGION
```

- make cluster 
```
gcloud dataproc clusters create ${CLUSTER_NAME} \
    --region ${REGION} \
    --scopes sql-admin \
    --initialization-actions gs://goog-dataproc-initialization-actions-${REGION}/cloud-sql-proxy/cloud-sql-proxy.sh \
    --properties hive:hive.metastore.warehouse.dir=gs://${HIVE_DATA_BUCKET}/hive-warehouse \
    --metadata "hive-metastore-instance=${PROJECT}:${REGION}:${INSTANCE_NAME}" \
    --master-boot-disk-size 1TB \
    --region $REGION 
```

### Test Dataproc

- copy pyspark file to bucket to test 
```
gsutil cp gs://goog-dataproc-initialization-actions-${REGION}/cloud-sql-proxy/pyspark_metastore_test.py gs://${HIVE_DATA_BUCKET}/pyspark_metastore_test.py 

```
- submit pyspark to dataproc for query to test
```
gcloud dataproc jobs submit pyspark --cluster $CLUSTER_NAME gs://${HIVE_DATA_BUCKET}/pyspark_metastore_test.py --region $REGION
```

### Demo Transactions
- copy file for transactions
```
gsutil cp gs://hive-solution/part-00000.parquet \
gs://${HIVE_DATA_BUCKET}/datasets/transactions/part-00000.parquet 
```

- submit hive job create table
```
gcloud dataproc jobs submit hive \
  --cluster ${CLUSTER_NAME} \
  --execute \
  "CREATE EXTERNAL TABLE transactions2(SubmissionDate DATE, TransactionType STRING, TransactionAmount DOUBLE) STORED AS PARQUET LOCATION 'gs://${HIVE_DATA_BUCKET}/datasets/transactions';" \
  --region $REGION 
```

- submit hive job query table
```
gcloud dataproc jobs submit hive \
--cluster $CLUSTER_NAME \
--execute \
"SELECT * FROM transactions LIMIT 10;" \
--region $REGION 
```

- query table from python
  - ssh into master node in cluster
```
gcloud compute ssh ${INSTANCE_NAME}-m 
(or use ssh within console)
```
- initialize pyspark
```
pyspark
```

- query from pyspark on master node
```
from pyspark.sql import HiveContext
hc = HiveContext(sc)
hc.sql("""
  SELECT SubmissionDate, AVG(TransactionAmount) as AvgDebit
  FROM transactions2
  WHERE TransactionType = 'debit'
  GROUP BY SubmissionDate
  HAVING SubmissionDate >= '2017-10-01' AND SubmissionDate < '2017-10-06'
  ORDER BY SubmissionDate
""").show() 
```

- query from beeline on master node
  - start another ssh session or logout / login
```
beeline -u "jdbc:hive2://localhost:10000"
```
- run normal SQL query
```
SELECT * 
FROM transactions2 
LIMIT 10;
```

- connect to mysql to check metastore
  - blank password (hit enter)
```
gcloud sql connect $INSTANCE_NAME --user=root
USE hive_metastore;
SELECT DB_LOCATION_URI FROM DBS;
SELECT TBL_NAME, TBL_TYPE FROM TBLS;
SELECT COLUMN_NAME, TYPE_NAME
FROM COLUMNS_V2 c, TBLS t
WHERE c.CD_ID = t.SD_ID
AND t.TBL_NAME = 'transactions2';
SELECT INPUT_FORMAT, LOCATION
FROM SDS s, TBLS t
WHERE s.SD_ID = t.SD_ID
AND t.TBL_NAME = 'transactions2';
```

### Log into master node to run scripts
```
hdfs dfs -ls /
hdfs dfs -ls -R /user

``


