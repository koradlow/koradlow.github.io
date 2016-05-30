---
title: "Waiting for multiple asynchronous events with timeouts"
excerpt: "Python events can be used to inform about asynchronous events in multiple threads. Sometimes
you come across a situation where you need to wait for multiple events to happen"
category: Asynchronous
tags: [Python, Asynchronous, Multithreading, Events, Synchronization]
---

A program written to interact with hardware that exhibits non-deterministic
asynchronous behavior will be improved greatly by exploiting parallelization
for independent system components.

A XYZ-Stage can be used as a good example for this - there are three Axis-Controllers
that operate independently. To move to a target position the simple sequential approach
could move each Axis and wait for it to reach the destination before the next movement is 
triggered. This has the advantage that it's very easy to implement and for each axis
the destination is either reached or a timeout event happens which can be handled directly.

<h3> Sequential execution, sequential wait</h3>
```python
#sequential code
axis_controllers = {axis:AxisController() for axis in ['x','y','z']} 

# move to target position (sequential)
target_postion = {'x':100, 'y': 650', z:'0'}
for axis in axis_controllers:
    axis_controllers[axis].moveToPosition(target_position[axis])
    # wait for 10 seconds to 
    if not axis_controllers[axis].targetReachedEvent.wait(10):
        print("Moving", axis, "to target position", target_position[axis], "failed with timeout")
        raise TimeoutError(axis)
```

  - The good: easy to determine which axis failed, concise code  
  - The bad: prevents exploiting the indepentent nature of the axis movement

<h3> Parallel execution, sequential wait</h3>
By implementing the AxisController class in a way that each instance runs 
in its own thread it is easy to move multiple axis at the same time. 

The problem that arises by the aproach is to implement a synchronization point 
after triggering the movement of multiple axis.
The next operation can only be started once all of the axis have completed 
their movement. 

In this case it is possible to use a sequential wait approach, because at the point
of synchronization it is necessary to wait for the last event to happen and 
the order in which events are set and detected does not matter.

```python
# move to target position (parallel)
for axis in axis_controllers:
    axis_controllers[axis].moveToPosition(target_position[axis])

# wait for all events (sequential)
results = {}
for axis in axis_controllers:
    results[axis] = axis_controllers[axis].targetReachedEvent.wait(10)

# check results
if False in results.values():
    # Generate a list of the axis that failed
    failed = [axis for axis in results if results[axis] is False]
    print("Moving", str(failed), "failed with timeout")
    raise TimeoutError(str(failed))
``` 

  - The good: easy to implement, little overhead
  - The bad: waiting for multiple asynchronous events from independent threads is
  done in a sequential way. This is not bad per se but it has limitations when
  the use case becomes more complex. This aproach for example forces the error
  handling to be sequential too

<h3>Parallel execution, parallel wait</h3>
A solution can be to perform a parallel wait for the events. To do so the same number
of waiting threads as events needs to be created. These threads wait for the event, 
possibly react to the error condition, return the result and terminate themselves.

Since thread creation has only a small overhead it doesn't have a negative performance impact 
(but do not use this for networking or situations where you need to wait for >>100 threads, there are better solutions
for this).

In the case of this example it is only the *event.wait()* function which is
executed in the thread. But in a real exemple you would probably implement a
custom function that performs additional steps like error handling and 
logging besides waiting for the event.

```python
from multiprocessing.pool import ThreadPool

# move to target position (parallel)
for axis in axis_controllers:
    axis_controllers[axis].moveToPosition(target_position[axis])
    
# wait for all events (parallel)
with ThreadPool(processes = len(self.axis_controllers)) as waiters:
    # use the map function to link the execution of the wait for 
    # targetReachedEvent function to the threads of the pool.
    # The result will be an list of booleans (False for timeout)
    result = waiters.map(lambda axis : axis.targetReachedEvent.wait(10),
                         self.axis_controllers.values())
    
    # check results 
    if False in result:
        # Generate a list of the axis that failed
        failed = [k for k,v in self.axis_controllers.items() if not v.targetReachedEvent.isSet()]
        print("Moving", str(failed), "failed with timeout")
        raise TimeoutError(str(failed)
```

  - The good: parallel waiting for multiple events, allows complex and efficient
  error handling
  - The bad: requires creation of one thread per event
