# Amazon MWAA automatic scaling<a name="mwaa-autoscaling"></a>

The autoscaling mechanism automatically increases the number of Apache Airflow workers in response to queued tasks on your Amazon Managed Workflows for Apache Airflow \(MWAA\) environment and disposes of extra workers when there are no more tasks queued or executing\. This page describes how you can configure autoscaling by specifying the maximum number of Apache Airflow workers that run on your environment using the Amazon MWAA console\.

**Topics**
+ [Maximum worker count](#mwaa-autoscaling-onconsole)
+ [How it works](#mwaa-autoscaling-how)
+ [Using the Amazon MWAA console](#mwaa-autoscaling-console)
+ [Example high performance use case](#mwaa-autoscaling-high-volume)
+ [Troubleshooting tasks stuck in the running state](#mwaa-autoscaling-stranded)
+ [What's next?](#mwaa-autoscaling-next-up)

## Maximum worker count<a name="mwaa-autoscaling-onconsole"></a>

The following image shows where you can customize the **Maximum worker count** to configure autoscaling on the Amazon MWAA console\.

![\[This image shows where you can find the Maximum worker count on the Amazon MWAA console.\]](http://docs.aws.amazon.com/mwaa/latest/userguide/images/mwaa-console-max-worker-count.png)

## How it works<a name="mwaa-autoscaling-how"></a>

When you create an environment, Amazon MWAA creates an AWS\-managed Amazon Aurora PostgreSQL metadata database and an Fargate container in each of your two private subnets in different availability zones\. For example, a metadata database and container in `us-east-1a` and a metadata database and container in `us-east-1b` availability zones for the `us-east-1` region\.
+ The Apache Airflow workers on an Amazon MWAA environment use the [Celery Executor](https://airflow.apache.org/docs/apache-airflow/stable/executor/celery.html) to queue and distribute tasks to multiple Celery workers from an Apache Airflow platform\. The Celery Executor runs in an AWS Fargate container\. If a Fargate container in one availability zone fails, Amazon MWAA switches to the other container in a different availability zone to run the Celery Executor, and the Apache Airflow scheduler creates a new task instance in the Amazon Aurora PostgreSQL metadata database\.
+ By default, Amazon MWAA configures an environment to run hundreds of tasks in parallel \(in `core.parallelism`\) and workers concurrently \(in `core.dag_concurrency`\)\. As tasks are queued, Amazon MWAA adds workers to meet demand, up to and until it reaches the number you define in **Maximum worker count**\.
+ For example, if you specified a value of `10`, Amazon MWAA adds up to 9 additional workers to meet demand\. This autoscaling mechanism will continue running the additional workers, until there are no more tasks to run\. When there are no more tasks running, or tasks in the queue, Amazon MWAA disposes of the workers and scales back down to a single worker\. 

## Using the Amazon MWAA console<a name="mwaa-autoscaling-console"></a>

You can choose the maximum number of workers that can run on your environment concurrently on the Amazon MWAA console\. By default, you can specify a maximum value up to 25\.

**To configure the number of workers**

1. Open the [Environments page](https://console.aws.amazon.com/mwaa/home#/environments) on the Amazon MWAA console\.

1. Choose an environment\.

1. Choose **Edit**\.

1. Choose **Next**\.

1. On the **Environment class** pane, enter a value in **Maximum worker count**\. 

1. Choose **Save**\.

**Note**  
It can take a few minutes before changes take effect on your environment\.

## Example high performance use case<a name="mwaa-autoscaling-high-volume"></a>

The following section describes the type of configurations you can use to enable high performance and parallelism on an environment\.

### On\-premise Apache Airflow<a name="mwaa-autoscaling-high-volume-aa"></a>

Typically, in an on\-premise Apache Airflow platform, you would configure task parallelism, autoscaling, and concurrency settings in your `airflow.cfg` file:
+ `core.parallelism` – The maximum number of task instances that can run simultaneously across the entire environment in parallel\.
+ `core.dag_concurrency` – The maximum concurrency for DAGs \(not workers\)\.
+ `celery.worker_autoscale` – The maximum and minimum number of tasks that can run concurrently on any worker\.

For example, if `core.parallelism` was set to `100` and `core.dag_concurrency` was set to `7`, you would still only be able to run a total of `14` tasks concurrently if you had 2 DAGs\. Given, each DAG is set to run only seven tasks concurrently \(in `core.dag_concurrency`\), even though overall parallelism is set to `100` \(in `core.parallelism`\)\.

### On an Amazon MWAA environment<a name="mwaa-autoscaling-high-volume-mwaa"></a>

On an Amazon MWAA environment, you can configure these settings directly on the Amazon MWAA console using [](configuring-env-variables.md), [Amazon MWAA environment class](environment-class.md), and the **Maximum worker count** autoscaling mechanism\. While `core.dag_concurrency` is not available in the dropdown list as an **Apache Airflow configuration option** on the Amazon MWAA console, you can add it as a custom [Apache Airflow configuration option](configuring-env-variables.md)\.

Let's say, when you created your environment, you chose the following settings:

1. The **mw1\.small** [environment class](environment-class.md) which controls the maximum number of concurrent tasks each worker can run by default and the vCPU of containers\.

1. The default setting of `10` Workers in **Maximum worker count**\.

1. An [Apache Airflow configuration option](configuring-env-variables.md) for `celery.worker_autoscale` of `5,5` tasks per worker\.

This means you can run 50 concurrent tasks in your environment\. Any tasks beyond 50 will be queued, and wait for the running tasks to complete\.

**Run more concurrent tasks**\. You can modify your environment to run more tasks concurrently using the following configurations:

1. Increase the maximum number of concurrent tasks each worker can run by default and the vCPU of containers by choosing the `mw1.medium` \(10 concurrent tasks by default\) [environment class](environment-class.md)\.

1. Add `celery.worker_autoscale` as an [Apache Airflow configuration option](configuring-env-variables.md)\.

1. Increase the **Maximum worker count**\. In this example, increasing maximum workers from `10` to `20` would double the number of concurrent tasks the environment can run\.

**Specify Minimum workers**\. You can also specify the minimum and maximum number of Apache Airflow *Workers* that run in your environment using the AWS Command Line Interface \(AWS CLI\)\. For example:

```
aws mwaa update-environment --max-workers 10 --min-workers 10 --name YOUR_ENVIRONMENT_NAME
```

To learn more, see the [update\-environment](https://docs.aws.amazon.com/cli/latest/reference/mwaa/update-environment.html) command in the AWS CLI\.

## Troubleshooting tasks stuck in the running state<a name="mwaa-autoscaling-stranded"></a>

In rare cases, Apache Airflow may think there are tasks still running\. To resolve this issue, you need to clear the stranded task in your Apache Airflow UI\. For more information, see the [I see my tasks stuck or not completing](troubleshooting.md) troubleshooting topic\.

## What's next?<a name="mwaa-autoscaling-next-up"></a>
+ Learn more about the best practices we recommend to tune the performance of your environment in [Performance tuning for Apache Airflow on Amazon MWAA](best-practices-tuning.md)\.