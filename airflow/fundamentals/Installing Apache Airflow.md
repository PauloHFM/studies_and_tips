Installing Apache Airflow

Set up and running Apache Airflow through Docker is undoubtedly the easiest way.

But,

Knowing how to install Apache Airflow manually is important as well.

Why?

How could you customize your Airflow Docker image, like installing dependencies, if you don't know what to modify?
Steps

Let's discover the different steps to install and set up Airflow.
1/ Python

First, make sure you've Python 3.6+ installed on your computer.

Notice that, since Apache Airflow 2.0 you can't run it on Python 2.7 anymore. (which is great btw 😀 )
2/ Pip

The official way of installing Airflow is with Pip.

Pip is a python package manager used to install and manage python packages. 

You will use pip to install Apache Airflow as well as all dependencies it needs to run.
3/ System dependencies

On Linux, you have to install certain operating system dependencies as shown below

sudo apt-get install -y --no-install-recommends \
        freetds-bin \
        krb5-user \
        ldap-utils \
        libffi6 \
        libsasl2-2 \
        libsasl2-modules \
        libssl1.1 \
        locales  \
        lsb-release \
        sasl2-bin \
        sqlite3 \
        unixodbc

4/ Installing Apache Airflow

Once you're done will the previous steps, you're ready to install Apache Airflow.

The command is pretty simple, you just need to execute:

pip install apache-airflow

But, wait a minute!

That's not the best way. Indeed, Airflow bring MANY dependencies.

The problem is, if one dependency is updated, the previous command will install the latest version which may cause dependency incompatibilities and errors.

That's why it is always recommend to add a constraint file.

A constraint file contains all python libraries needed by Airflow along with their pined version.

That way, you make sure the right dependencies are installed.

Execute the following command,

pip install apache-airflow --constraint

https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt

Where AIRFLOW_VERSION is the version of Airflow you want to install, 2.0.0 and PYTHON_VERSION, your python version, 3.6, 3.7 or 3.8.

Well done! 

At this point you've successfully installed Airflow but... you can't run it.

Why?

Because it needs to be initialized first.

This process does 2 steps:

    generating the files and folders needed by Airflow (logs/, airflow.cfg, unitests.cfg etc.)
    initializing the metadata database (SQLite by default)

AIRFLOW_HOME

By default, the files and folders are generated in the folder ~/airflow

You can modify this behaviour by setting the AIRFLOW_HOME environment variable.

For example, to initialize airflow in the folder /opt, export the following environment variable:

AIRFLOW_HOME=/opt/airflow
DB init

Once you're good with your Airflow home, initialize Airflow with:

airflow db init
Congratulations!

Airflow is ready to run with the default configuration settings!