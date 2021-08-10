# Workers

A worker is a Python process that typically runs in the background and
exists solely as a work horse to perform lengthy or blocking tasks that
you don’t want to perform inside web processes.

## Starting Workers

To start crunching work, simply start a worker from the root of your
project directory:

```console
$ rq worker high default low
*** Listening for work on high, default, low
Got send_newsletter('me@nvie.com') from default
Job ended normally without result
*** Listening for work on high, default, low
...
```

Workers will read jobs from the given queues (the order is important) in
an endless loop, waiting for new work to arrive when all jobs are done.

Each worker will process a single job at a time. Within a worker, there
is no concurrent processing going on. If you want to perform jobs
concurrently, simply start more workers.

You should use process managers like
{doc}`Supervisor </patterns/supervisor>` or
{doc}`systemd </patterns/systemd>` to run RQ workers in production.

### Burst Mode

By default, workers will start working immediately and will block and
wait for new work when they run out of work. Workers can also be started
in *burst mode* to finish all currently available work and quit as soon
as all given queues are emptied.

```console
$ rq worker --burst high default low
*** Listening for work on high, default, low
Got send_newsletter('me@nvie.com') from default
Job ended normally without result
No more work, burst finished.
Registering death.
```

This can be useful for batch work that needs to be processed
periodically, or just to scale up your workers temporarily during peak
periods.

### Worker Arguments

In addition to `--burst`, `rq worker` also accepts these arguments:

- `--url` or `-u`: URL describing Redis connection details (e.g
  `rq worker --url redis://:secrets@example.com:1234/9` or
  `rq worker --url unix:///var/run/redis/redis.sock`)
- `--path` or `-P`: multiple import paths are supported (e.g
  `rq worker --path foo --path bar`)
- `--config` or `-c`: path to module containing RQ settings.
- `--results-ttl`: job results will be kept for this number of
  seconds (defaults to 500).
- `--worker-class` or `-w`: RQ Worker class to use (e.g
  `rq worker --worker-class 'foo.bar.MyWorker'`)
- `--job-class` or `-j`: RQ Job class to use.
- `--queue-class`: RQ Queue class to use.
- `--connection-class`: Redis connection class to use, defaults to
  `redis.StrictRedis`.
- `--log-format`: Format for the worker logs, defaults to
  `'%(asctime)s %(message)s'`
- `--date-format`: Datetime format for the worker logs, defaults to
  `'%H:%M:%S'`
- `--disable-job-desc-logging`: Turn off job description logging.
- `--max-jobs`: Maximum number of jobs to execute.

*New in version 1.8.0.*

- `--serializer`: Path to serializer object (e.g
  “rq.serializers.DefaultSerializer” or “rq.serializers.JSONSerializer”)

## Inside the worker

### The Worker Lifecycle

The life-cycle of a worker consists of a few phases:

1. *Boot*. Loading the Python environment.
2. *Birth registration*. The worker registers itself to the system so it
   knows of this worker.
3. *Start listening*. A job is popped from any of the given Redis
   queues. If all queues are empty and the worker is running in burst
   mode, quit now. Else, wait until jobs arrive.
4. *Prepare job execution*. The worker tells the system that it will
   begin work by setting its status to `busy` and registers job in the
   `StartedJobRegistry`.
5. *Fork a child process.* A child process (the “work horse”) is forked
   off to do the actual work in a fail-safe context.
6. *Process work*. This performs the actual job work in the work horse.
7. *Cleanup job execution*. The worker sets its status to `idle` and
   sets both the job and its result to expire based on `result_ttl`.
   Job is also removed from `StartedJobRegistry` and added to to
   `FinishedJobRegistry` in the case of successful execution, or
   `FailedJobRegistry` in the case of failure.
8. *Loop*. Repeat from step 3.

### Performance Notes

Basically the `rq worker` shell script is a simple fetch-fork-execute
loop. When a lot of your jobs do lengthy setups, or they all depend on
the same set of modules, you pay this overhead each time you run a job
(since you’re doing the import *after* the moment of forking). This is
clean, because RQ won’t ever leak memory this way, but also slow.

A pattern you can use to improve the throughput performance for these
kind of jobs can be to import the necessary modules *before* the fork.
There is no way of telling RQ workers to perform this set up for you,
but you can do it yourself before starting the work loop.

To do this, provide your own worker script (instead of using
`rq worker`). A simple implementation example:

```python
#!/usr/bin/env python
import sys
from rq import Connection, Worker

# Preload libraries
import library_that_you_want_preloaded

# Provide queue names to listen to as arguments to this script,
# similar to rq worker
with Connection():
    qs = sys.argv[1:] or ['default']

    w = Worker(qs)
    w.work()
```

### Worker Names

Workers are registered to the system under their names, which are
generated randomly during instantiation (see
{doc}`monitoring </docs/monitoring>`). To override this default, specify
the name when starting the worker, or use the `--name` cli option.

```python
from redis import Redis
from rq import Queue, Worker

redis = Redis()
queue = Queue('queue_name')

# Start a worker with a custom name
worker = Worker([queue], connection=redis, name='foo')
```

## Retrieving Worker Information

*Updated in version 0.10.0.*

`Worker` instances store their runtime information in Redis. Here’s
how to retrieve them:

```python
from redis import Redis
from rq import Queue, Worker

# Returns all workers registered in this connection
redis = Redis()
workers = Worker.all(connection=redis)

# Returns all workers in this queue (new in version 0.10.0)
queue = Queue('queue_name')
workers = Worker.all(queue=queue)
worker = workers[0]
print(worker.name)
```

Aside from `worker.name`, worker also have the following properties:

- `hostname` - the host where this worker is run
- `pid` - worker’s process ID
- `queues` - queues on which this worker is listening for jobs
- `state` - possible states are `suspended`, `started`, `busy`
  and `idle`
- `current_job` - the job it’s currently executing (if any)
- `last_heartbeat` - the last time this worker was seen
- `birth_date` - time of worker’s instantiation
- `successful_job_count` - number of jobs finished successfully
- `failed_job_count` - number of failed jobs processed
- `total_working_time` - amount of time spent executing jobs, in
  seconds

If you only want to know the number of workers for monitoring purposes,
`Worker.count()` is much more performant.

```python
from redis import Redis
from rq import Worker

redis = Redis()

# Count the number of workers in this Redis connection
workers = Worker.count(connection=redis)

# Count the number of workers for a specific queue
queue = Queue('queue_name', connection=redis)
workers = Worker.all(queue=queue)
```

## Worker with Custom Serializer

When creating a worker, you can pass in a custom serializer that will be
implicitly passed to the queue. Serializers used should have at least
`loads` and `dumps` method. An example of creating a custom
serializer class can be found in serializers.py
(rq.serializers.JSONSerializer). The default serializer used is
`pickle`

```python
from rq import Worker
from rq.serialzers import JSONSerializer

job = Worker('foo', serializer=JSONSerializer)
```

or when creating from a queue

```python
from rq import Queue, Worker
from rq.serialzers import JSONSerializer

w = Queue('foo', serializer=JSONSerializer)
```

Queues will now use custom serializer

### Worker Statistics

If you want to check the utilization of your queues, `Worker`
instances store a few useful information:

```python
from rq.worker import Worker
worker = Worker.find_by_key('rq:worker:name')

worker.successful_job_count  # Number of jobs finished successfully
worker.failed_job_count # Number of failed jobs processed by this worker
worker.total_working_time  # Amount of time spent executing jobs (in seconds)
```

## Better worker process title

Worker process will have a better title (as displayed by system tools
such as ps and top) after you installed a third-party package
`setproctitle`:

```sh
pip install setproctitle
```

## Taking Down Workers

If, at any time, the worker receives `SIGINT` (via Ctrl+C) or
`SIGTERM` (via `kill`), the worker wait until the currently running
task is finished, stop the work loop and gracefully register its own
death.

If, during this takedown phase, `SIGINT` or `SIGTERM` is received
again, the worker will forcefully terminate the child process (sending
it `SIGKILL`), but will still try to register its own death.

(usingaconfigfile)=

## Using a Config File

If you’d like to configure `rq worker` via a configuration file
instead of through command line arguments, you can do this by creating a
Python file like `settings.py`:

```python
REDIS_URL = 'redis://localhost:6379/1'

# You can also specify the Redis DB to use
# REDIS_HOST = 'redis.example.com'
# REDIS_PORT = 6380
# REDIS_DB = 3
# REDIS_PASSWORD = 'very secret'

# Queues to listen on
QUEUES = ['high', 'default', 'low']

# If you're using Sentry to collect your runtime exceptions, you can use this
# to configure RQ for it in a single step
# The 'sync+' prefix is required for raven: https://github.com/nvie/rq/issues/350#issuecomment-43592410
SENTRY_DSN = 'sync+http://public:secret@example.com/1'

# If you want custom worker name
# NAME = 'worker-1024'
```

The example above shows all the options that are currently supported.

To specify which module to read settings from, use the `-c` option:

```console
$ rq worker -c settings
```

Alternatively, you can also pass in these options via environment
variables.

## Custom Worker Classes

There are times when you want to customize the worker’s behavior. Some
of the more common requests so far are:

1. Managing database connectivity prior to running a job.
2. Using a job execution model that does not require `os.fork`.
3. The ability to use different concurrency models such as
   `multiprocessing` or `gevent`.
4. Using a custom strategy for dequeuing jobs from different queues. See
   {ref}`link <roundrobinrandomqueues>`.

You can use the `-w` option to specify a different worker class to
use:

```console
$ rq worker -w 'path.to.GeventWorker'
```

(roundrobinrandomqueues)=

## Round Robin and Random strategies for dequeuing jobs from queues

In certain circumstances it can be useful that a when a worker is
listening to multiple queues, say `q1`,`q2`,`q3`, the jobs are
dequeued using a Round Robin strategy. That is, the 1st dequeued job is
taken from `q1`, the 2nd from `q2`, the 3rd from `q3`, the 4th
from `q1`, the 5th from `q2` and so on. The custom worker class
`rq.worker.RoundRobinWorker` implements this strategy.

In some other circumstances, when a worker is listening to multiple
queues, it can be useful to pull jobs from the different queues
randomly. The custom class `rq.worker.RandomWorker` implements this
strategy. In fact, whenever a job is pulled from any queue, the list of
queues is shuffled, so that no queue has more priority than the other
ones.

## Custom Job and Queue Classes

You can tell the worker to use a custom class for jobs and queues using
`--job-class` and/or `--queue-class`.

```console
$ rq worker --job-class 'custom.JobClass' --queue-class 'custom.QueueClass'
```

Don’t forget to use those same classes when enqueueing the jobs.

For example:

```python
from rq import Queue
from rq.job import Job

class CustomJob(Job):
    pass

class CustomQueue(Queue):
    job_class = CustomJob

queue = CustomQueue('default', connection=redis_conn)
queue.enqueue(some_func)
```

## Custom DeathPenalty Classes

When a Job times-out, the worker will try to kill it using the supplied
`death_penalty_class` (default: `UnixSignalDeathPenalty`). This can
be overridden if you wish to attempt to kill jobs in an application
specific or ‘cleaner’ manner.

DeathPenalty classes are constructed with the following arguments
`BaseDeathPenalty(timeout, JobTimeoutException, job_id=job.id)`

## Custom Exception Handlers

If you need to handle errors differently for different types of jobs, or
simply want to customize RQ’s default error handling behavior, run
`rq worker` using the `--exception-handler` option:

```console
$ rq worker --exception-handler 'path.to.my.ErrorHandler'

# Multiple exception handlers is also supported
$ rq worker --exception-handler 'path.to.my.ErrorHandler' --exception-handler 'another.ErrorHandler'
```

If you want to disable RQ’s default exception handler, use the
`--disable-default-exception-handler` option:

```console
$ rq worker --exception-handler 'path.to.my.ErrorHandler' --disable-default-exception-handler
```

## Sending Commands to Worker

*New in version 1.6.0.*

Starting in version 1.6.0, workers use Redis’ pubsub mechanism to listen
to external commands while they’re working. Two commands are currently
implemented:

### Shutting Down a Worker

`send_shutdown_command()` instructs a worker to shutdown. This is
similar to sending a SIGINT signal to a worker.

```python
from redis import Redis
from rq.command import send_shutdown_command
from rq.worker import Worker

redis = Redis()

workers = Worker.all(redis)
for worker in workers:
   send_shutdown_command(redis, worker.name)  # Tells worker to shutdown
```

### Killing a Horse

`send_kill_horse_command()` tells a worker to cancel a currently
executing job. If worker is not currently working, this command will be
ignored.

```python
from redis import Redis
from rq.command import send_kill_horse_command
from rq.worker import Worker, WorkerStatus

redis = Redis()

workers = Worker.all(redis)
for worker in workers:
   if worker.state == WorkerStatus.BUSY:
      send_kill_horse_command(redis, worker.name)
```

### Stopping a Job

*New in version 1.7.0.*

You can use `send_stop_job_command()` to tell a worker to immediately
stop a currently executing job. A job that’s stopped will be sent to
{ref}`FailedJobRegistry <dealingwithexceptions>`.

```python
from redis import Redis
from rq.command import send_stop_job_command

redis = Redis()

# This will raise an exception if job is invalid or not currently executing
send_stop_job_command(redis, job_id)
```