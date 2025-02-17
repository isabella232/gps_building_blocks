# Function Flow
A low cost, lightweight workflow orchestration framework based on Cloud Functions.

## Background
In many data and machine learning projects, we need to have an infrastructure that can manage our “workflows” or “data pipelines”. For example, consider a workflow like this:

```ingest data from GCS -> generate features with user data -> call AutoML to get predictions -> send results to Google Ads```

The workflows are usually in the form of a DAG, where each task can depend on a few other tasks. A task can be run only if all of its dependencies are successful.

One option is to use [Cloud Composer](https://cloud.google.com/composer) to orchestrate the tasks, which can manage task dependencies automatically. Unfortunately Cloud Composer needs an always-on cluster(>= 3 compute engines) to run and costs a few hundred USD/month (even if not running anything) which is not acceptable by developers who only run the workflow a few times per month.

This solution builds workflows on top of Cloud Functions, offers similar task dependency management ability to Cloud Composer, and is much cheaper and more lightweight.

## Example
Here is a simple example of using function flow.

This example job contains four tasks from task 1 to task 4. The dependency is

```
task1 --> task2 --> task4
      \-> task3 -/
```

The four tasks will be scheduled according to the dependencies defined in the @task decorator.

File: `main.py`

```python
import json
import logging
from typing import Dict

import google.auth
from google.cloud import bigquery
from gps_building_blocks.cloud.workflows import futures
from gps_building_blocks.cloud.workflows import tasks

# Create this BQ table to run the example. See `Deployment` section for details.
TEST_BQ_TABLE_NAME = 'test_dataset.test_table'

example_job = tasks.Job(name='test_job',
                        schedule_topic='SCHEDULE')

@example_job.task(task_id='step1')
def task1(task: tasks.Task, job: tasks.Job) -> str:
  """Task 1: a simple task that returns a string."""
  del task, job  # unused
  return 'result1'


@example_job.task(task_id='step2', deps=['step1'])
def task2(task: tasks.Task, job: tasks.Job) -> str:
  """Task 2: a simple task that returns a string."""
  del task, job  # unused
  return 'result2'


@example_job.task(task_id='step3', deps=['step1'])
def task3(task: tasks.Task, job: tasks.Job) -> str:
  """Task 3: a BigQuery asynchronous job."""
  del task, job  # unused
  _, project = google.auth.default()
  dst_table_id = f'{project}.{TEST_BQ_TABLE_NAME}'
  client = bigquery.Client()
  job_config = bigquery.QueryJobConfig(
      destination=dst_table_id,
      write_disposition=bigquery.job.WriteDisposition.WRITE_TRUNCATE)

  sql = f"""
      SELECT id, content
      FROM `{project}.{TEST_BQ_TABLE_NAME}`
  """

  query_job = client.query(sql, job_config=job_config)
  bq_job_id = query_job.job_id
  return futures.BigQueryFuture(bq_job_id)


@example_job.task(task_id='step4', deps=['step2', 'step3'])
def task4(task: tasks.Task, job: tasks.Job) -> str:
  """Task 4: a job that checks the result of task 2."""
  del task  # unused
  result2 = job.get_task_result('step2')
  logging.info('in task4, got task2 result: %s', result2)
  assert result2 == 'result2'
  return 'result4'

# The cloud function to schedule next tasks to run.
scheduler = example_job.make_scheduler()

# The cloud function triggered by external events(e.g. finished bigquery jobs)
external_event_listener = example_job.make_external_event_listener()


def start(unused_request: 'flask.Request') -> str:
  """The workflow entry point."""
  example_job.start()
  return json.dumps({'id': example_job.id})
```

File: `requirements.txt`

```
gps-building-blocks
pyOpenSSL
```

## Deployment
To run the example:

1. Enable Firestore by visiting [this page](https://console.cloud.google.com/firestore). Select 'Native' mode when asked.
1. Create a table called `test_dataset.test_table` in your BigQuery, and add some fake data.
   The table schema should be `(id: String, content: String)`.
1. Create new folder and cd into it.
1. Add the files `main.py` and `requirements.txt` with the contents above.
1. Deploy the cloud functions by running the following commands:

  ```
  gcloud functions deploy start --runtime python37 --trigger-http
  gcloud functions deploy scheduler --runtime python37 --trigger-topic SCHEDULE
  gcloud functions deploy external_event_listener --runtime python37 --trigger-topic SCHEDULE_EXTERNAL_EVENTS
  ```

1. Create a log router to send BigQuery job complete logs into your PubSub
   topic for external messages (used by `task3`).

  ```
  PROJECT_ID=your_gcp_project_id

  gcloud logging sinks create bq_complete_sink \
      pubsub.googleapis.com/projects/$PROJECT_ID/topics/SCHEDULE_EXTERNAL_EVENTS \
       --log-filter='resource.type="bigquery_resource" \
       AND protoPayload.methodName="jobservice.jobcompleted"'

  sink_service_account=$(gcloud logging sinks describe bq_complete_sink|grep writerIdentity| sed 's/writerIdentity: //')

  gcloud pubsub topics add-iam-policy-binding SCHEDULE_EXTERNAL_EVENTS --member $sink_service_account --role roles/pubsub.publisher
  ```

The workflow can then be started by calling the `start` Cloud Function using the
HTTP trigger (Cloud Functions supports many ways of invocation including HTTP,
PubSub and others. See https://cloud.google.com/functions/docs/calling for
details).
