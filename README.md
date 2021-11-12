# Cloud Composer: Qwik Start - Console

Adapted from: https://www.qwiklabs.com/focuses/2456

0. Enable API services: Cloud Composer and Cloud Dataproc.

1. Go to Cloud Composer.

2. Create environment
	1. **Name:** standard-cpu
	2. **Location:** us-central1
	3. **Zone:** us-central1-a
	4. **Machine type:** n1-standard-1
	5. **Image version:** select latest version
	6. **Python version:** 3
	7. Leave all other settings as default.

3. Click **Create**.

4. Create a cloud storage bucket (e.g. `mantooth-bucket-20211112`)

5. Good explanation to how Airflow works:
> Cloud Composer workflows are comprised of  [DAGs (Directed Acyclic Graphs)](https://airflow.incubator.apache.org/concepts.html#dags) . DAGs are defined in standard Python files that are placed in Airflow’s DAG_FOLDER. Airflow will execute the code in each file to dynamically build the DAG objects. You can have as many DAGs as you want, each describing an arbitrary number of tasks. In general, each one should correspond to a single logical workflow.

6. Create DAG file: [hadoop_tutorial.py](https://www.qwiklabs.com/focuses/2456?catalog_rank=%7B%22rank%22%3A2%2C%22num_filters%22%3A3%2C%22has_search%22%3Atrue%7D&parent=catalog&search_id=14488885#:~:text=the%20concepts%20here.-,Defining%20the%20workflow,-Now%20let%27s%20discuss):
```py
"""
Example Airflow DAG that:
1. Creates a Cloud Dataproc cluster
2. Runs the Hadoop wordcount example
3. Deletes the cluster

This DAG relies on three Airflow variables
https://airflow.apache.org/concepts.html#variables
1. gcp_project - Google Cloud Project to use for the Cloud Dataproc cluster.
2. gce_zone - Google Compute Engine zone where Cloud Dataproc cluster should be.
3. gcs_bucket - Google Cloud Storage bucket to use for result of Hadoop job.

See https://cloud.google.com/storage/docs/creating-buckets for creating a bucket.
"""

import datetime
import os
from airflow import models
from airflow.contrib.operators import dataproc_operator
from airflow.utils import trigger_rule

# Output file for Cloud Dataproc job
output_file = os.path.join(
	models.Variable.get("gcs_bucket"),
	datetime.datetime.now().strftime("%Y%m%d-%H%M%S")
) + os.sep

# Path to Hadoop wordcount example available on every Dataproc cluster
WORDCOUNT_JAR = (
	"file://usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar"
)

# Arguments to pass to Cloud Dataproc job
wordcount_args = [
	"wordcount",
	"gs://pub/shakespeare/rose.txt",
	output_file
]

yesterday = datetime.datetime.combine(
	datetime.datetime.today() - datetime.timedelta(1),
	datetime.datetime.min.time()
)

default_dag_args = {
	# Setting start date as yesterday starts the DAG
	# immediately when it is detected in the Cloud Storage
	# bucket
	"start_date": yesterday,
	
	# To email on failure or retry set "email" arg to your
	# email and enable
	"email_on_failure": False,
	"email_on_retry": False,

	# If a task fails, retry it once after waiting at least 5
	# minutes
	"retries": 1,
	"retry_delay": datetime.timedelta(minutes=5),
	"project_id": models.Variable.get("gcp_project")
}

# [START composer_hadoop_schedule]
with models.DAG(
	"composer_hadoop_tutorial",
	
	# Continue to run DAG once per day
	schedule_interval=datetime.timedelta(days=1),
	default_args=default_dag_args
) as dag:
	# [END composer_hadoop_schedule]
	
	# Create a Cloud Dataproc cluster
	create_dataproc_cluster =\
	dataproc_operator.DataprocClusterCreateOperator(
		task_id="create_dataproc_cluster",
		cluster_name="composer-hadoop-tutorial-cluster-{{ds_nodash}}",
		num_workers=2,
		region="us-central1",
		zone=models.Variable.get("gce_zone"),
		image_version="2.0",
		master_machine_type="n1-standard-2",
		worker_machine_type="n1-standard-2"
	)

	# Run the Hadoop wordcount example installed on
	# the Cloud Dataproc cluster master node
	run_dataproc_hadoop =\
	dataproc_operator.DataProcHadoopOperator(
		task_id="run_dataproc_hadoop",
		region="us-central1",
		main_jar=WORDCOUNT_JAR,
		cluster_name="composer-hadoop-tutorial-cluster-{{ds_nodash}}",
		arguments=wordcount_args
	)
	
	# Delete Cloud Dataproc cluster
	delete_dataproc_cluster =\
	dataproc_operator.DataprocClusterDeleteOperator(
		task_id="delete_dataproc_cluster",
		region="us-central1",
		cluster_name="composer-hadoop-tutorial-cluster-{{ds_nodash}}",
		# Setting trigger_rule to ALL_DONE causes the
		# cluster to be deleted even if the Dataproc
		# job fails
		trigger_rule=trigger_rule.TriggerRule.ALL_DONE
	)
	
	# [START composer_hadoop_steps]
	# Define DAG dependencies
	create_dataproc_cluster >> run_dataproc_hadoop >>\
	delete_dataproc_cluster
	# [END composer_hadoop_steps]
```

7. Copy to DAG file to the dags folder in the Cloud Storage bucket that was automatically created when you created the environment:
```bash
gsutil cp hadoop_tutorial.py gs://us-central1-standard-cpu-6ab1cefd-bucket/dags
```
