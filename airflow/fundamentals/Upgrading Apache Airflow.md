Upgrading Apache Airflow

You don't want to get stuck with an older version of Airflow, do you?
Steps

Let's discover the different steps to upgrade Airflow 2.0.0 to 2.0.1 for example.
1/ DB

Make sure made a backup of your Airflow metadata database. Losing your data is the worst thing you can end up with so, always backup your data before any update.

If you're based on Postgres, pg_dump and pg_dumpall are your friends.

There are different ways of backing up your db, but taking a snapshot is the easiest one. 
2/ DAGs

Make sure the is no deprecated features in your DAGs. Features that are not supported anymore.

Pause all the DAGs and make sure no tasks are running.

Why? Because you don't want to have anything being written into the DB while it is getting upgraded.
3/ Upgrading Apache Airflow

Once you're done will the previous steps, you're ready to install Apache Airflow.

The command is pretty simple, you just need to execute:

pip install "apache-airflow[any_extra]==2.0.1" --constraint constraint-file
4/ Upgrade the DB

You have to execute

airflow db upgrade

to modify and apply the latest database schema as well as to map the existing data into it.
5/ Restart

Finally, restart the scheduler, webserver and worker(s)

Notice that this is the easiest way of upgrading Apache Airflow. Obviously, it might be more complex than that according to your constraints.

If you shouldn't have downtime, the size of your data, your architecture, multiple schedulers or not etc.