# Batch processing
The pipelined we decided to use was sqoop for the measurements table and spark/jdbc for the smaller tables. Sqooping the measurements table was necessary for parallel ingestion of 500m records. Spark, through the sqlContext read method, does not ingest in parallel but for smaller tables this was satisfactory. This gain us the ability to sqoop only once and immediately join the tables in spark. 

# Spark applications
We implemented 2 versions of the spark application; python and scala. Both apps have exactly the same logic. We ran into issues with long running times. Broadcasting the join tables greatly improved the exec time to 15-20 min from small table ingestion, transformation of datatypes to joining of the all 4 tables.

* [Python](batch.py)
* Scala see spark project [here](../spark). Scala code [here](../spark/src/main/scala/com/cloudera/bootcamp/Main.scala)


## implementation 
scala 
```
val flagged = measurements.withColumn("flag", 
  measurements("amplitude_1") > 0.995 && 
  measurements("amplitude_3") > 0.995 && 
  measurements("amplitude_2") < 0.005)
val joined = flagged.
    join(org.apache.spark.sql.functions.broadcast(galaxies), Seq("galaxy_id")).
    join(org.apache.spark.sql.functions.broadcast(detectors), Seq("detector_id")).
    join(org.apache.spark.sql.functions.broadcast(astrophysicists), Seq("astrophysicist_id"))
```

python
```
# Add Flag
flagged=dfs[0].withColumn('flag', (dfs[0].amplitude_1 > 0.995) & (dfs[0].amplitude_3 > 0.995) & (dfs[0].amplitude_2 < 0.005) )

# Join Tables
joined_table = flagged.join(broadcast(dfs[1]), ["galaxy_id"]).join(broadcast(dfs[2]), ["detector_id"]).join(broadcast(dfs[3]),["astrophysicist_id"])

```

Knowing python is advantageous in engagements where there is NOT a way to compile scala code on an edgenode.

# OOZIE
we created an OOZIE flow that called the sqoop ingestion below then the python script above. Total execution time was around 30 min.

2 sets of artifacts:
* [ruslan](oozie)
* [arunssundar](oozie2/workflow.xml)

### Notes
PFB the directory structure of the workflow.xml.

  --> gwProcess/workflow.xml

  --> sqoopIngest/workflow.xml

  --> sqoopIngest/lib/hive-site.xml
  
  --> sqoopIngest/lib/ojdbc6.jar


The workflow.xml is the main workflow and it needs to be put under the gwProcess directory. 
The workflow1.xml is the subworkflow which needs to be put under sqoopIngest directory.


# sqoop
```
sqoop import \
  --num-mappers 9 \
  --connect "jdbc:oracle:thin:@gravity.cghfmcr8k3ia.us-west-2.rds.amazonaws.com:15210:gravity" \
  --username=gravity \
  --password=bootcamp \
  --direct \
  --fetch-size 10000 \
  --table MEASUREMENTS \
  --hive-import \
  --create-hive-table \
  --hive-database gravity2 \
  --hive-overwrite \
  --hive-table measurements_raw
```

