---
layout: post
title: "RIP flatMapGroupsWithState"
date: 2025-02-27
summary: "The new transformWithState API is here and you'd be crazy not to try it."
description: "You'd be crazy not to try the new transformWithState API for your Spark Streaming jobs with arbitrary state"
image: /2025/transform-with-state/image.png
tags: [databricks]
---

The new `transformWithState` API is now available on [Databricks Runtime 16.2](https://www.databricks.com/blog/introducing-transformwithstate-apache-sparktm-structured-streaming) and you'd be crazy not to [try it](https://docs.databricks.com/aws/en/stateful-applications/).

The improvements over the old `flatMapGroupsWithState` and `applyInPandasWithState` approaches to handling custom state are compelling from an API perspective and **a total no brainer for performance**.

Here's a stab at migrating a simple PySpark Streaming job to use `transformWithState` with some inline commentary that highlight the relevant API improvements and performance implications.

## Why applyInPandasWithState and flatMapGroupsWithState suck

Here's a simple streaming operator written with the old `applyInPandasWithState` API. It's job is to aggregate events for a fleet of delivery vehicles and write them out to a table only when the vehicle sends a `delivered` event.

```python
# our aggregating function that takes:
#    - key (grouping key)
#    - pdf_iter (rows that belong to the key)
#    - state (arbitrary state object for the key)
def stateful_accumulate(key, pdf_iter, state: GroupState):
    if state.exists:
        stored_tuple = state.get
        current_events = stored_tuple[0] if stored_tuple[0] else []
    else:
        current_events = []

    for pdf in pdf_iter:
        current_events.extend(pdf.to_dict("records"))

    if any(e["type"] == "delivered" for e in current_events):
        yield pd.DataFrame([{
            "orderid": key[0],
            "events":  current_events
        }])
        state.remove()
    else:
        state.update((current_events,))

# we read from an append only table
df = spark.readStream.table("orders.default.events")

# define our stream to group by orderid and apply the above stateful_accumulate function
aggregateFleetEvents = df.groupBy("orderid").applyInPandasWithState(
    func=stateful_accumulate,
    outputStructType=output_schema,
    stateStructType=state_schema,
    outputMode="append",
    timeoutConf="NoTimeout"
)

# flush our stream to the target table
aggregateFleetEvents.writeStream.toTable("orders.default.drives")
```

The logic in `stateful_accumulate` works fine, but there are some issues...

### No explicit state lifecycle management

The first issue is that we have no state lifecycle separation. The first few lines are mostly concerned with correctly initializing the state object because it's *undefined to start*. This looks tolerable for such a simple job but if this was even a little bit more complex the initialization would be a major sore (imagine migrating a job with existing state).

```python
if state.exists:
    stored_tuple = state.get
    current_events = stored_tuple[0] if stored_tuple[0] else []
else:
    current_events = []
```

### Single state object

The second issue is that we have to handle the entire state object at once *and* rewrite it entirely. This is subtle for small jobs, but a complete deal breaker if you need to scale. Rewriting the entire state every time new events appear simply doesn't make any sense, especially once we need to track multiple logical states per key.

```python
# add new events to `current_events`
for pdf in pdf_iter:
    current_events.extend(pdf.to_dict("records"))

# ...now current_events = old state + new state

if any(e["type"] == "delivered" for e in current_events):
    yield pd.DataFrame([{
        "orderid": key[0],
        "events":  current_events
    }])
    state.remove()
else:
    # 'update' the state aka overwrite the ENTIRE state X_X
    state.update((current_events,))
```

## Why transformWithState is better than flatMapGroupsWithState and applyInPandasWithState

Here's the same job rewritten to use `transformWithState`:

```python
class DeliveryFleetEventAggregator(StatefulProcessor):
    def init(self, handle: StatefulProcessorHandle) -> None:
        self.list_state = handle.getListState(stateName="listState", schema=event_struct)

    def handleInputRows(self, key, rows, timerValues) -> Iterator[pd.DataFrame]:
        should_flush = False

        for pdf in rows:
            self.list_state.appendList(pdf)
            if 'delivered' in pdf['type'].values:
                should_flush = True

        if should_flush:
            yield pd.DataFrame([{
                "orderid": key[0],
                "events":  list(self.list_state.get())
            }])
            self.list_state.clear()

    def close(self):
        super().close()

aggregateFleetEvents = df.groupBy("orderid").transformWithStateInPandas(
    statefulProcessor=DeliveryFleetEventAggregator(),
    outputStructType=output_schema,
    outputMode="append",
    timeMode="none"
)
```

Lifecycle methods `init` and `close` separate setup and teardown concerns from the main processing logic. This is a major improvement in terms of readability and maintainability.

```python
def init(self, handle: StatefulProcessorHandle) -> None:
    self.list_state = handle.getListState(stateName="listState", schema=event_struct)
```

We have a separate logical state `self.list_state` that we initialize with `handle.getListState`. This is part of the new composite types capability that also includes `ValueState` and `MapState`. This apparently small difference has major implications. [We can work with multiple separate state objects independently, as needed, *and* we get a massive performance boost as a consequence.](https://docs.databricks.com/aws/en/stateful-applications?language=Python#custom-state-types) The new version only needs to `appendList` while taking a single pass over the input rows.

```python
def handleInputRows(self, key, rows, timerValues) -> Iterator[pd.DataFrame]:
    should_flush = False

    for pdf in rows:
        self.list_state.appendList(pdf)
        if 'delivered' in pdf['type'].values:
            should_flush = True

    if should_flush:
        yield pd.DataFrame([{
            "orderid": key[0],
            "events":  list(self.list_state.get())
        }])
        self.list_state.clear()
```

We don't have any complex timing or expiration needs in this simple job but `transformWithState` supports some awesome features like defining timers for custom logic and setting TTL for automatic state eviction, giving you [fine-grained control over how and when your state data is updated or removed.](https://docs.databricks.com/aws/en/stateful-applications/#program-timed-events)

You'd be crazy not to seriously consider rewriting your old jobs with this new API.
