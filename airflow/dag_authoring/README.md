# Airflow Dag Authoring Course


# Dag Files and Dag syntax 

Scheduler will only parse the file in dags/ folder if it contains the word "airflow" or "dag".
(attention when generating dags dynamically)

DAG_DISCOVERY_SAFE_MODE (core) to False, in case you want to parse all files in dags/ folder.

.airflowignore in case you want to have thins int he dag folder but not dags.

### How to instantiate a Dag

`from airflow import DAG`

`from datetime import datetime, timedelta`

How to instantiate your DAG? Two Options:
1 - Context Manager (recommended)
```python
with DAG(...) as dag:
	DummyOperator()

```

2 - Creating variable dag (you have to do dag=dag in every operator, not recommended)

```python
dag = DAG(...)
DummyOperator(dag=dag)

```

### Dag Parameters

* dag_id
unique identifier of your dag among all your dags
If you have two dags with same dag_id, you dont receive any error and it will show ramdonly one or another after parsing

* description
Best practice to describe your dag. 
Ex: description="DAG in charge of processing the customer data"

* start_date
It's not required in the dag object, but Operators will fail if not instantiated.
Instatiate in default arguments or dag object? Course says dag object.
from datetime import datetime
Ex: start_date=datetime(2021,1,1)
- Each operator, each task has a start_date!


* schedule_interval
Defines the frequency that the DAG is triggered.
Interval of time from the min(start_date) at which DAG is triggered.
schedule_interval='* * * * * '
Exs: '@daily','@weekly',timedelta(minutes=5)

* dagrun_timeout
If dag more than X time, kill dag.
By default there is no timeout.
Useful for DAGs with little schedule interval, like 10 minutes, set a timeout of 12 minutes,for example.
EX: dagrun_timeout=timedelta(minutes=12)

* tags
Tags to each DAG, to filter dags in the UI. Useful for organizing DAGs, Teams, purpose, area, business unit...

* catchup
This says to trigger past dag runs automatically, if a daily dag paused for 5 days, when you unpause it if catchup is True, it will trigger the DAG for all the days.
Best practice to always set this to False.
Always set to false with enviromnent variable... (SEARCH)

* max_active_runs
When you dont want many dag runs for the same dag at the same time, you can limit it with this parameter.


## Dag Scheduling

- Dag Trigger X Dag Run

Important parameters:
start_date,schedule_interval

"
The DAG X starts being scheduled from the start_date and will be triggered after every schedule_interval
"

![Dag Scheduling](dag_scheduling.png "Dag Intervals")

So the execution_date is when the DAG run starts, but the DAG is TRIGGERED only AFTER the period of time.


- Cron X Time Delta

Cron is stateless - always at that time.

Time Delta is stateful - get the last execution_date or if the first, the start_date. The DAG run starts at start_date + timedelta, and the DAG is really triggered at start_date +timedelta* 2


### Task Idempotence and Determinism

Deterministic DAG: IF execute DAG with same input you get same output

Idempotence: If you execute DAG multiple times, you get the same effect.

Idempotence Example: adding if not exists to create table statement,
```python

PostgresOperator(task_id='create_table', sql='CREATE TABLE IF NOT EXISTS...')
```
Make sure you can execute your tasks more than once, and with the same input, same output. To rerun your tasks whenever you need!

### Backfill

You created your pipeline, but you want to run it in the past. 
With catchup = True, you can backfill all your DAG runs. But if your schedule interval is little and start_date is far away, you have too many dag runs at the same time.
To overcome this we can use the airflow CLI:
```bash
airflow dags backfill -s 2020-01-01 -e 2021-01-02
```

To control the number of dag runs, ise the parameters, max_active_runs

To use the UI to trigger past runs, use Search, filter your dag, and "start date" and "end_date" and clear the state.

## Variables

If you have a S3 bucket that is used in many DAGS, OR an API URL... use variables!

Create variable
- UI: Admin => Variables => Create new
- API
- CLI

Tip: name your variables with the dag_id and then the variable = "my_dag_variable_name"

To use variables you need to:
```python
from airflow.models import Variable

my_variable = Variable.get("my_dag_variable_name")
```

### How to hide variables:

Just add one of the chosen words to your variable name:
'password','secret','apikey','api_key'

how to add more word SENSITIVE_VAR_CONN_NAMES at core in airflow.cfg

Then the variable is hidden in the UI and in the logs, even if you print it!

### Limitations of variables!

Variable.get() create a connection to the DB. If you instantiate this outside of a task, it will create a connection everytime the dag is parsed!

Maybe you need to instantiate a variable outside task, maybe if you generate dynamically tasks based on a variable, but always avoid this!

Merge many variables inside one!
Using a json in the value of the variable! then you only use one connection to fetch many correlated information!
```python
my_dag_variable_name = {
	"name": "partner_a",
	"api_secret":"my_secret",
	"path": "/tmp/path"
}
partner_setting = Variable.get("my_dag_variable_name", deserialize_json=True)
name = partner_setting['name']
api_key= partner_setting['api_secret']
path = partner_setting['path']

```

If you use the variabel in a "op_args" or "op_kwargs", it will connect every DAG parse!
TO overcome this, use jinja templating!
```python
extract = PythonOperator(
task_id = 'extract',
python_callable = extract,
op_args = ["{{ var.json.my_dag_variable_name.name }}"]
```

### The Power of Environment Variables

To create a env variable in Docker, add this to your dockerfile:

```bash
ENV AIRFLOW_VAR_VARIABLE_NAME='{"name":"partner_a","api_secret":"my_secret","path": "/tmp/path"}'

```
That way, doesnt show on UI or CLI, and doesnt make a connection to the database!

There are 6 different ways of creating variables in Airflow 😱

    Airflow UI   👌
    Airflow CLI 👌
    REST API 👌
    Environment Variables ❤️
    Secret Backend ❤️
    Programatically 😖

Whether to choose one way or another depends on your use case and what you prefer.

Overall, by creating a variable  with an environment variable you

    avoid making a connection to your DB
    hide sensitive values (you variable can be fetched only within a DAG)

Notice that it is possible to create connections with environment variables. You just have to export the env variable:

AIRFLOW_CONN_NAME_OF_CONNECTION=your_connection

and get the same benefits as stated previously.

Ultimately, if you really want a secure way of storing your variables or connections, use a Secret Backend.

To learn more about this, click on this linkhttps://airflow.apache.org/docs/apache-airflow/stable/security/secrets/secrets-backend/index.html

Finally, here is the order in which Airflow checks if your variable/connection exists:

Secret Backends => Environment variable => DB


### Jinja Templating

One of the most powerful things to do with Airflow when authoring a DAG.

Using with a json environment variable: 

```python
"{{ var.json.mydagvariable.name }}"
```
Go to [Registry Astronomer PostgresOperator](https://registry.astronomer.io/providers/postgres/modules/postgresoperator)

See that `sql` is templated. If you go to the code of the PostgresOperator, you can see:
```python
template_fields = ('sql')
template_fields_renderes = {'sql': 'sql'}
template_ext = ('.sql',)
ui_coler = '#ededed'
```
To see that you can use `.sql` 


```python
fetching_data = PostgresOperator(
    task_id="fetching_data",
    sql="SELECT partner__name FROM partners WHERE date={{ ds }}"
    )
```
You shouldn't put a query like this in your DAG!
Create a QUERY.sql in your `dags/sql`

Like this:
```sql
SELECT partner__name FROM partners WHERE date={{ ds }}
```
And then the PostgresOperator:
```python
fetching_data = PostgresOperator(
    task_id="fetching_data",
    sql="sql/QUERY.sql"
    )
```
This will work and make a "different" query in runtime!


But, you can customize the Operator, to template some parameters!

On the top of you dag file, define a new custom operator based on PostgresOperator:
```python
class CustomPostgresOperator(PostgresOperator):
    """docstring for CustomPostgresOperator"""
    template_fields = ('sql', 'parameters')
        
```
And then your `fetching_data` becomes:

```python
fetching_data = CustomPostgresOperator(
    task_id="fetching_data",
    sql="sql/QUERY.sql",
    parameters={
        'next_ds': '{{ next_ds }}',
        'prev_ds': '{{ prev_ds }}',
        'partner_name': '{{ var.json.mydagvariable.partner }}'
    }
    )
```

Then go to the Airflow UI, click on the dag, graph view, click on the task, `rendered` and see the template!


 

To see a list of envs you can enter [here](https://airflow.apache.org/docs/apache-airflow/stable/macros-ref.html) 



## Data with XCOMs

Share data between two different tasks. 
To create a XCOM you need to access the task instance object.
What's task intance? Whenever you trigger a task in Airflow you create a task instance (DAG > DAG Run).

```python
def _extract(ti):
    partner_name = "netflix"
    ti.xcom_push(key="partner_name",value=partner_name)

extract = PythonOperator(
    task_id="extract",
    python_callable=_extract)
```

The value that's "pushed" from `xcom_push` is serialized in JSON!
To get "back" the "pushed" value:

```python
def _process(ti):
    partner_name = ti.xcom_pull(key="partner_name", task_ids="extract")
    print(partner_name)
    
process = PythonOperator(
    task_id="process",
    python_callable=_process)
```

You can work with XCOMs in a simpler way!
Just with the `return` in the python function! But then the key name will be `return_value` as default.

```python
def _extract(ti):
    partner_name = "netflix"
    return partner_name


def _process(ti):
    partner_name = ti.xcom_pull(task_ids="extract")
    print(partner_name)
    

extract = PythonOperator(
    task_id="extract",
    python_callable=_extract)

process = PythonOperator(
    task_id="process",
    python_callable=_process)
```

If you have multiple values in XCOM?

```python
def _extract(ti):
    partner_name = "netflix"
    partner_path = "/partners/netflix"
    return {"partner_name":partner_name,"partner_path":partner_path}


def _process(ti):
    partner_settings = ti.xcom_pull(task_ids="extract")
    print(partner_settings['partner_name'])
    print(partner_settings['partner_path'])
    

extract = PythonOperator(
    task_id="extract",
    python_callable=_extract)

process = PythonOperator(
    task_id="process",
    python_callable=_process)
```

IF you need two XCOMs just do `xcom_push` two times!


### Limitation with XCOMs

XCOMs are limited in size:
- for SQL Lite, max XCOM size = 2GB per XCOM
- for PostgreSQL, max XCOM size = 1GB per XCOM
- for MySQL, max XCOM size = 64Kb per XCOM

Work with the data references! Not the data itself!
Ex: share the s3 object path, not the object itself


## TaskFlow API

TaskFlow API uses Decorators and deals with XCOM arguments. [TaskFlow Tutorial](https://airflow.apache.org/docs/apache-airflow/stable/tutorial_taskflow_api.html)

### Decorators:
- Create a Operator automatically for you
- @task.python
- @task.virtualenv
- @task_group

Always import  `task` `from airflow.decorators`. Here's a full example with decorators on the functions:

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.decorators import task

from datetime import datetime, timedelta

@task.python
def extract():
    partner_name = "netflix"
    partner_path = "/partners/netflix"
    return {"partner_name":partner_name,"partner_path":partner_path}

@task.python
def process(ti):
    partner_settings = ti.xcom_pull(task_ids="extract")
    print(partner_settings['partner_name'])
    print(partner_settings['partner_path'])

with DAG("my_dag",
    description="DAG in charge of processing customer data",
    start_date=datetime(2021, 1, 1),
    schedule_interval="@daily",
    dagrun_timeout=timedelta(minutes=10),
    tags=["data_science","customers"],
    catchup=False,
    max_active_runs=1) as dag:
    

    extract() >> process()
```

And now a full example with dag decorators as well.
As a good practice, instance your decorator as `@task.python` or `@task.virtualenv` and not only `@task`, but you can.


```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.decorators import task, dag

from datetime import datetime, timedelta

@task.python
def extract():
    partner_name = "netflix"
    partner_path = "/partners/netflix"
    return {"partner_name":partner_name,"partner_path":partner_path}

@task.python
def process():
    print('process')

@dag(description="DAG in charge of processing customer data",
    start_date=datetime(2021, 1, 1),
    schedule_interval="@daily",
    dagrun_timeout=timedelta(minutes=10),
    tags=["data_science","customers"],
    catchup=False,
    max_active_runs=1)
def mydag():
    extract() >> process()

my_dag
```


### XCOM Arguments:
- Share data between your tasks without XCOM push and pull
- Helps explicit your implicit dependecies between your tasks

Using the same example as before:

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.decorators import task, dag

from datetime import datetime, timedelta

@task.python
def extract():
    partner_name = "netflix"
    partner_path = "/partners/netflix"
    return partner_name

@task.python
def process(partner_name):
    print(partner_name)

@dag(description="DAG in charge of processing customer data",
    start_date=datetime(2021, 1, 1),
    schedule_interval="@daily",
    dagrun_timeout=timedelta(minutes=10),
    tags=["data_science","customers"],
    catchup=False,
    max_active_runs=1)
def mydag():
    process(extract())

my_dag
```

We can see that we only used the argument in the function `process` and then instantiate one "inside" the other in `mydag`. With this Airflow make the dependecie like `extract >> process`


Now we will see some arguments to the `task.python` decorator and "push" multiple XCOMs.

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.decorators import task, dag

from datetime import datetime, timedelta

# from typing import Dict  # Here to use with "-> Dict"

@task.python(task_id="extract_partners",multiple_outputs=True)
# def extract() -> Dict[str, str]: # Is the same as using "multiple_outputs above"
def extract():
    partner_name = "netflix"
    partner_path = "/partners/netflix"
    return {"partner_name":partner_name,"partner_path":partner_path}

@task.python
def process(partner_name,partner_path):
    print(partner_name)
    print(partner_path)

@dag(description="DAG in charge of processing customer data",
    start_date=datetime(2021, 1, 1),
    schedule_interval="@daily",
    dagrun_timeout=timedelta(minutes=10),
    tags=["data_science","customers"],
    catchup=False,
    max_active_runs=1)
def mydag():

    partner_settings = extract()
    process(partner_settings['partner_settings'],partner_settings['partner_path'])

my_dag
```


If you dont want to push the xcom dict `{"partner_name":partner_name,"partner_path":partner_path}` just do:

```python
@task.python(task_id="extract_partners", do_xcom_push=False, multiple_outputs=True)
```


## Grouping Tasks

### Subdags (hard way)

The sub DAG file at `dags/subdag/subdag_factory.py`
```python
from airflow.models import DAG
from airflow.decorators import task
from airflow.operators.python import get_current_context

@task.python
def process_a():
    ti.get_current_context()['ti']
    partner_name = ti.xcom_pull(key='partner_name', task_ids='extract_partners',dag_id='my_dag')
    partner_path = ti.xcom_pull(key='partner_path', task_ids='extract_partners',dag_id='my_dag')
    print('Process A')
    print(partner_name)
    print(partner_path)

@task.python
def process_b():
    ti.get_current_context()['ti']
    partner_name = ti.xcom_pull(key='partner_name', task_ids='extract_partners',dag_id='my_dag')
    partner_path = ti.xcom_pull(key='partner_path', task_ids='extract_partners',dag_id='my_dag')
    print('Process B')
    print(partner_name)
    print(partner_path)

@task.python
def process_c():
    ti.get_current_context()['ti']
    partner_name = ti.xcom_pull(key='partner_name', task_ids='extract_partners',dag_id='my_dag')
    partner_path = ti.xcom_pull(key='partner_path', task_ids='extract_partners',dag_id='my_dag')
    print('Process C')
    print(partner_name)
    print(partner_path)


def subgdag_factory(parent_dag_id, subdag_dag_id, default_args, partner_settings):
    with DAG(f'{parent_dag_id}.{subdag_dag_id}',default_args=default_args) as dag:

        process_a(partner_settings['partner_name'],partner_settings['partner_path'])
        process_b(partner_settings['partner_name'],partner_settings['partner_path'])
        process_c(partner_settings['partner_name'],partner_settings['partner_path'])
    return dag

```

The main DAG that runs the Sub Dag `dags/my_dag.py`

```python
from airflow.models import DAG
from airflow.operators.python import PythonOperator
from airflow.decorators import task, dag
from airflow.operators.subdag import SubDagOperator

from datetime import datetime, timedelta
from typing import Dict
from subdag.subgdag_factory import subgdag_factory

@task.python(task_id="extract_partners",do_xcom_push=False, multiple_outputs=True)
def extract():
    partner_name = 'netflix'
    partner_path = '/partners/netflix'
    return {"partner_name": partner_name, "partner_path": partner_path}

default_args = {
    'start_date': start_date=datetime(2021, 1, 1)
}

@dag(description="DAG in charge of processing customer data",
    default_args=default_args,
    schedule_interval="@daily",
    dagrun_timeout=timedelta(minutes=10),
    tags=["data_science","customers"],
    catchup=False,
    max_active_runs=1)

def my_dag():
    partner_settings = extract()

    process_tasks = SubDagOperator(
        task_id='process_tasks',
        subdag=subgdag_factory('my_dag', 'process_tasks', default_args),
        poke_interval=15,
        mode='reschedule'
    )

    extract() >> process_tasks

dag = my_dag()

```

### Task Groups (easy way)

The main DAG that runs the Task Group `dags/my_dag.py`

```python
from airflow.models import DAG
from airflow.operators.python import PythonOperator
from airflow.decorators import task, dag
from groups.process_tasks import process_tasks

from datetime import datetime, timedelta
from typing import Dict

@task.python(task_id="extract_partners",do_xcom_push=False, multiple_outputs=True)
def extract():
    partner_name = 'netflix'
    partner_path = '/partners/netflix'
    return {"partner_name": partner_name, "partner_path": partner_path}

default_args = {
    'start_date': start_date=datetime(2021, 1, 1)
}

@dag(description="DAG in charge of processing customer data",
    default_args=default_args,
    schedule_interval="@daily",
    dagrun_timeout=timedelta(minutes=10),
    tags=["data_science","customers"],
    catchup=False,
    max_active_runs=1)

def my_dag():

    partner_settings = extract()

    process_tasks(partner_settings)


dag = my_dag()

```

The task group file to keep your dag file clean at `dags/groups/process_tasks.py`

```python
from airflow.decorators import task
from airflow.utils.task_group import TaskGroup


@task.python
def process_a(partner_name,partner_path):
    print('Process A')
    print(partner_name)
    print(partner_path)

@task.python
def process_b(partner_name,partner_path):
    print('Process B')
    print(partner_name)
    print(partner_path)

@task.python
def process_c(partner_name,partner_path):
    print('Process C')
    print(partner_name)
    print(partner_path)

def process_tasks(partner_settings):
    with TaskGroup(group_id='process_tasks') as process_tasks:
        process_a(partner_settings['partner_name'],partner_settings['partner_path'])
        process_b(partner_settings['partner_name'],partner_settings['partner_path'])
        process_c(partner_settings['partner_name'],partner_settings['partner_path'])

    return process_tasks

```

You can have a task group inside a task group.
Just import a task group inside a task group definition!

Airflow have a task group decorator!

## Dynamic Tasks

If you have a dict of values, and want to execute the same tasks for each one!

- add_suffix_on_collision parameter when difining dynamic tasks
- or you can use a f'{value}' for defining the task_id 



## Choices with Branching

- BranchPythonOperator
```python
from airflow.operators.python import PythonOperator, BranchPythonOperator

def choosing_based_on_day_func(execution_date):
    #HAve to return a task_id
    day = execution_date.day_of_week
    if day == 1:
        return task_id_1
    elif day == 3:
        return task_id_2
    elif day == 5:
        return task_id_3
    else:
        return 'stop' #or using a shortcircuit operator before this


choosing_based_on_day = BranchPythonOperator(
    task_id=choosing_based_on_day,
    python_callable=choosing_based_on_day_func)
```
- BranchSQLOperator
- BranchDatetimeOperator
- BranchDayofWeekOperator

Common problem using branching is a task after the branching. 
If you choose one path and has a task after, it will be skipped, since not ALL tasks reached the final task in sucess. You can use trigger rules.
add to your final task:
```python
final_task = DummyOperator(task_id='final_task', trigger_rule='none_failed_or_skipped')
```


## Trigger Rules

Behavior of the task based on the tasks before.
The default is `trigger_rule='all_sucesss`
- all_sucesss
- all_failed
- all_done
- one_failed (does not wait to all tasks to complete, trigger if any task completed is failed)
- one_sucess (does not wait to all tasks to complete, trigger if any task completed is sucess)
- none_failed
- none_skipped
- none_failed_or_skipped (all the tasks completed, and none failed, can be just one sucess and other skipped)
- dummy - just trigger


## Set dependencies in tasks

```python
t2.set_upstream(t1)
#or
t1.set_downstream(t2)

t2 << t1
#or
t1 >> t2

[t1,t2,t3] >> [t4,t5,t6]  # (Does not work)

# We could try: 
[t1,t2,t3] >> t4
[t1,t2,t3] >> t5
[t1,t2,t3] >> t6

# But you can use cross_downstream
# example 1
from airflow.models.baseoperator import cross_downstream

cross_downstream([t1,t2,t3],[t4,t5,t6])

#if you have another task after [t4,t5,t6] you have to

cross_downstream([t1,t2,t3],[t4,t5,t6])
[t4,t5,t6] >> t7

# chain

#example 2
from airflow.models.baseoperator import chain

chain(t1,[t2,t3],[t4,t5],t6) #middle must have same number of steps

#mix the two!

#example 3
from airflow.models.baseoperator import chain

cross_downstream([t2,t3],[t4,t5])
chain(t1,t2,t5,t6)


```

![Dependencies Example 1](dependencies_example_1.png "Tasks Dependencies")
![Dependencies Example 2](dependencies_example_2.png "Tasks Dependencies")
![Dependencies Example 3](dependencies_example_3.png "Tasks Dependencies")

## Control your tasks

* Parallelism
- Default is 32 = You can run 32 tasks in the same time in your entire airflow instance

* Dag Concurrency
- Default is 16 = You can run 32 tasks in the same time in your DAG

* Max Active runs per dag
- Default is 16 = You can run 32 Dag Runs in the same time in your DAG

You can define some things in your DAG!

In the DAG definition: `concurrency=2`, `max_active_runs=1`

In the Task definition: `task_concurrency=1`, `pool='default_pool`

### Pools

Only for these specific tasks, I want 1 task at a time and these other 3 at a time.
Pool with a worker slots. Default all of the tasks, run in the pool `default_pool`
You can go to the UI in Admin/Pools to see your pools

- Create pool
    - In Admin/Pools create a pool with 1 slot
    - In the task definitions: `pool='new_pool'`
    - In any operator, `pool_slots=3`
    - If you instantiate pool in the SubDagOperator, only in the operator, not the tasks inside it
    -  

### Task and DAG Priorities

All tasks have the parameter `priority_weight`, that means higher the number the higher the priority.
Priorities are evaluated in the same pool, if they are in different pools, each pool see its 'own' priorities.

There is an argument `wait_pool` that defines how the `priority_weight` of the tasks are computed.
It's how the airflow computes which tasks runs first! Sum of the priorities backwards 
- Downstream (default) - final task = 1, middle task = 1 + 1 = 2 first task = 1+1+1=3
- Upstream - Opposite of downstream
- Absolute - Just execute as the definition of priority_weights

You can prioritize the DAG over another, setting the weights of one all 99 and the other all 1, for example.


### Depends On Past

Depends on past is evaluated as tasks. If True, and task X failed on previous DAG RUN, it will not be triggered, no status at all (no failed, no skipped...).
Depends on past triggers the task if the task has a success status or skipped status
Manually triggered is also evaluated.
In first DAG RUN, depends on past is not evaluated (obvious)
In backfill jobs, the started date is overridden.

### Wait for downstream

Wait for downstream means that the same task in previous DAG RUN has sucess status AND all directly downstream too.

Imagine a pipeline with 3 tasks A >> B >> C.
With `wait_for_downstream = True` for task A, on the second DAG RUN, task A and B need to have sucess status.
If b has a failed status, A will not trigger. But if A,B is sucess and C is failed, A WILL trigger.

`wait_for_downstream = True` also sets `depend_on_past = True` automatically


### Sensors

Sensors waits for something to happen to continue the pipeline.
It takes a worker slot from the pool! Some times just to do nothing!


Datetime sensor: You can use if some tasks you have to wait until determined datetime to trigger
```python
from airflow.sensors.date_time import DateTimeSensor

task_sensor_date_time = DateTimeSensor(
    task_id='task_sensor_date_time',
    target_time="{{ execution_date.add(hours=9) }}", #especific for DateTimeSensor
    poke_interval=60, # in seconds
    mode='poke', #default is 'poke'
    timeout=60 * 60,  # in seconds, 7 days by default
    soft_fail=True, # after timeout, puts status = skipped
    exponencial_backoff=True # increase the poke_interval exponencially in each poke

    )

```

With `mode='reschedule'` you use a worker slot only when needed (good for large poke_intervals + 10 min)

There's chance that the sensor is never True, so we need to define `timeout`! use wisely.

`execution_timeout` (that all operators have) doesnt evaluate `soft_fail`

Always define a timeout!

## Timeouts

Timeout in the DAG instance
- `dagrun_timeout=timedelta(minutes=10)`
- It doesnt get evaluated by manually trigged DAG RUNs.

Timeout at the operator level
- `execution_timeout=timedelta(minutes=10)`

Always define both! It's a good practice!

## What to do after the tasks/DAGs sucess/failed?

You can leverage trigger_rules, tasks only triggers if other tasks failed.

Or you can use callbacks!

DAGs:
- Want to do something if DAG succeded? `on_success_callback=success_function`
```python
def success_function(context):
    print(context) #context contains much information about the DAG

```
- Want to do something if DAG failed? `on_failure_callback=failure_function`
```python
def failure_function(context):
    print(context) # send email, notification...

```

Tasks

- Failure: `on_failure_callback=failure_function`
```python
from airflow.exceptions import AirflowTaskTimeout, AirflowSensorTimeout

def failure_function(context):
    if context['exception']:
        if (isintance(context['exception'], AirflowTaskTimeout)):
            print('Task Timeout Error')
        if (isintance(context['exception'], AirflowSensorTimeout)):
            print('Sensor Timeout Error')
```
- Success: `on_success_callback=success_function`
- Retry: `on_retry_callback=retry_function`
```python
from airflow.exceptions import AirflowTaskTimeout, AirflowSensorTimeout

def retry_function(context):
    tries = context['ti'].try_number()
    if tries > 2:
        print('Too much tries')
```


Learn more about context:
https://composed.blog/airflow/execute-context
https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html

### Ways to retry tasks

We should define the retries of your task, because often we dont want to fail the task on first try.

We can define at the Airflow instance level, with `DEFAULT_TASK_RETRY` configuration, All tasks receive this by default.

Or in the DAG level with `retries` argument in the `default_args`, all tasks inside that DAG received this retry count.

Or we can define in the task level, that overrides all of the above.
- `retries=2` - Argument in the definition of the task.
- `retry_delay=timedelta(minutes=5)` - States the interval between retries.
- `retry_exponencial_backoff=True` - Add more time to the interval between retries.
- `max_retry_delay=timedelta(minutes=60)` - Often used with `retry_exponencial_backoff` and `retries` with a large number.



### Notifications - SLA

We can use SLAs to know if a task is getting longer to complete than expected!
SLA is different than timeouts. Just notify, and not fail the tasks itself.

If the DAG is triggered manually, slas dont apply.

You have to have configures the SMTP, to get the emails.

In the task definition, `sla=timedelta(minutes=5)`.
It uses the DAG `execution_date` + `sla` to define the timestamp of the SLA.

For example if you want to set a SLA for the DAG, just define a SLA argument in the last task!

To notify or do somenthing, use the `sla_miss_callback=sla_function` IN THE DAG DEFINITION, NOT TASK!
Parameters:
- dag: dag object
- task_list: all tasks
- blocking_task_list: tasks dependencies, if task 2 miss sla because of task 1 
- slas: 
- blocking_tis: same blocking_task_list, but for ti

```python
from airflow.exceptions import AirflowTaskTimeout, AirflowSensorTimeout

def sla_function(dag, task_list, blocking_task_list, slas, blocking_tis):
    print(task_list)
    print(blocking_task_list)
    print(slas)
    print(blocking_tis)
```

### DAG versioning

When you add tasks to your DAG, the past DAGRuns will not execute the new task, will have no status at all.

If you remove a task, the logs and task runs will be missing in the UI and hard to find logs.

So a best practice add a suffix in the DAG id `with DAG('process_x_0_1,...)` for example.

Wait for DAG versioning feature!

### Dynamic DAGs


### Dag Dependencies with External Task Sensor

A sensor that waits for another task in another DAG to complete.
It waits a specific task within a especific execution_date.

IT brings a issue, if the DAG with the sensor does not have the same schedule interval?

Can be very complicated to sync the Task Sensor and the schedule intervals...


```python
from airflow.sensors.external_task import ExternalTaskSensor

wait_for_external_task = DateTimeSensor(
    task_id='wait_for_external_task',
    external_dag_id='my_dag',
    external_task_id='task_id',
    execution_delta=timedelta(minutes=600),
    execution_date_fn= date_func(), #expects a function when the execution delta have to be more complicated
    failed_states=['failed','skipped'], # status to fail the external task sensor
    allowed_states=['success']

    )

```


### Dag Dependencies with Trigger DAG Run Operator

Maybe, you can do a task in the DAG that calls the other DAG, because wait for one task in the other DAG can be complicated! Easier!

But it's not a Sensor! For example, it does not have `mode` argument.

When you run a DAG in the same execution date with `reset_dag_run` False, it will fail cause it already exists!


```python
from airflow.operators.trigger_dagrun TriggerDagRunOperator

trigger_next_dag = DateTimeSensor(
    task_id='trigger_next_dag',
    trigger_dag_id='my_dag',
    execution_date='{{ execution_date }}', # expects a string or datetime object
    wait_for_completion=True,  # wait for other DAG to complete to continue to next task in this DAG
    poke_interval=600,  # poke interval for wait_for_completion
    reset_dag_run=True, # Cleans the DAG Run with the same execution date if needed
    failed_states=['failed'] # Default is an empty list, DEfine THIS!
    )

```



### Nice cheats

* Test Dag
```bash
airflow tasks test dag_id task_id 2021-01-01
```
* Variable: Webserver Instance Name (use this to diferentiate dev and prod environments)
```python
AIRFLOW__WEBSERVER__INSTANCE_NAME = "DEV"
AIRFLOW__WEBSERVER__INSTANCE_NAME = "PROD"
```
* Variable: Webserver Instance Name (use this with instance name to diferentiate dev and prod environments)
```python
AIRFLOW__WEBSERVER__NAVBAR_COLOR = "#ffff"
AIRFLOW__WEBSERVER__NAVBAR_COLOR = "#00008B"
```


### Airflow Scheduling webinar
- Always specify a static ``startdate`
- Always specify cacthup = False and use Refill Dags!
- 

- start_date, schedule_interval, end_date


NEW Schdeduling!
- data_interval_start = execution_date
- data_interval_end
- ds
- prev_data_interval_start_success
- prev_data_interval_end_success
- prev_start_date_success
    - You can use to use a branchoperator to make a dag run only if the last is success
- ALL DATES USE IN UTC!
-

Whats the diffrence between dinifin start_date in default args or in DAG()

- You can define start_date in tasks!
    - but if three tasks with != start_date the first DAG run will the lower date
- default_args => to all of your tasks!
- if != start_dates in default_args or DAG(), airflow get the MAX(start_date)

- Always put start_date in DAG()


TIMEDELTA X CRON
- horario de verão (DST) - if you set local timezone on DAG


TIMETABLES
- TIMETABLES ARE PARSED BY SCHDELUER EVERY X SECONDS
- 4 steps to implement
    - Define constraints
    - Register your timetable as a `plugin`
    - 

```python
from weekday_time import AfterWorkdayTimeTable


```

### Version 2.2

* Timetables
* Deferrable Tasks
    Good when submiting a job in a external system
    does not consume a worker slot, runs a hurndrer in a async process
        async:
            -use less resources
            - resilient against restarts
            - Paves the way to event-based dags
    Smart Sensors from AirBnb
    DateTimeSensorAsync
    TimeDeltaSensorAsync
* task.docker decorator
    - WHen you have multiple teams and cannot install all reqs, use docker operator!
    - import inside functions
    @task.docker(image='python3.9-slim-buster')
    @task.container???

* validation of dag Parameters