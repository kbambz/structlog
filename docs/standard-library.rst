Standard Library Logging
========================

Ideally, ``structlog`` should be able to be used as a drop-in replacement for standard library's :mod:`logging` by wrapping it.
In other words, you should be able to replace your call to :func:`logging.getLogger` by a call to :func:`structlog.get_logger` and things should keep working as before (if ``structlog`` is configured right, see :ref:`stdlib-config` below).

If you run into incompatibilities, it is a *bug* so please take the time to `report it <https://github.com/hynek/structlog/issues>`_!
If you're a heavy :mod:`logging` user, your `help <https://github.com/hynek/structlog/issues?q=is%3Aopen+is%3Aissue+label%3Astdlib>`_ to ensure a better compatibility would be highly appreciated!


Concrete Bound Logger
---------------------

To make ``structlog``'s behavior less magicy, it ships with a standard library-specific wrapper class that has an explicit API instead of improvising: :class:`structlog.stdlib.BoundLogger`.
It behaves exactly like the generic :class:`structlog.BoundLogger` except:

- it's slightly faster due to less overhead,
- has an explicit API that mirrors the log methods of standard library's :class:`logging.Logger`,
- hence causing less cryptic error messages if you get method names wrong.


Processors
----------

``structlog`` comes with a few standard library-specific processors:

:func:`~structlog.stdlib.render_to_log_kwargs`:
   Renders the event dictionary into keyword arguments for :func:`logging.log` that attaches everything except the `event` field to the *extra* argument.
   This is useful if you want to render your log entries entirely within :mod:`logging`.

:func:`~structlog.stdlib.filter_by_level`:
   Checks the log entry's log level against the configuration of standard library's logging.
   Log entries below the threshold get silently dropped.
   Put it at the beginning of your processing chain to avoid expensive operations happen in the first place.

:func:`~structlog.stdlib.add_logger_name`:
   Adds the name of the logger to the event dictionary under the key ``logger``.

:func:`~structlog.stdlib.add_log_level`:
   Adds the log level to the event dictionary under the key ``level``.

:class:`~structlog.stdlib.PositionalArgumentsFormatter`:
   This processes and formats positional arguments (if any) passed to log methods in the same way the ``logging`` module would do, e.g. ``logger.info("Hello, %s", name)``.


``structlog`` also comes with :class:`~structlog.stdlib.ProcessorFormatter` which is a :class:`logging.Formatter` that enables you to format non-``structlog`` log entries using ``structlog`` renderers *and* multiplex ``structlog``’s output with different renderers (see below for an example).


.. _stdlib-config:

Suggested Configurations
------------------------

Depending *where* you'd like to do your formatting, you can take one of three approaches:


Rendering Using :mod:`logging`-based Formatters
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    import structlog

    structlog.configure(
        processors=[
            structlog.stdlib.filter_by_level,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            structlog.stdlib.PositionalArgumentsFormatter(),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.UnicodeDecoder(),
            structlog.stdlib.render_to_log_kwargs,
        ],
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

Now you have the event dict available within each log record.
If you want all your log entries (i.e. also those not from your app/``structlog``) to be formatted as JSON, you can use the `python-json-logger library <https://github.com/madzak/python-json-logger>`_:

.. code-block:: python

    import logging
    import sys

    from pythonjsonlogger import jsonlogger

    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(jsonlogger.JsonFormatter())
    root_logger = logging.getLogger()
    root_logger.addHandler(handler)

Now both ``structlog`` and ``logging`` will emit JSON logs:

.. code-block:: pycon

    >>> structlog.get_logger("test").warning("hello")
    {"message": "hello", "logger": "test", "level": "warning"}

    >>> logging.getLogger("test").warning("hello")
    {"message": "hello"}


Rendering Using ``structlog``-based Formatters Within :mod:`logging`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can leave rendering for later and yet use ``structlog``’s renderers for it:

.. code-block:: python

    import logging.config
    import structlog

    timestamper = structlog.processors.TimeStamper(fmt="%Y-%m-%d %H:%M:%S")
    pre_chain = [
        # Add the log level and a timestamp to the event_dict if the log entry
        # is not from structlog.
        structlog.stdlib.add_log_level,
        timestamper,
    ]

    logging.config.dictConfig({
            "version": 1,
            "disable_existing_loggers": False,
            "formatters": {
                "plain": {
                    "()": structlog.stdlib.ProcessorFormatter,
                    "processor": structlog.dev.ConsoleRenderer(colors=False),
                    "foreign_pre_chain": pre_chain,
                },
                "colored": {
                    "()": structlog.stdlib.ProcessorFormatter,
                    "processor": structlog.dev.ConsoleRenderer(colors=True),
                    "foreign_pre_chain": pre_chain,
                },
            },
            "handlers": {
                "default": {
                    "level": "DEBUG",
                    "class": "logging.StreamHandler",
                    "formatter": "colored",
                },
                "file": {
                    "level": "DEBUG",
                    "class": "logging.handlers.WatchedFileHandler",
                    "filename": "test.log",
                    "formatter": "plain",
                },
            },
            "loggers": {
                "": {
                    "handlers": ["default", "file"],
                    "level": "DEBUG",
                    "propagate": True,
                },
            }
    })
    structlog.configure(
        processors=[
            structlog.stdlib.add_log_level,
            structlog.stdlib.PositionalArgumentsFormatter(),
            timestamper,
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.stdlib.ProcessorFormatter.wrap_for_formatter,
        ],
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

This defines two formatters: one plain and one colored.
Both are run for each log entry.
Log entries that do not originate from ``structlog``, are additionally pre-processed using a cached ``timestamper`` and :func:`~structlog.stdlib.add_log_level`.

.. code-block:: pycon

    >>> logging.getLogger().warning("bar")
    2017-03-06 11:49:27 [warning  ] bar

    >>> structlog.get_logger("structlog").warning("foo", x=42)
    2017-03-06 11:49:32 [warning  ] foo                            x=42

    >>> print(open("test.log").read())
    2017-03-06 11:49:27 [warning  ] bar
    2017-03-06 11:49:32 [warning  ] foo                            x=42

(sadly, you have to imagine the colors in the first two outputs)

If you leave ``foreign_pre_chain`` `None`, formatting will be left to :mod:`logging`.
Meaning: you can define a ``format`` for :class:`~structlog.stdlib.ProcessorFormatter` too!


Rendering Within ``structlog``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A basic configuration to output structured logs in JSON format looks like this:

.. code-block:: python

    import structlog

    structlog.configure(
        processors=[
            structlog.stdlib.filter_by_level,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            structlog.stdlib.PositionalArgumentsFormatter(),
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.UnicodeDecoder(),
            structlog.processors.JSONRenderer()
        ],
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

(if you're still runnning Python 2, replace :class:`~structlog.processors.UnicodeDecoder` through :class:`~structlog.processors.UnicodeEncoder`)

To make your program behave like a proper `12 factor app`_ that outputs only JSON to ``stdout``, configure the ``logging`` module like this::

  import logging
  import sys

  logging.basicConfig(
      format="%(message)s",
      stream=sys.stdout,
      level=logging.INFO,
  )

In this case *only* your own logs are formatted as JSON:

.. code-block:: pycon

    >>> structlog.get_logger("test").warning("hello")
    {"event": "hello", "logger": "test", "level": "warning", "timestamp": "2017-03-06T07:39:09.518720Z"}

    >>> logging.getLogger("test").warning("hello")
    hello

.. _`12 factor app`: https://12factor.net/logs
