# Kylin

[TOC]

## 1. Cube Build

MR引擎的主入口是`BatchCubingJobBuilder2.java`

Spark引擎的主入口是`SparkBatchCubingJobBuilder2.java`


```
graph TB
A[Create Intermediate Flat Hive Table / addStepPhase1_CreateFlatTable]-->B[Extract Fact Table Distinct Columns / FactDistinctColumnsJob.java]
B-->C[Build Dimension Dictionary / CreateDictionaryJob.java]
C-->D[Save Cuboid Statistics / SaveStatisticsStep.java]
D-->E[Create HTable / HBaseJobSteps.java]
E-->F[Build Base Cuboid / BaseCuboidJob.java]
F-->G[Build N-Dimension Cuboid / NDCuboidJob.java]
G-->|循环N_dimension - 1次|G
G-->H[Build Cube In-Mem / InMemCuboidJob.java]
H-->I[Convert Cuboid Data to HFile / HBaseJobSteps.java]
I-->J[Load HFile to HBase Table / HBaseJobSteps.java]
J-->K[Update Cube Info / UpdateCubeInfoAfterBuildStep.jaa]
K-->L[Hive Cleanup / GarbageCollectionStep.java]
L-->M[Garbage Collection on HBase / HDFSPathGarbageCollectionStep.java]
```
## 2. Create Intermediate Flat Hive Table
```sql
DROP TABLE IF EXISTS kylin_intermediate_test_cube_f1ad98f8_2b43_5fb5_0512_5ae342531f3f;

CREATE EXTERNAL TABLE IF NOT EXISTS kylin_intermediate_test_cube_f1ad98f8_2b43_5fb5_0512_5ae342531f3f
(
KYLIN_SALES_TRANS_ID bigint
,KYLIN_SALES_PART_DT date
,KYLIN_SALES_LEAF_CATEG_ID bigint
,KYLIN_SALES_BUYER_ID bigint
,KYLIN_SALES_OPS_REGION string
,KYLIN_CATEGORY_GROUPINGS_LEAF_CATEG_ID bigint
,KYLIN_CAL_DT_CAL_DT date
,KYLIN_ACCOUNT_ACCOUNT_ID bigint
,KYLIN_ACCOUNT_ACCOUNT_COUNTRY string
,KYLIN_COUNTRY_COUNTRY string
,KYLIN_SALES_PRICE decimal(19,4)
,KYLIN_SALES_ITEM_COUNT bigint
)
STORED AS SEQUENCEFILE
LOCATION 'hdfs://quickstart.cloudera:8020/kylin/kylin_metadata/kylin-8424937c-cd17-6b27-7906-cb2aa4540f58/kylin_intermediate_test_cube_f1ad98f8_2b43_5fb5_0512_5ae342531f3f';
ALTER TABLE kylin_intermediate_test_cube_f1ad98f8_2b43_5fb5_0512_5ae342531f3f SET TBLPROPERTIES('auto.purge'='true');

INSERT OVERWRITE TABLE kylin_intermediate_test_cube_f1ad98f8_2b43_5fb5_0512_5ae342531f3f SELECT
KYLIN_SALES.TRANS_ID as KYLIN_SALES_TRANS_ID
,KYLIN_SALES.PART_DT as KYLIN_SALES_PART_DT
,KYLIN_SALES.LEAF_CATEG_ID as KYLIN_SALES_LEAF_CATEG_ID
,KYLIN_SALES.BUYER_ID as KYLIN_SALES_BUYER_ID
,KYLIN_SALES.OPS_REGION as KYLIN_SALES_OPS_REGION
,KYLIN_CATEGORY_GROUPINGS.LEAF_CATEG_ID as KYLIN_CATEGORY_GROUPINGS_LEAF_CATEG_ID
,KYLIN_CAL_DT.CAL_DT as KYLIN_CAL_DT_CAL_DT
,KYLIN_ACCOUNT.ACCOUNT_ID as KYLIN_ACCOUNT_ACCOUNT_ID
,KYLIN_ACCOUNT.ACCOUNT_COUNTRY as KYLIN_ACCOUNT_ACCOUNT_COUNTRY
,KYLIN_COUNTRY.COUNTRY as KYLIN_COUNTRY_COUNTRY
,KYLIN_SALES.PRICE as KYLIN_SALES_PRICE
,KYLIN_SALES.ITEM_COUNT as KYLIN_SALES_ITEM_COUNT
FROM KYLIN.KYLIN_SALES as KYLIN_SALES 
INNER JOIN KYLIN.KYLIN_CATEGORY_GROUPINGS as KYLIN_CATEGORY_GROUPINGS
ON KYLIN_SALES.LEAF_CATEG_ID = KYLIN_CATEGORY_GROUPINGS.LEAF_CATEG_ID
INNER JOIN KYLIN.KYLIN_CAL_DT as KYLIN_CAL_DT
ON KYLIN_SALES.PART_DT = KYLIN_CAL_DT.CAL_DT
INNER JOIN KYLIN.KYLIN_ACCOUNT as KYLIN_ACCOUNT
ON KYLIN_SALES.BUYER_ID = KYLIN_ACCOUNT.ACCOUNT_ID
INNER JOIN KYLIN.KYLIN_COUNTRY as KYLIN_COUNTRY
ON KYLIN_ACCOUNT.ACCOUNT_COUNTRY = KYLIN_COUNTRY.COUNTRY
WHERE 1=1;

```

## 3. Extract Fact Table Distinct Columns
```shell
-conf /Users/sean/workspace/kylin/server/../examples/test_case_data/sandbox/kylin_job_conf.xml \
-cubename test_cube \
-output hdfs://quickstart.cloudera:8020/kylin/kylin_metadata/kylin-96f63284-986e-ae48-32f9-ab4f28e2d447/test_cube/fact_distinct_columns \
-segmentid 7cba9299-dfa3-15dc-5762-5e7944acb5d8 \
-statisticsoutput hdfs://quickstart.cloudera:8020/kylin/kylin_metadata/kylin-96f63284-986e-ae48-32f9-ab4f28e2d447/test_cube/fact_distinct_columns/statistics \
-statisticssamplingpercent 100 \
-jobname Kylin_Fact_Distinct_Columns_test_cube_Step \
-cubingJobId 96f63284-986e-ae48-32f9-ab4f28e2d447
```