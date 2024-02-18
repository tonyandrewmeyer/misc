# Handling Temporary Errors in Charms

There are a few types of errors that can occur while a charm is handling an event that are expected to resolve within a fairly short timeframe without human intervention, and can occur at any point during the event handling, despite Juju's efforts to provide a stable snapshot of the model state.

Common examples are an issue communicating with Pebble, or communicating directly with Kubernetes (with [LightKube](https://lightkube.readthedocs.io/en/latest/) for example), or more generally working with any workload or external API.

Let's use Pebble as an example, where we want to do a simple replan of the workload:

```python
container.replan()
```

If Pebble can't be reached, then this will raise a `ConnectionError` and the charm will enter an error state. In some cases, that might be the right thing to do! However, we'd rather avoid the error state unless human intervention is required to fix the problem, and we're confident that in most cases this is actually just a temporary issue and trying again after a short time will solve the problem.

A common pattern is to check Pebble first, and defer if it's not available:

```python
if container.can_connect():
    container.replan()
else:
    event.defer()
    return
```

A problem with this approach is that `can_connect()` doesn't guarantee that Pebble will be reachable the next time the charm tries to communicate with it - just that it was at that point in time. This means that we've introduced a race condition where the `can_connect()` can succeed but the `replan()` fails, and we again end up in an error state. Let's try to fix that:

```python
if container.can_connect():
    try:
        container.replan()
    except ops.pebble.ConnectionError:
        event.defer()
        return
else:
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
    except ops.pebble.ConnectionError:
        time.sleep(PEBBLE_RETRY_DELAY ** attempt)
    else:
        break 
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
    except ops.pebble.ConnectionError:
        time.sleep(PEBBLE_RETRY_DELAY ** attempt)
    else:
        break 
else:
    logger.error("Unable to reach Pebble after %.1f seconds.", too_lazy_to_do_the_maths)
    raise SystemExit(1)
```

The only time we aren't expecting Pebble to be available is before we get the container's `PebbleReadyEvent`. If our code is running in an event handler that's potentially before then (see [a charm's lifecycle](https://juju.is/docs/sdk/charm-lifecycle)), we'd want to have the work be handled by a later event (or defer). If the code is running in a handler than could occur in the "setup" as well as "operation" or "teardown" phases, then we might prefer to always avoid the error state.

Let's put it all together into a simple, reusable, method:

.. TODO: I don't love how this is. A decorator or context manager is likely best. Maybe the handling of defer, return etc is too much? A class with a __getattr__ could make the call look much more like the normal version.

```python
def tolerant_pebble(event, container, method, *args, **kwargs):
    for attempt in range(MAX_PEBBLE_ATTEMPTS):
        try:
            getattr(container, method)(*args, **kwargs)
        except ops.pebble.ConnectionError:
            time.sleep(PEBBLE_RETRY_DELAY ** attempt)
        else:
            break
    else:
        if isinstance(event, ops.ActionEvent):
            event.fail("Pebble is unreachable.")
            raise SystemExit(0)
        elif not hasattr(event, "defer"):
            logger.error("Cannot connect to Pebble.")
            raise SystemExit(1)  # Go into error state.
        elif isinstance(event, (ops.StartEvent, ops.StorageAttachedEvent, ops.InstallEvent, ops.RelationCreatedEvent, ops.LeaderElectedEvent)):
            logger.info("Unable to connect to Pebble.")
            return  # Our charm knows that a later event will complete the work.
        else: 
            event.defer()
```

And use that for our `replan`:

```python
tolerant_pebble(event, container, "replan")
```
