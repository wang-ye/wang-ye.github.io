Datadog supports multiple ways to collect stats. One commonly used way is setting up a datadog agent and receive local stats via UDF packets. However, if you do not want to set up the agent, a direct API call using the ``ThreadStats`` module is also possible. In this post, I would cover the setup of ThreadStats, and discuss some pitfalls I encountered while using the ThreadStats module.

## What is ThreadStats?
[ThreadStats](https://docs.datadoghq.com/developers/faq/data-aggregation-with-dogstatsd-threadstats/) is another way to collect aggregated stats. You can set up thread stats in Python easily.

```python
from datadog import initialize
from datadog import ThreadStats
options = {
    'api_key': YOUR_API_KEY,
    'app_key': YOUR_APP_KEY,
}
initialize(**options)
dd_client = ThreadStats()
dd_client.start()
dd_client.increment('test_metric')
```

That's it! You can start sending metrics to datadog with this simple setup using increment now.

## What Happened In the Async Task
However, I encountered a wired error when trying to send metrics in Celery async task. Here is what I did. I have a celery job sending health checks

```python
@celery.task()
def health_check():
    dd_client.increment("health_check")
    return 'ok'
```

Pretty simple, right? However, the ``health_check`` metrics are never sent to datadog side. What happened? To understand this, we need to know a bit of Celery and ThreadStats internal working.

## Celery Prefork Model
When you start celery using the following command:

```shell
celery --app=your_app ...
```
You actually trigger Celery in multiprocessing mode. Every time a new task comes, the main process would distribute it to a worker process.

## Datadog ThreadStats
The ThreadStats collects stats locally and  periodically flushes them to remote Datadog server. It does so by starting a daemon thread specifically for flushing stats. The default flushing interval is 10 seconds.

```python
# In threadstats/base.py
def _start_flush_thread(self):
    """ Start a thread to flush metrics. """
    from datadog.threadstats.periodic_timer import PeriodicTimer
    # Unrelated code ignored ...
    def flush():
        try:
            log.debug("Flushing metrics in thread")
            self.flush()
        except:
            try:
                log.exception("Error flushing in thread")
            except:
                pass

    log.info("Starting flush thread with interval %s." % self.flush_interval)
    self._flush_thread = PeriodicTimer(self.flush_interval, flush)
    self._flush_thread.start()
```

## What Went Wrong?
The root cause is, inside the worker process, there is no flushing thread running (confirmed from local runs)! It is likely because when copying context from main process to the worker, we cannot copy the running thread :). To send the metrics, you have to call flush manually. Also, the flush method only flush the previous interval, if you want to flush everything in the thread, a call like the following will be necessary.

```python
# ...
dd_client.flush(float('inf'))
```

## Summary
In this post I discussed a disturbing issue about the ThreadStats in async Celery tasks. Digging into the mysterious metrics sending issue, we found that the flush method is not called in worker processes at all, and we have to manually call ``flush`` to send the metrics.