Context managers
----------------

Context managers allow you to allocate and release resources precisely
when you want to. The most widely used example of context managers is
the ``with`` statement. Suppose you have two related operations which
you’d like to execute as a pair, with a block of code in between.
Context managers allow you to do specifically that. For example:

::

    with open('some_file', 'wb') as opened_file:
        opened_file.write('Hola!')

The above code opens the file, writes some data to it and then closes
it. If an error occurs while writing the data to the file, it tries to
close it. The above code is equivalent to:

::

    file = open('some_file', 'wb')
    try:
        file.write('Hola!')
    finally:
        file.close()

While comparing it to the first example we can see that a lot of
boilerplate code is eliminated just by using ``with``. The main
advantage of using a ``with`` statement is that it makes sure our file
is closed without paying attention to how the nested block exits.

A common use case of context managers is locking and unlocking resources
and closing opened files (as I have already showed you).

Let's see how we can implement our own Context Manager. This would allow
us to understand exactly what's going on behind the scenes.

Implementing Context Manager as a Class:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

At the very least a context manager has an ``__enter__`` and
``__exit__`` methods defined. Let's make our own file opening Context
Manager and learn the basics.

::

    class File(object):
        def __init__(self, file_name, method):
            self.file_obj = open(file_name, method)
        def __enter__(self):
            return self.file_obj
        def __exit__(self, type, value, trace_back):
            self.file_obj.close()

Just by defining ``__enter__`` and ``__exit__`` methods we can use it in
a ``with`` statement. Let's try:

::

    with File('demo.txt', 'wb') as opened_file:
        opened_file.write('Hola!')

Our ``__exit__`` function accepts three arguments. They are required by
every ``__exit__`` method which is a part of a Context Manager class.
Let's talk about what happens under-the-hood.

1. The ``with`` statement stores the ``__exit__`` method of ``File``
   class.
2. It calls the ``__enter__`` method of ``File`` class.
3. ``__enter__`` method opens the file and returns it.
4. the opened file handle is passed to ``opened_file``.
5. we write to the file using ``.write()``
6. ``with`` statement calls the stored ``__exit__`` method.
7. the ``__exit__`` method closes the file.

Handling exceptions
^^^^^^^^^^^^^^^^^^^

We did not talk about the ``type``, ``value`` and ``trace_back``
arguments of the ``__exit__`` method. Between the 4th and 6th step, if
an exception occurs, Python passes the type, value and traceback of the
exception to the ``__exit__`` method. It allows the ``__exit__`` method
to decide how to close the file and if any further steps are required.
In our case we are not paying any attention to them.

What if our file object raises an exception? We might be trying to
access a method on the file object which it does not supports. For
instance:

::

    with File('demo.txt', 'wb') as opened_file:
        opened_file.fuck('Hola!')

Let's list down the steps which are taken by the ``with`` statement when
an error is encountered.

1. It passes the type, value and traceback of the error to the
   ``__exit__`` method.
2. It allows the ``__exit__`` method to handle the exception.
3. If ``__exit__`` returns True then the exception was gracefully
   handled.
4. If anything else than True is returned by ``__exit__`` method then
   the exception is raised by ``with`` statement.

In our case the ``__exit__`` method returns ``None`` (when no return
statement is encountered then the method returns ``None``). Therefore,
``with`` statement raises the exception.

::

    Traceback (most recent call last):
      File "<stdin>", line 2, in <module>
    AttributeError: 'file' object has no attribute 'fuck'

Let's try handling the exception in the ``__exit__`` method:

::

    class File(object):
        def __init__(self, file_name, method):
            self.file_obj = open(file_name, method)
        def __enter__(self):
            return self.file_obj
        def __exit__(self, type, value, trace_back):
            print "Exception has been handled"
            self.file_obj.close()
            return True
            
    with File('demo.txt', 'wb') as opened_file:
        opened_file.fuck()
        
    # Output: Exception has been handled

Our ``__exit__`` method returned True, therefore no exception was raised
by the ``with`` statement.

This is not the only way to implement context managers. There is another
way and we will be looking at it in this next section.

Implementing a Context Manager as a Generator
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We can also implement Context Managers using decorators and generators.
Python has a contextlib module for this very purpose. Instead of a
class, we can implement a Context Manager using a generator function.
Let's see a basic, useless example:

::

    from contextlib import contextmanager

    @contextmanager
    def open_file(name):
        f = open(name, 'wb')
        yield f
        f.close()

Okay! This way of implementing Context Managers appear to be more
intuitive and easy. However, this method requires some knowledge about
generators, yield and decorators. In this example we have not caught any
exceptions which might occur. It works in mostly the same way as the
previous method.

Let's disect this method a little.

1. Python encounters the ``yield`` keyword. Due to this it creates a
   generator instead of a normal function.
2. Due to the decoration, contextmanager is called with the function
   name (open\_file) as it's argument.
3. The ``contextmanager`` function returns the generator wrapped by the
   ``GeneratorContextManager`` object.
4. The ``GeneratorContextManager`` is assigned to the ``open_file``
   function. Therefore, when we later call ``open_file`` function, we
   are actually calling the ``GeneratorContextManager`` object.

So now that we know all this, we can use the newly generated Context
Manager like this:

::

    with open_file('some_file') as f:
        f.write('hola!')

What does the Python interpretter do when it reaches the with statement?

TODO: http://preshing.com/20110920/the-python-with-statement-by-example/
