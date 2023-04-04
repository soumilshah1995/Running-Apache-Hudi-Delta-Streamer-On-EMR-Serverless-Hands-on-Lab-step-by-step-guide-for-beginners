Running Apache Hudi Delta Streamer On EMR Serverless Hands on Lab step by step guide for beginners

![1](https://user-images.githubusercontent.com/39345855/229940404-f3efeaae-6e5b-446b-a229-b1fb86e4ea2b.JPG)

## Video based guide 


# Steps 
## Step 1: Download the sample Parquet files from the links 
* https://drive.google.com/drive/folders/1BwNEK649hErbsWcYLZhqCWnaXFX3mIsg?usp=share_link
#### Uplaod to S3 Folder as shown in diagram 
![image](https://user-images.githubusercontent.com/39345855/229939875-6f2f22ae-c792-4904-bcf8-b1e53ce1e122.png)



## Step 2: Start EMR Serverless Cluster 
![image](https://user-images.githubusercontent.com/39345855/229940052-29f6e2a8-9568-4100-8a1b-e988c405f505.png)
![image](https://user-images.githubusercontent.com/39345855/229940099-cf002f04-18f8-4d26-8d89-d512e96bef76.png)
![image](https://user-images.githubusercontent.com/39345855/229940131-836414cf-a85f-4b9f-b1d6-c36115d335c2.png)

# Step 3 Run Python Code to submit Job 
* Please change nd edit the varibales 

```
try:
    import json
    import uuid
    import os
    import boto3
    from dotenv import load_dotenv

    load_dotenv(".env")
except Exception as e:
    pass

global AWS_ACCESS_KEY
global AWS_SECRET_KEY
global AWS_REGION_NAME

AWS_ACCESS_KEY = os.getenv("DEV_ACCESS_KEY")
AWS_SECRET_KEY = os.getenv("DEV_SECRET_KEY")
AWS_REGION_NAME = os.getenv("DEV_REGION")

client = boto3.client("emr-serverless",
                      aws_access_key_id=AWS_ACCESS_KEY,
                      aws_secret_access_key=AWS_SECRET_KEY,
                      region_name=AWS_REGION_NAME)


def lambda_handler_test_emr(event, context):
    # ------------------Hudi settings ---------------------------------------------
    glue_db = "hudi_db"
    table_name = "invoice"
    op = "UPSERT"
    table_type = "COPY_ON_WRITE"

    record_key = 'invoiceid'
    precombine = "replicadmstimestamp"
    partition_feild = 'destinationstate'
    source_ordering_field = 'replicadmstimestamp'

    delta_streamer_source = 's3://XXXXXXXXXXXX/raw'
    hudi_target_path = 's3://XXXXXXXXX/hudi'

    # ---------------------------------------------------------------------------------
    #                                       EMR
    # --------------------------------------------------------------------------------
    ApplicationId = "XXXXXXXXXXXXXXX"
    ExecutionTime = 600
    ExecutionArn = "XXXXXXXXXXXXXXXXXXXXXX"
    JobName = 'delta_streamer_{}'.format(table_name)

    # --------------------------------------------------------------------------------

    spark_submit_parameters = ' --conf spark.jars=/usr/lib/hudi/hudi-utilities-bundle.jar'
    spark_submit_parameters += ' --conf spark.serializer=org.apache.spark.serializer.KryoSerializer'
    spark_submit_parameters += ' --conf spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension'
    spark_submit_parameters += ' --conf spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog'
    spark_submit_parameters += ' --conf spark.sql.hive.convertMetastoreParquet=false'
    spark_submit_parameters += ' --conf mapreduce.fileoutputcommitter.marksuccessfuljobs=false'
    spark_submit_parameters += ' --conf spark.hadoop.hive.metastore.client.factory.class=com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory'
    spark_submit_parameters += ' --class org.apache.hudi.utilities.deltastreamer.HoodieDeltaStreamer'

    arguments = [
        "--table-type", table_type,
        "--op", op,
        "--enable-sync",
        "--source-ordering-field", source_ordering_field,
        "--source-class", "org.apache.hudi.utilities.sources.ParquetDFSSource",
        "--target-table", table_name,
        "--target-base-path", hudi_target_path,
        "--payload-class", "org.apache.hudi.common.model.AWSDmsAvroPayload",
        "--hoodie-conf", "hoodie.datasource.write.keygenerator.class=org.apache.hudi.keygen.SimpleKeyGenerator",
        "--hoodie-conf", "hoodie.datasource.write.recordkey.field={}".format(record_key),
        "--hoodie-conf", "hoodie.datasource.write.partitionpath.field={}".format(partition_feild),
        "--hoodie-conf", "hoodie.deltastreamer.source.dfs.root={}".format(delta_streamer_source),
        "--hoodie-conf", "hoodie.datasource.write.precombine.field={}".format(precombine),
        "--hoodie-conf", "hoodie.database.name={}".format(glue_db),
        "--hoodie-conf", "hoodie.datasource.hive_sync.enable=true",
        "--hoodie-conf", "hoodie.datasource.hive_sync.table={}".format(table_name),
        "--hoodie-conf", "hoodie.datasource.hive_sync.partition_fields={}".format(partition_feild),
    ]

    response = client.start_job_run(
        applicationId=ApplicationId,
        clientToken=uuid.uuid4().__str__(),
        executionRoleArn=ExecutionArn,
        jobDriver={
            'sparkSubmit': {
                'entryPoint': "command-runner.jar",
                'entryPointArguments': arguments,
                'sparkSubmitParameters': spark_submit_parameters
            },
        },
        executionTimeoutMinutes=ExecutionTime,
        name=JobName,
    )
    print("response", end="\n")
    print(response)


lambda_handler_test_emr(context=None, event=None)

```

### Adhoc Query 
![image](https://user-images.githubusercontent.com/39345855/229940357-49106e3a-2d5c-4f53-9ce2-9f45759d9dea.png)


