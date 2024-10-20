---
layout: article
description:  "A dive into Django Signals and the auditlog package, how the package utilizes said signals to log all data model changes"
title: "Django Signals, Auditlog and Weakrefs"
date: 2024-10-20
categories:
 - programming
tags: 
    - django
    - python
comments: true
draft: true
---

# Django Signals

Django has it’s own pub sub model of receiving and sending events, called Signals. From the docs:

> Django includes a “signal dispatcher” which helps decoupled applications get
notified when actions occur elsewhere in the framework. In a nutshell, signals
allow certain *senders* to notify a set of *receivers* that some action has
taken place. They’re especially useful when many pieces of code may be
interested in the same events.
> 

A signal is an instance of the django.dispatch.Signal class. 

```python
import django.dispatch

order_status = django.dispatch.Signal()
```

### Senders

`Signal.send` is used to send the event.

```python
class Pizza:
    ...

    def send_pizza(self, toppings, size):
        order_status.send(sender=self.__class__, toppings=toppings, size=size)
        ...
```

> You must provide the`sender` argument (which is a class most of the time)
> 

> Returns a list of tuple pairs `[(receiver, response), ...]`,
representing the list of called receiver functions and their response values.
> 

### Receivers

A receiver can be any Python function or method
`def update_order(sender, **kwargs):`

Note that adding `**kwargs` to your receivers is important, since the sender can add more parameters at any time, the receiver should be able to handle these.

### Connecting Receivers

`Signal.connect` helps us to connect receivers to the signal’s events.

`Signal.connect(receiver, sender=None, weak=True, dispatch_uid=None)` [[source]](https://github.com/django/django/blob/stable/5.1.x/django/dispatch/dispatcher.py#L50)[¶](https://docs.djangoproject.com/en/5.1/topics/signals/#django.dispatch.Signal.connect)

To run the `update_order` on the `order_status_signal` we can connect it by so

```python
order_status_signal.connect(update_order)
```

or just add the `@receiver(order_status_signal)` decorator to the `update_order` function.

You can also control on which senders to receive events from, and block out all the noise using the sender argument.

```python
order_status_signal.connect(update_order, sender=Pizza)
```

> Different signals use different objects as their senders; you’ll need to consult
the [built-in signal documentation](https://docs.djangoproject.com/en/5.1/ref/signals/) for details of each
particular signal.
> 

Some examples of common senders

`django.db.models.signals.pre_save`

- the Model being saved

`django.db.models.signals.pre_migrate` 

- AppConfig instance for the app

`django.core.signals.request_started` 

- The handler class – e.g. `django.core.handlers.wsgi.WsgiHandler` – that
handled the request.

The code connecting the signal to the receiver may run many times, causing the receiver to be called many times for the one signal. This can happen easily if the file containing the signal definition is imported in many places, and on each import it registers the receiver again and again.

The `dispatch_uid` identifier helps to avoid the above issue. It can be any hashable object.

```python
order_status_signal.connect(update_order,dispatch_uid="unique_identifier")
```

The weak argument is interesting to dive into. More on this at the end

> **weak** – Django stores signal handlers as weak references by
default. Thus, if your receiver is a local function, it may be
garbage collected. To prevent this, pass `weak=False` when you call
the signal’s `connect()` method.
> 

# Auditlog

https://github.com/jazzband/django-auditlog

> `django-auditlog` (Auditlog) is a reusable app 
for Django that makes logging object changes a breeze. Auditlog tries to
 use as much as Python and Django's built in functionality to keep the 
list of dependencies as short as possible. Also, Auditlog aims to be 
fast and simple to use.
> 

It tracks object changes when saving and stores the changes in a table. The fields that it stores:

- content_type - `from django.contrib.contenttypes.models.ContentType` which gives you which model we’re looking at
- object_pk
- object_id
- object_repr
- serialized_data
- changes_text
- changes
- actor - The Django User who made the changes
- cid - correlation id
- remote_addr
- remote_port
- timestamp
- additional_data

### Registering models

The magic begins with registering models you want to track with [auditlog.register](https://github.com/jazzband/django-auditlog/blob/512cd2831850d958b1080a165139b11b439195ad/auditlog/registry.py#L62). I’ll keep just the barebones here.

```python
from django.db.models.signals import (
    ModelSignal,
    m2m_changed,
    post_delete,
    post_save,
    pre_save,
)
```

First off, we import all the Django signals to subscribe to.

```python
class AuditlogModelRegistry:
    """
    A registry that keeps track of the models that use Auditlog to track changes.
    """
    def __init__(
        self,
        create: bool = True,
        update: bool = True,
        delete: bool = True,
        access: bool = True,
        m2m: bool = True,
        custom: Optional[dict[ModelSignal, Callable]] = None,
    ):
        from auditlog.receivers import log_access, log_create, log_delete, log_update
        
        # These receivers track and write the changes into the LogEntry model. 

        self._registry = {} # Stores the models along with many of the parameters for the model to be tracked
        self._signals = {} # Stores the receivers for each type of Django signal
        self._m2m_signals = defaultdict(dict) # Many to many relations are handled separately

        if create:
            self._signals[post_save] = log_create
        if update:
            self._signals[pre_save] = log_update
        if delete:
            self._signals[post_delete] = log_delete
        if access:
            self._signals[accessed] = log_access
        self._m2m = m2m

        if custom is not None:
            self._signals.update(custom)
            
```

The `AuditlogModelRegistry` has 3 fields it uses to keep track of things

- `self._registry` → models along with the parameters passed in
- `self._signals` → The Django signals that should be listened to
    - Each of the corresponding signal, has it’s own receiver
    - Eg: `post_save` has `log_create` attached to it.

The `register` function, which we use to register a model with Auditlog, adds the model along with all the options we passed, into its `_registry` . 

```python

    def register(
        self,
        model: ModelBase = None
        ... # all the options we talked about earlier
       ):

        def registrar(cls): # cls here is the Model 
            """Register models for a given class."""
            if not issubclass(cls, Model):
                raise TypeError("Supplied model is not a valid model.")

            self._registry[cls] = {
							... various options
            }
            self._connect_signals(cls)

            # We need to return the class, as the decorator is basically
            # syntactic sugar for:
            # MyClass = auditlog.register(MyClass)
            return cls

        if model is None:
            # If we're being used as a decorator, return a callable with the
            # wrapper.
            return lambda cls: registrar(cls)
        else:
            # Otherwise, just register the model.
            registrar(model)

```

The `_connect_signals` then takes each signal (pre_save, post_save .. ) and it’s corresponding receiver(log_create, log_update .. ) and starts connecting these for the models that were registered. 

```python
    def _connect_signals(self, model):
    		....
				
        for signal, receiver in self._signals.items():
            signal.connect(
                receiver,
                sender=model,
                dispatch_uid=self._dispatch_uid(signal, receiver),
            )
```

Note how the `sender=model` param has been used, so that auditlog receives signals only for the models that were registered.

### The Receivers

```python
@check_disable
def log_create(sender, instance, created, **kwargs):
    """
    Signal receiver that creates a log entry when a model instance is first saved to the database.

    Direct use is discouraged, connect your model through :py:func:`auditlog.registry.register` instead.
    """
    if created:
        _create_log_entry(
            action=LogEntry.Action.CREATE,
            instance=instance,
            sender=sender,
            diff_old=None,
            diff_new=instance,
        )
```

Nothing fancy happening here. Just receive the event, and `_create_log_entry` ends up writing a row in the `LogEntry` table, giving us the final result, and similar flows for update and delete.

Wait, how does the library know about the actor, remote_address, remote_port? There’s no mentions in the above logic fetching that

## Auditlog Middleware and the pre_save Signal for LogEntry itself

Audit logs are much more useful with the actual user who initiated the action also tracked as a part of the changes.

To achieve this with minimal effort, it has a middleware, which adds a receiver onto the `pre_save` signal for the `LogEntry` model itself. This works, because `LogEntry` itself, is a Django model.

Now whenever the LogEntry is written, the middleware adds the necessary data (actor, ip address .. ) onto the instance just before it’s written in the pre_save hook.

```python

class AuditlogMiddleware:
    """
    Middleware to couple the request's user to log items. This is accomplished by currying the
    signal receiver with the user from the request (or None if the user is not authenticated).
    """

		...

    @staticmethod
    def _get_actor(request):
        user = getattr(request, "user", None)
        if isinstance(user, get_user_model()) and user.is_authenticated:
            return user
        return None
    
    ...

    def __call__(self, request):
        remote_addr = self._get_remote_addr(request)
        remote_port = self._get_remote_port(request)
        user = self._get_actor(request)

        set_cid(request)

        with set_actor(actor=user, remote_addr=remote_addr, remote_port=remote_port):
            return self.get_response(request)
 
```

Looking at set_actor, my assumption here is that partials are used to make the code more readable and doesn’t have other implications. Helps to avoid defining a nested function inside `set_actor` just to pass the user and signal_duid

```python
def set_actor(actor, remote_addr=None, remote_port=None):
		...
		
    # Connect signal for automatic logging
    set_actor = partial(_set_actor, user=actor, signal_duid=context_data["signal_duid"])
    pre_save.connect(
        set_actor,
        sender=LogEntry,
        dispatch_uid=context_data["signal_duid"],
        weak=False,
    )

def _set_actor(user, sender, instance, signal_duid, **kwargs):
    """Signal receiver with extra 'user' and 'signal_duid' kwargs.

    This function becomes a valid signal receiver when it is curried with the actor and a dispatch id.
    """
    
		...
		
    else:
        if signal_duid != auditlog["signal_duid"]:
            return
        auth_user_model = get_user_model()
        if (
            sender == LogEntry
            and isinstance(user, auth_user_model)
            and instance.actor is None
        ):
            instance.actor = user

        instance.remote_addr = auditlog["remote_addr"]
        instance.remote_port = auditlog["remote_port"]
```

# Weakref and weak=true

### [The weakref package](https://docs.python.org/3/library/weakref.html)

- [Garbage Collection](https://docs.python.org/3/glossary.html#term-garbage-collection):
    
    > The process of freeing memory when it is not used anymore.  Python
    performs garbage collection via reference counting and a cyclic garbage
    collector that is able to detect and break reference cycles.  The
    garbage collector can be controlled using the [`gc`](https://docs.python.org/3/library/gc.html#module-gc) module.
    > 

The distinction comes in where weak references are not counted by the garbage collector. This allows for creating caches and mappings where the objects don’t remain alive just because it’s referenced in the mapping objects.

`weakref.ref` → allows to create a weak reference, that can be called to return the object if it’s still alive, else just returns None. 

`finalize` → cleanup function to be called when an object is garbage collected.

Looking at how [django.Signals.connect](https://github.com/django/django/blob/main/django/dispatch/dispatcher.py#L103C1-L112C1) handles the weak argument:

```python
import weakref
...
    def connect(self, receiver, sender=None, weak=True, dispatch_uid=None):
		    ...
        if weak:
            ref = weakref.ref
            receiver_object = receiver
            
            # Check for bound methods
            if hasattr(receiver, "__self__") and hasattr(receiver, "__func__"):
                ref = weakref.WeakMethod
                receiver_object = receiver.__self__
            receiver = ref(receiver)
            weakref.finalize(receiver_object, self._remove_receiver)

				.....
				self.receivers.append((lookup_key, receiver, is_async))
```

If weak=True was passed, the receiver is replaced with a weak reference to the receiver so that it doesn’t prevent it from getting garbage collected.

Getting sidetracked, Why the extra check for bound methods (a method defined on a class and looked up on an instance) ?

> Since a bound method is ephemeral, a standard weak reference cannot keep
hold of it.  [`WeakMethod`](https://docs.python.org/3/library/weakref.html#weakref.WeakMethod) has special code to recreate the bound
method until either the object or the original function dies
> 

What does it do differently?

WeakMethod stores the weak references to both the bound method’s function and the object it is bound to. So it first, fetches the reference of the object, then re-creates the bound method

```jsx
class WeakMethod(ref):        
    def __new__(cls, meth, callback=None):
        try:
            obj = meth.__self__
            func = meth.__func__
        except AttributeError:
            raise TypeError("argument should be a bound method, not {}"
                            .format(type(meth))) from None
        ...
        self = ref.__new__(cls, obj, _cb)
        self._func_ref = ref(func, _cb)
        self._meth_type = type(meth)
				...
        return self

    def __call__(self):
        obj = super().__call__()
        func = self._func_ref()
        if obj is None or func is None:
            return None
        return self._meth_type(func, obj)
```

To understand this better, Let’s just try this logic out on the shell

```python
import weakref
import types

class MyClass:
    def my_method(self):
        print("Hello from MyClass!")

obj = MyClass()

# Get the function part of the bound method
wm_func = obj.my_method.__func__

# Get the instance (object) the method is bound to
wm_obj = obj.my_method.__self__

# Save the type to re-create bound method
saved_type = type(obj.my_method)

# Now create weak references
wm_func_ref = weakref.ref(wm_func)
wm_obj_ref = weakref.ref(wm_obj)

# Recreate the bound method
recreated_method = saved_type(wm_func_ref(), wm_obj_ref())
recreated_method()
# "Hello from MyClass!" ✅
```

Finally, now when to use weak=False?

A non-exhaustive list:

- When the receiver is defined as a nested function
    
    ```python
    >>> from django.db.models.signals import post_save
    >>> from django.dispatch import Signal
    >>>
    >>> # Custom signal for the example
    >>> my_signal = Signal()
    >>>
    >>> # Outer function with a local signal receiver
    >>> def connect_local_signal(weak=True):
    ...     def local_receiver(sender, **kwargs):
    ...         print("Local receiver received signal!")
    ...     my_signal.connect(local_receiver, weak)
    ...
    >>> # Connect and fire the signal
    >>> connect_local_signal()
    >>> my_signal.send(sender=None)
    []
    
    >>> connect_local_signal(weak=False)
    >>> my_signal.send(sender=None)
    
    Local receiver received signal!
    [(<function connect_local_signal.<locals>.local_receiver at 0x100df89a0>, None)]
    ```
    
- When the handler is a class method tied to an object
    
    ```python
    # Define a class with a method that will receive the signal
    class MyHandler:
        def __init__(self, name):
            self.name = name
        def my_method(self, sender, **kwargs):
            print(f"Signal received by {self.name}")
    
    # Create an instance of the handler and connect its method to the signal
    handler = MyHandler("Handler1")
    my_signal.connect(handler.my_method, weak=False)
    
    # Fire the signal
    my_signal.send(sender=None)
    # Signal received by Handler1
    
    # Now, delete the handler
    del handler
    
    # The method is still connected and will receive the signal
    my_signal.send(sender=None)
    # >>> Signal received by Handler1
    ```
    
- Connecting lambda functions to signals
    
    ```python
    my_signal.connect(lambda sender, **kwargs: print("Signal received by lambda"), weak=False)
    ```
    

and so on.

It is worth noting that the Auditlog middleware uses weak=False to avoid the set_actor receiver getting garbage collected.
