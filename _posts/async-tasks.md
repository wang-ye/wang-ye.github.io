I was setting up the celery for a hobby project, and refreshed some knowledge about setting up async task queues.

## Async Tasks VS Web APIs
Surprisingly, the two shares a lot in common, although at first glance they look quite different. The underlying reason - tasks fails frequently and unexpectedly, and we need some common approaches to achieve our goal.

Both need to consider idempotency, in case some errors happen in the middle and the task/request fails and we need to retry. Also, atomicity is also an important requirement for both to eliminate inconsistent states. Retrying, rate limit and timeout are also common considerations for both types.

## Do not Pass Complex Object to Async Tasks
Often times, the data is passed to message queue with serializations, during which failures can happen (object not serializable) or information can be lost. So pass simple data!

## Monitoring
Log frequently, and use tools to communicate failures and errors.

## What If You Need to Pull Async Results