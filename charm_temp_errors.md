# Handling Temporary Errors in Charms

There are a few types of errors that can occur while a charm is handling an event that are expected to resolve within a fairly short timeframe without human intervention, and can occur at any point during the event handling, despite Juju's efforts to provide a stable snapshot of the model state.

Common examples are an issue communicating with Pebble, or communicating directly with Kubernetes (with [LightKube](https://lightkube.readthedocs.io/en/latest/) for example), or more generally working with any workload or external API. A typical example is:

```python
def _service_is_running(self):
    if not self.container.can_connect():
        return False
    try:
        service = self.container.get_service(self._service_name)
    except (ops.ModelError, ops.pebble.ConnectionError):
        return False
    return service.is_running()
``` 

Let's take a dive into this pattern. We'll use Pebble as an example, where we want to do a simple replan of the workload:

```python
container.replan()
```

If Pebble can't be reached, then this will raise a `ConnectionError` and the charm will enter an error state. In some cases, that might be the right thing to do! However, we'd rather avoid the error state unless human intervention is required to fix the problem, and we're confident that in most cases this is actually just a temporary issue and trying again after a short time will solve the problem.

A common pattern is to check Pebble first, and defer if it's not available:

> :warning: **Don't use this code** - there's a better alternative below!

```python
if not container.can_connect():
    event.defer()
    return
container.replan()
```

A problem with this approach is that `can_connect()` doesn't guarantee that Pebble will be reachable the next time the charm tries to communicate with it - just that it was at that point in time. This means that we've introduced a race condition where the `can_connect()` can succeed but the `replan()` fails, and we again end up in an error state. Let's try to fix that:

> :warning: **Don't use this code** - there's a better alternative below!

```python
if not container.can_connect():
    event.defer()
    return
try:
    container.replan()
except ops.pebble.ConnectionError:
    event.defer()
    return
```

However, it's clear now that the `can_connect()` call isn't doing anything useful - it's pinging Pebble but most of the time we expect Pebble will be available (so it's not required) and when it's not we already have the `try`/`except` to handle that. Let's make an attempt to simplify the code:

```python
try:
    container.replan()
except ops.pebble.ConnectionError:
    event.defer()
    return
```

We're pretty sure that when we do get this failure, it's a temporary problem - perhaps there's high load, or some temporary networking issues. One issue with using `defer()` is that we don't know, or have any control over, when the event handler will be retried. `defer()` also restarts the entire handler, so any work we've already done will get repeated, which is wasteful. Let's adjust the code so that we give Pebble a few chances:

```python
for attempt in range(MAX_PEBBLE_RETRIES):
    try:
        container.replan()
        break
    except ops.pebble.ConnectionError:
        time.sleep(PEBBLE_RETRY_DELAY * attempt)
else:
    event.defer()
    return
```

[tenacity](https://tenacity.readthedocs.io/en/latest/) is a really nice library providing decorators and context managers that handle different backoff mechanisms, attempt limits, and so forth - but if you want to keep your dependencies to a minimum, it's simple enough to have something like the above as reusable code within your charm.

At this point, if Pebble is responding, then we'll get our `replan()` executed in essentially the same time as if we just called it (certainly faster than if we guarded with `can_connect()`), and if there's an issue with Pebble, we'll retry until we're confident that the problem isn't going to resolve itself within a timeframe we're comfortable with (perhaps a couple of minutes - remembering that the Charm won't handle other events while it's sleeping between retries). If Pebble is failing longer than that, do we really want to defer? That would be appropriate if we believe that the issue will resolve itself, but if Pebble has been unreachable this long, it's likely that someone needs to investigate what's happening - ie. manual intervention is required, and a Juju error state is appropriate in that case. Let's adjust the code again:

```python
for attempt in range(MAX_PEBBLE_RETRIES):
    try:
        container.replan()
        break
    except ops.pebble.ConnectionError:
        time.sleep(PEBBLE_RETRY_DELAY * attempt)
else:
    raise RuntimeError("Unable to reach Pebble")
```

The only time we should expect Pebble to be unavailable is during the setup phase of [a charm's lifecycle](https://juju.is/docs/sdk/charm-lifecycle). During this phase, we would generally expect another event to occur that would also trigger the work we're doing and so simply exiting (with success) is often the best choice.

Let's put this together into a simple, reusable, method with some example use:


```python
def retry_pebble(func):
    @functools.wrap
    def wrapped(*args, **kwargs)
        attempt = 0
        while True:
            attempt += 1
            logger.debug("Attempt %d at calling %s.", attempt, func.name)
            try:
                return func(*args, **kwargs) 
            except ops.pebble.ConnectionError:
                if attempt >= MAX_PEBBLE_RETRIES:
                    raise
                time.sleep(PEBBLE_RETRY_DELAY * attempt)
    return wrapped


class MyCharm(ops.CharmBase):
    def __init__(self, framework):
        super().__init__(framework)
        for container in self.containers:
            container.replan = retry_pebble(container.replan)
            # Also wrap the other container methods.
        ...
```

And use that for our `replan` in a few situations, as well as our friend `can_connect`:

```python
def _on_install(self, event):
    try:
        self.containers[CONTAINER_NAME].replan()
    except ops.pebble.ConnectionError:
        logger.info("Unable to connect to Pebble - setup will continue later.")

def _on_pebble_ready(self, event):
    try:
        event.workload.replan()
    except ops.pebble.ConnectionError:
        logger.warning("Unable to connect to Pebble.")
        event.defer()

def _on_leader_changed(self, event):
    try:
        self.containers[CONTAINER_NAME].replan()
    except ops.pebble.ConnectionError:
        raise RuntimeError("Pebble appears to be down - please investigate!")

def _on_collect_status(self, event):
    if not self.containers[CONTAINER_NAME].can_connect():
        event.set_status(ops.WaitingStatus("Waiting for Pebble."))
    ...
```
