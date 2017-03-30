

flask-base uses redis for running asynchronous jobs. For example, when user registers, flask-base will send an confirmation email like this:

```python
get_queue().enqueue(
    send_email,
    recipient=user.email,
    subject='You Are Invited To Join',
    template='account/email/invite',
    user=user,
    invite_link=invite_link, )
```


Why using Redis?

How does this queuing work?

An extension of flask-rq.