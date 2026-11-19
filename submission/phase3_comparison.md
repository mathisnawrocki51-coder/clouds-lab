# Phase 3 Comparison Writeup

Mathis Nawrocki Lachapelle. EURECOM, Cloud Computing, Phase 3.

## 1. Facts from my runs

All the numbers come from submission/phase3_report.json.

Map tasks in the cloud job. <!--answer:num_map_tasks-->12<!--/answer-->

Reduce tasks in the cloud job. <!--answer:num_reduce_tasks-->4<!--/answer-->

Cloud MapReduce wall time in seconds. <!--answer:cloud_elapsed_s-->48.0<!--/answer-->

Local single process wall time in seconds. <!--answer:local_elapsed_s-->0.019<!--/answer-->

Spark wall time for the collect in seconds. <!--answer:spark_elapsed_s-->8.4<!--/answer-->

Number of stages in the Spark job. <!--answer:num_stages-->4<!--/answer-->

Task retries in my chaos run. <!--answer:chaos_retries-->15<!--/answer-->

### Explain the timings

The three wall times cover the same 60 documents. The single process run and the cloud MapReduce do the same word count. Spark computes TF IDF, which is more work. Rank the three times and explain what each one spends its time on. Say which comparison is like for like. Say what the Spark number tells you about where the cloud job time goes. (about 80 words)

<!--answer:timing_explanation-->
The three wall times rank in this order. The local run is the fastest with 0.019 s. Spark is second with 8.4 s. The cloud MapReduce is the slowest with 48.0 s. The like for like pair is the local run and the cloud job, because both do the same word count. The local run only counts words in one Python process. The cloud job spends almost all its time on overhead, with 16 HTTP calls, many small files in Cloud Storage and 15 retries. Spark does more work on one machine and it takes only 8.4 s. This tell me that the cloud time is mostly network and scheduling, not computation.
<!--/answer-->

### What the counts tell you

How many shuffles did the Spark job have, and which operations caused them? What fraction of the 16 task attempts was retried, and is that what chaos at 0.3 should produce? (about 50 words)

<!--answer:counts_interpretation-->
From my stage count, the Spark job had 4 stages, so it had 3 shuffles. They come from the reduceByKey in term_freq, the reduceByKey in doc_freq and the join. From my retry count, 15 attempts were retried for 16 tasks. With chaos at 0.3 we expect about 7 retries, so less than the half. My number is two times higher. All the 503 errors came from the simulated crash, so I was just unlucky.
<!--/answer-->

## 2. Spark transformations, actions and lineage

### Transformations vs actions

Name each RDD operation and say if it is a transformation or an action. What happened when you ran build_tfidf_rdd? What happened when you ran collect? (about 60 words)

<!--answer:transformations_vs_actions-->
In spark_tfidf.py and the notebook, flatMap, reduceByKey, map and join are transformations. The parallelize call only creates the first RDD. The only action is collect. When I ran build_tfidf_rdd, nothing was computed. Spark only recorded the lineage and the cell returned at once. When I ran collect, Spark split the lineage into 4 stages and ran 12 tasks in 8.4 s.
<!--/answer-->

### Lineage and recovery

The worker that holds one output partition of the first reduceByKey dies. What would Spark have to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery starts. (about 50 words)

<!--answer:lineage_recovery-->
If the worker with one output partition of the first reduceByKey died, Spark would rebuild only this partition. The recovery starts at the first shuffle boundary, ShuffledRDD[3] in my lineage, between stage 0 and stage 1. Spark reuses the shuffle files that stage 0 already wrote and redoes the sum for this partition. Then it runs again the tasks of stages 1 and 2 that read it. It never restart the whole job.
<!--/answer-->

## 3. My cloud MapReduce mapped onto Hadoop

Name the GCP service or file that played each role.

JobTracker, which schedules tasks, waits for the map phase and retries failures.
<!--answer:role_jobtracker-->Cloud Workflows plays the JobTracker. The job is defined in workflow.yaml.<!--/answer-->

Workers, which run the map and reduce code.
<!--answer:role_workers-->The Cloud Run function in main.py plays the workers. Workflows calls it once for each map task and each reduce task.<!--/answer-->

HDFS, which holds the inputs, the shuffle files and the outputs.
<!--answer:role_hdfs-->Cloud Storage plays HDFS. My bucket holds the input splits, the shuffle files and the output partitions.<!--/answer-->

Partitioner, which decides which reducer owns a key.
<!--answer:role_partitioner-->The partition function in mapreduce.py plays the Partitioner. It takes zlib.crc32 of the word modulo the number of reducers.<!--/answer-->

### The barrier

There is no explicit step that waits for all mappers in workflow.yaml. Where exactly is the barrier between the map and reduce phases? Which construct provides it? (about 30 words)

<!--answer:barrier_location-->
The barrier between the map and reduce phases is the end of the parallel step map_phase in workflow.yaml. A parallel step only finishes when all its branches finish. So reduce_phase cannot start before the last map task is done.
<!--/answer-->

### Idempotence

The workflow retries a task that failed. Why is that safe with the worker as written? What would go wrong if run_map appended its pairs to the shuffle file instead of overwriting it? (about 60 words)

<!--answer:idempotence-->
The workflow retries a failed task, and it is safe because run_map is idempotent. A map task always reads the same split and writes the same files with the same names. On a retry it overwrites these files with the same content. If run_map appended its pairs instead, a retried task would write it's pairs two times. The reducer would read duplicated pairs and some counts would be too high. The answer would not match the local reference anymore.
<!--/answer-->

### The chaos run

What did you see in the execution log and in Logs Explorer when you ran with chaos at 0.3? Did the job still succeed? Was the final answer still identical to the local reference? (about 50 words)

<!--answer:chaos_observations-->
When I ran with chaos at 0.3, I read the worker logs with a gcloud logging read query. I found 15 requests with status 503 and 16 requests with status 200. The 503 answers was very fast because the crash happens before any work. The job still succeeded in 48.0 s, against 35.6 s for the normal run. The answer was still identical to the local reference, with 2221 distinct words.
<!--/answer-->
