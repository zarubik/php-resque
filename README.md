# Resque for PHP
## 2.0.0-beta

Resque is a Redis-backed library for creating background jobs, placing
those jobs on multiple queues, and processing them later.

[php-resque](https://github.com/chrisboulton/php-resque) is a PHP port of the 
Redis queuing strategy. This makes it compatible with the resque-web interface,
and with other Resque libraries. (You could enqueue jobs from Ruby and dequeue
them in PHP, for instance).

This library is a more generic version of php-resque, that has been refactored
to remove global state and improve decoupling. This makes it easier to use and 
extend.

## Why this branch?

Unfortunately, several things about the original release of php-resque made it 
a candidate for refactoring:

* php-resque only supported the (rather limited) Redisent connection library. 
  And has hard-coded this support by using static accessors. This meant more
  advanced features such as replication and pipelining were unavailable.
  
  * Now, Resque for PHP supports any client object that implements a suitable
    subset of Redis commands.  No type-checking is done on the passed in connection,
    meaning you're free to use Predis, or whatever you like.
    
  * Redisent is no longer bundled along with this library. Feel free to continue
    using it though.
    
* While the public API of `php-resque` was alright, the protected API was pretty 
  much useless.

  * For instance, almost every property of the `Resque_Worker` is marked private.
    Yet, logLevel (a property that could take only three valid values) was public.
    So, the author did more to stop people extending the worker class than he 
    did to stop people using it incorrectly. That is is bass ackwards.

* Important state (the underlying connection) was thrown into the global scope 
  by hiding it behind static accessors (`Resque::redis()`). This makes things
  easier for the library author (because he/she need not think about dependencies)
  but also statically ties together the classes in the library: it makes 
  testing and extending the library hard.

  * There's no reason to do this: `Resque` instances should simply use DI and 
    take a client connection as a required constructor argument.
 
  * This improvement also allows the connection to be mocked without extending 
    the `Resque` class.
    
  * And it lets you reuse your existing connection to Redis, if you have one.
    No need to open a new connection just to enqueue a job.
    
* Logging was hard-coded to fwrite to stdout. This is unsuitable if you've 
  already forked and closed STDOUT to daemonize (i.e. if you are using a different
  forking model from the original Ruby library, such as a worker PHP-FPM pool).
  Overriding this could be done in php-resque by extending the Worker. However,
  with static methods such as `Resque_Worker::find()`, this is hard to do completely.
  
  * Logging is now provided by the Resque instance, and can be overriden easily.
  
  * Expected to allow passing in a closure soon.

* Statistic classes were static for no good reason.

  * To work, statistics need a connection to Redis, a name, and several methods 
    (get, set). State and methods to manipulate it? Sounds like a task for 
    objects! *Not* just static methods, that don't encapsulate any of the state.
    
  * Because these were static calls, the `Resque_Stat` class was hard-coded into
    several other classes, and could not be easily extended.
    
* php-resque was not PSR-0 compatible. Specifically, there's no top level namespace, 
  because the Resque class is defined outside of the Resque directory.
  
  * Fair enough, because the original was PHP 5.2 compatible.

  * The library is now fully namespaced and compatible with PSR-0. The top level
    namespace is `Resque`.

In practice, these difficulties meant php-resque was almost unextendable in its
current state, and required major refactoring.

## Other Changes

* The events system has been removed. There is now little need for it.
  It seems like the events system was just a workaround due to the poor 
  extensibility of the Worker class. The library should allow you to extend any 
  class you like, and no method should be too long or arduous to move into a
  subclass.
  
## Requirements

* PHP 5.3+
* Redis 2.2+

## Jobs ##

### Queueing Jobs ###

Jobs are queued as follows:

    use Resque\Resque;
    use Predis\Client;
    
    $resque = new Resque(new Client());
    $resque->enqueue('default_queue', 'My_Job', array('foo' => 'bar'));

<!--
### Defining Jobs ###

Each job should be in it's own class, and include a `perform` method.

	class My_Job
	{
		public function perform()
		{
			// Work work work
			echo $this->args['name'];
		}
	}

When the job is run, the class will be instantiated and any arguments
will be set as an array on the instantiated object, and are accessible
via `$this->args`.

Any exception thrown by a job will result in the job failing - be
careful here and make sure you handle the exceptions that shouldn't
result in a job failing.

Jobs can also have `setUp` and `tearDown` methods. If a `setUp` method
is defined, it will be called before the `perform` method is run.
The `tearDown` method if defined, will be called after the job finishes.

	class My_Job
	{
		public function setUp()
		{
			// ... Set up environment for this job
		}
		
		public function perform()
		{
			// .. Run job
		}
		
		public function tearDown()
		{
			// ... Remove environment for this job
		}
	}

### Tracking Job Statuses ###

php-resque has the ability to perform basic status tracking of a queued
job. The status information will allow you to check if a job is in the
queue, currently being run, has finished, or failed.

To track the status of a job, pass `true` as the fourth argument to
`Resque::enqueue`. A token used for tracking the job status will be
returned:

	$token = Resque::enqueue('default', 'My_Job', $args, true);
	echo $token;

To fetch the status of a job:

	$status = new Resque_Job_Status($token);
	echo $status->get(); // Outputs the status

Job statuses are defined as constants in the `Resque_Job_Status` class.
Valid statuses include:

* `Resque_Job_Status::STATUS_WAITING` - Job is still queued
* `Resque_Job_Status::STATUS_RUNNING` - Job is currently running
* `Resque_Job_Status::STATUS_FAILED` - Job has failed
* `Resque_Job_Status::STATUS_COMPLETE` - Job is complete
* `false` - Failed to fetch the status - is the token valid?

Statuses are available for up to 24 hours after a job has completed
or failed, and are then automatically expired. A status can also
forcefully be expired by calling the `stop()` method on a status
class.

## Workers ##

Workers work in the exact same way as the Ruby workers. For complete
documentation on workers, see the original documentation.

A basic "up-and-running" resque.php file is included that sets up a
running worker environment is included in the root directory.

The exception to the similarities with the Ruby version of resque is
how a worker is initially setup. To work under all environments,
not having a single environment such as with Ruby, the PHP port makes
*no* assumptions about your setup.

To start a worker, it's very similar to the Ruby version:

    $ QUEUE=file_serve php resque.php

It's your responsibility to tell the worker which file to include to get
your application underway. You do so by setting the `APP_INCLUDE` environment
variable:

   $ QUEUE=file_serve APP_INCLUDE=../application/init.php php resque.php

Getting your application underway also includes telling the worker your job
classes, by means of either an autoloader or including them.

### Logging ###

The port supports the same environment variables for logging to STDOUT.
Setting `VERBOSE` will print basic debugging information and `VVERBOSE`
will print detailed information.

    $ VERBOSE QUEUE=file_serve php resque.php
    $ VVERBOSE QUEUE=file_serve php resque.php

### Priorities and Queue Lists ###

Similarly, priority and queue list functionality works exactly
the same as the Ruby workers. Multiple queues should be separated with
a comma, and the order that they're supplied in is the order that they're
checked in.

As per the original example:

	$ QUEUE=file_serve,warm_cache php resque.php

The `file_serve` queue will always be checked for new jobs on each
iteration before the `warm_cache` queue is checked.

### Running All Queues ###

All queues are supported in the same manner and processed in alphabetical
order:

    $ QUEUE=* php resque.php

### Running Multiple Workers ###

Multiple workers ca be launched and automatically worked by supplying
the `COUNT` environment variable:

	$ COUNT=5 php resque.php

### Forking ###

Similarly to the Ruby versions, supported platforms will immediately
fork after picking up a job. The forked child will exit as soon as
the job finishes.

The difference with php-resque is that if a forked child does not
exit nicely (PHP error or such), php-resque will automatically fail
the job.

### Signals ###

Signals also work on supported platforms exactly as in the Ruby
version of Resque:

* `QUIT` - Wait for child to finish processing then exit
* `TERM` / `INT` - Immediately kill child then exit
* `USR1` - Immediately kill child but don't exit
* `USR2` - Pause worker, no new jobs will be processed
* `CONT` - Resume worker.

### Process Titles/Statuses ###

The Ruby version of Resque has a nifty feature whereby the process
title of the worker is updated to indicate what the worker is doing,
and any forked children also set their process title with the job
being run. This helps identify running processes on the server and
their resque status.

**PHP does not have this functionality by default.**

A PECL module (<http://pecl.php.net/package/proctitle>) exists that
adds this funcitonality to PHP, so if you'd like process titles updated,
install the PECL module as well. php-resque will detect and use it.

-->

## Contributors ##

### php-resque
* chrisboulton
* thedotedge
* hobodave
* scraton
* KevBurnsJr
* jmathai
* dceballos
