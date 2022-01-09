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