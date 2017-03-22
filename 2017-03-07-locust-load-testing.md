---
layout: post
title:  "Understanding Locust Load Testing"
date:   2017-03-07 22:49:03 -0800
---
[Locust](http://locust.io/) is a powerful, yet easy to understand Python framework for load testing. It has several nice features:

1. Locust has a flexible loadtesting task definition language - Python! Instead of writing XML or JSON configuration, Locust defines user and behaviors using plain Python directly.
2. By utilizing Gevent, Locust can issue concurrent requests easily. Besides, it supports master/slave mode, making it easily scalable to multiple machines.
3. Locust provides a clean statistics collection framework, with a UI showing the testing statistics and displays to the users.
4. Testers can easily create a client and test any request-response system with Locust's scheduling magic.

These four characteristics are also key for a load testing framework. In this blog, I will dig into Locust source code and investigate how to implement a load testing framework.

## Understanding LoadTest Task Configuration
Rather than having a JSON config file, Locust allows users to write tests in Python. In Locust, there are several key concepts:

1. Task Set: The test suite we want to perform.
2. Locust: a container of task set, also the scheduling unit for performing load testing.
3. HTTPLocust: Subclass of Locust. It simulates a user performing HTTP requests to a given server.

End user writes a Python file called *locustfile* to define the behaviors of the loadtests. Inside the file contains at least one class inherited from Locust. The subclass usually contains a task_set attribute, which is a essentially a subclass of TaskSet. The Taskset subclass defines a list of actions we want to perform. As a concrete example, a *locustfile* can be as follows:

```python
from locust import Locust, TaskSet, task

class MyTaskSet(TaskSet):
    @task
    def index(self):
        # client is an instance of HttpSession. It can take care of the cookies among HTTP requests.
        self.client.get("/")

    @task(3)
    def status(self):
        self.client.get("/status")

class MyLocust(HttpLocust):
    host = 'http://www.google.com'
    task_set = MyTaskSet
```

### Diving Into TaskSet
How does TaskSet know the behaviors it want to perform? Through the use of decorator and metaclass, TaskSet has an *tasks* attribute implicitly generated. The code below shows the trick.

```python
def task(weight=1):
    def decorator_func(func):
        func.locust_task_weight = weight
        return func
    # task method assigns a locust_task_weight for the method it decorates.
    if callable(weight):
        func = weight
        weight = 1
        return decorator_func(func)
    else:
        return decorator_func


class TaskSetMeta(type):
    """
    Meta class for the main Locust class. It's used to allow Locust classes to specify task execution 
    ratio using an {task:int} dict, or a [(task0,int), ..., (taskN,int)] list.
    """
    def __new__(mcs, classname, bases, classDict):
        new_tasks = []
        ...        
        # Find methods containing locust_task_weight attribute, and add them
        # to a tasks method based on weight.
        for item in six.itervalues(classDict):
            if hasattr(item, "locust_task_weight"):
                for i in xrange(0, item.locust_task_weight):
                    new_tasks.append(item)
        
        # This is the implicitly generated tasks.
        classDict["tasks"] = new_tasks
        
        return type.__new__(mcs, classname, bases, classDict)

# Through meta programming, TaskSet now has a method tasks returning
# a list of methods.
@six.add_metaclass(TaskSetMeta)
class TaskSet(object):
    ...
```

TaskSet has a *run* method, which randomly picks a task from *tasks*, and start running it. All the tasks will be running under gevent's greenlet. We will discuss this later.

### Locust Loadtest Import
As we discussed before, tests are in plain Python file. How does Locust understand the tests? The tests are imported dynamically through the magic *__import__* method. By importing the locustfile and keeping only the Locust subclasses, Locust has successfully loaded the configuration Python files.

```python
def is_locust():
    """
    Takes (name, object) tuple, returns True if it's a public Locust subclass.
    """
    name, item = tup
    return bool(
        inspect.isclass(item)
        # and issubclass(item, Locust)
        and hasattr(item, "task_set")
        and getattr(item, "task_set")
        and not name.startswith('_')
    )

def load_locustfile(path):
    directory, locustfile = os.path.split(path)
    
    imported = __import__(os.path.splitext(locustfile)[0])

    # Return our two-tuple
    locusts = dict(filter(is_locust, vars(imported).items()))
    return imported.__doc__, locusts
```

### Task Inspection
Locust also provides some task inspection utilities, which shows the task structures and the weights of the different tasks. Locust provides an option called get_task_ratio_dict to conduct the inspection:

```python
# tasks can be either a list of Locust subclasses or 
# a list of tasks/task_sets. This method computes the task ratio based on
# the corresponding weight of a Locust/task_set/task.
def get_task_ratio_dict(tasks, total=False, parent_ratio=1.0):
    """
    Return a dict containing task execution ratio info
    """
    # divisor is the total weight of the tasks.
    if hasattr(tasks[0], 'weight'):
        divisor = sum(t.weight for t in tasks)
    else:
        divisor = len(tasks) / parent_ratio
    # ratio contains the per task weight.
    ratio = {}
    for task in tasks:
        ratio.setdefault(task, 0)
        ratio[task] += task.weight if hasattr(task, 'weight') else 1

    # get percentage
    ratio_percent = dict((k, float(v) / divisor) for k, v in six.iteritems(ratio))

    task_dict = {}
    for locust, ratio in six.iteritems(ratio_percent):
        d = {"ratio":ratio}
        if inspect.isclass(locust):
            if issubclass(locust, Locust):
                T = locust.task_set.tasks
            # There can be nested task_sets. Recursively get the weight of the tasks in the task_set.
            elif issubclass(locust, TaskSet):
                T = locust.tasks
            if total:
                d["tasks"] = get_task_ratio_dict(T, total, ratio)
            else:
                d["tasks"] = get_task_ratio_dict(T, total)
        
        task_dict[locust.__name__] = d
```

Essentially, this get_task_ratio_dict would traverse each level of task sets and compute the corresponding test weights. It is a powerful tool to understand what the test structures are.

## Issuing Concurrent Requests
Locust defines **LocustRunner** to schedule the Locust tasks.
End user provides the number of locusts to spawn and the rate of the hatching.

LocustRunner first computes the number of locusts we want to spawn based on the weights. Then, with the help of gevent groups, it emits the locust for running. gevent and greenlet here provide the performance improvement for those IO-extensive applications. The code below shows how the hatching is done.

```python
class Locust
    # Locust class defines the *run* method as follows
    def run(self):
        try:
            # Initialize a TaskSet instance and call its run method. 
            # This essentially run all the tasks inside the task sets based
            # on task weights (counts).
            self.task_set(self).run()
        except StopLocust:
            pass
        except (RescheduleTask, RescheduleTaskImmediately) as e:
            ...

# Group is gevent data structure to utilize Python multiprocessing lib.
locusts = Group()

def hatch():
    sleep_time = 1.0 / self.hatch_rate
    while True:
        # bucket contains the locusts for hatching.
        if not bucket:
            return
        locust = bucket.pop(random.randint(0, len(bucket)-1))
        def start_locust(_):
            try:
                # Calling the run method of the Locust subclass.
                locust().run()
            except GreenletExit:
                pass
        new_locust = self.locusts.spawn(start_locust, locust)
        gevent.sleep(sleep_time) 

hatch()
```

### Running Across Multiple Machines
Locust also supports a distributed mode with master/slave structure.
Through message passing, master node distributes the tasks among the available slave node. Slaves execute the tasks and report the task statistics back to master. This communication between master and slave is achieved by [*zmq.green*]().

```python
class MasterLocustRunner(DistributedLocustRunner):
    # Master distributes the tasks to slaves.
    def start_hatching(self, locust_count, hatch_rate):
        num_slaves = len(self.clients.ready) + len(self.clients.running)

        self.num_clients = locust_count
        slave_num_clients = locust_count // (num_slaves or 1)
        slave_hatch_rate = float(hatch_rate) / (num_slaves or 1)
        remaining = locust_count % num_slaves
        
        for client in six.itervalues(self.clients):
            data = {
                "hatch_rate":slave_hatch_rate,
                "num_clients":slave_num_clients,
                "num_requests": self.num_requests,
                "host":self.host,
                "stop_timeout":None
            }

            if remaining > 0:
                data["num_clients"] += 1
                remaining -= 1

            # Distribute tasks to available clients.
            self.server.send(Message("hatch", data, None))

    # Handle the response from the client side.
    def client_listener(self):
        while True:
            msg = self.server.recv()
            if msg.type == "client_ready":
                id = msg.node_id
                self.clients[id] = SlaveNode(id)
            elif msg.type == "stats":
                events.slave_report.fire(client_id=msg.node_id, data=msg.data)
            elif msg.type == "hatching":
                self.clients[msg.node_id].state = STATE_HATCHING
            elif msg.type == "hatch_complete":
                self.clients[msg.node_id].state = STATE_RUNNING
                self.clients[msg.node_id].user_count = msg.data["count"]
                if len(self.clients.hatching) == 0:
                    count = sum(c.user_count for c in six.itervalues(self.clients))
                    events.hatch_complete.fire(user_count=count)
            elif msg.type == "quit":
                if msg.node_id in self.clients:
                    del self.clients[msg.node_id]
```

Master runner maintains a map of the available slave nodes, and dynamically updates it based on the communication with the clients. It distributes the tasks to client with start_hatching method, and listen to client response in client_listener method. Once client message arrives, master will need to update the slave clients mapping and collect the statistics accordingly.

```python
# Slave does the actual heavy lifting.
class SlaveLocustRunner(DistributedLocustRunner):
    def __init__(self, *args, **kwargs):
        # Once slave node is up, it sends a client_ready message to master so that master is aware of this slave node and can send tasks to it.
        self.client = rpc.Client(self.master_host, self.master_port)
        self.client.send(Message("client_ready", None, self.client_id))

        self.greenlet = Group()
        # worker is in charge of the reaal hatching.
        self.greenlet.spawn(self.worker).link_exception(callback=self.noop)
        # Periodically send the slave stats to master.
        self.greenlet.spawn(self.stats_reporter).link_exception(callback=self.noop)
       
    def worker(self):
        while True:
            msg = self.client.recv()
            if msg.type == "hatch":
                self.client.send(Message("hatching", None, self.client_id))
                job = msg.data
                self.hatch_rate = job["hatch_rate"]
                #self.num_clients = job["num_clients"]
                self.num_requests = job["num_requests"]
                self.host = job["host"]
                self.hatching_greenlet = gevent.spawn(lambda: self.start_hatching(locust_count=job["num_clients"], hatch_rate=job["hatch_rate"]))
            elif msg.type == "stop":
                ...

    def stats_reporter(self):
        while True:
            data = {}
            # Collect all the stats in data dict, and send it to master
            # periodically.
            events.report_to_master.fire(client_id=self.client_id, data=data)
            try:
                self.client.send(Message("stats", data, self.client_id))
            except:
                logger.error("Connection lost to master server. Aborting...")
                break
            
            gevent.sleep(SLAVE_REPORT_INTERVAL)
```

In SlaveLocustRunner, the worker does the real job of running the load testing. It also sends the slave report to master periodically.

## Collecting statistics and Customized Logging

Locust has a system to collect the test statistics. The statistics are delivered to the testers either via user interface or the console logs. How does Locust track the different actions and collect the statistics elegantly? Also, Locust override the system print behavior to also log some essential information. We will also investigate the implementation of this.

### Statistics Collection
Locust provides *EventHook*. An end user provides corresponding callbacks/handlers which are triggered after certain events happen. The statistics collection also happen as a callback inside an event hook. Add custom handlers is also easy - user only needs to append it to event hook.

```python
class EventHook(object):
    """
    Simple event class used to provide hooks for different types of events in Locust.

    Here's how to use the EventHook class::

        my_event = EventHook()
        def on_my_event(a, b, **kw):
            print "Event was fired with arguments: %s, %s" % (a, b)
        my_event += on_my_event
        my_event.fire(a="foo", b="bar")
    """
    def __init__(self):
        self._handlers = []

    def __iadd__(self, handler):
        self._handlers.append(handler)
        return self

    def __isub__(self, handler):
        self._handlers.remove(handler)
        return self

    def fire(self, **kwargs):
        for handler in self._handlers:
            handler(**kwargs)

request_success = EventHook()

# The callbacks associated with an event hook.
def on_request_success(request_type, name, response_time, response_length):
    # Global_stats is a RequestStats object, essentially a hash holding different stats.
    if global_stats.max_requests is not None and (global_stats.num_requests + global_stats.num_failures) >= global_stats.max_requests:
        raise StopLocust("Maximum number of requests reached")
    global_stats.get(name, request_type).log(response_time, response_length) 

events.request_success += on_request_success
```

When a request succeeds, a success event is triggered and the stats are thus collected.

```python
events.request_success.fire(
    request_type=self.locust_request_meta["method"],
    name=self.locust_request_meta["name"],
    response_time=self.locust_request_meta["response_time"],
    response_length=self.locust_request_meta["content_size"],
)
```

The statistics is thus updated in global_stats every time an request_success event fires. The user interface will take the global_stats and display to users.

### Logging Setup
User can print stuff inside the user-provided Locust subclasses. Locust overrides the sys.stdout and sys.stderr for consistent printing results.

```python
host = socket.gethostname()
def setup_logging(loglevel, logfile):
    numeric_level = getattr(logging, loglevel.upper(), None)
    log_format = "[%(asctime)s] {0}/%(levelname)s/%(name)s: %(message)s".format(host)
    logging.basicConfig(level=numeric_level, filename=logfile, format=log_format)
    # Override the stdout to also print with a logging object.
    sys.stdout = StdOutWrapper()

stdout_logger = logging.getLogger("stdout")
class StdOutWrapper(object):
    def write(self, s):
        stdout_logger.info(s.strip())

    def flush(self, *args, **kwargs):
        """No-op for wrapper"""
        pass
```

## Support different protocols
Locust also supports other types of requests. For example, it can load test a rpc-xml server. To do custom load testing, tester only needs to implement a custom client and add proper statistics collection to the client. Locust will take care of the scheduling for you! Examples can be found [here](http://docs.locust.io/en/latest/testing-other-systems.html).

## Nice Tricks
Locust has a socketrpc.py file. It implements a simple master-slave model from scratch, by distributing the tasks to slaves using round-robin. Also, do not forget to use six package if Python compatibility is desired.

## Take Aways
Locust is a powerful tool for loadtesting. Its source code throws lights on building the system for issuing concurrent requests. In this blog, we talked about configuration parsing/understanding, issuing concurrent requests with gevent and greenlet, collecting stats with the help of hook and loadtesting for any request-response system. Please let me know if you have any comments.
