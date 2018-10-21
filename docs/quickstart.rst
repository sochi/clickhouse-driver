.. _quickstart:

Quickstart
==========

This page gives a good introduction to clickhouse-driver.
It assumes you already have clickhouse-driver installed.
If you do not, head over to the :ref:`installation` section.

A minimal working example looks like this:

    .. code-block:: python

        from clickhouse_driver import Client

        client = Client('localhost')

        print(client.execute('SHOW TABLES'))


This code will show all tables from ``'default'`` database.

There are two conceptual types of queries:

- Queries that do not transfer data: SELECT, SHOW, etc.
- Queries that transfer data: INSERT.


Every query should be executed by calling one of client's execute
methods: `execute`, `execute_with_progress`, `execute_iter method`.


Selecting data
--------------

Simple select query looks like:

    .. code-block:: python

        >>> client.execute('SELECT * FROM system.numbers LIMIT 5')
        [(0,), (1,), (2,), (3,), (4,)]


Of course queries can and should be parameterized to avoid SQL injections:

    .. code-block:: python

        >>> from datetime import date
        >>> client.execute(
        ...     'SELECT %(date)s, %(a)s + %(b)s',
        ...     {'date': date.today(), 'a': 1, 'b': 2}
        ... )
        [('2018-10-21', 3)]

.. _execute-with-progress:

Selecting data with progress information
----------------------------------------

You can get query progress information by using `execute_with_progress`. It can be useful for canceling long queries.

    .. code-block:: python

        from datetime import datetime

        progress = client.execute_with_progress('LONG AND COMPLICATED QUERY')

        timeout = 20
        started_at = datetime.now()

        for num_rows, total_rows in progress:
            done = float(num_rows) / total_rows if total_rows else total_rows
            now = datetime.now()
            # Cancel query if it takes more than 20 seconds to process 50% of rows.
            if (now - started_at).total_seconds() > timeout and done < 0.5:
                client.cancel()
                break
        else:
            rv = progress.get_result()
            print(rv)


.. _execute-iter:

Streaming results
-----------------

When you are dealing with large datasets block by block results streaming may be useful:

    .. code-block:: python

        settings = {'max_block_size': 100000}
        rows_gen = client.execute_iter('QUERY WITH MANY ROWS', settings=settings)

        for row in rows_gen:
            print(row)


Inserting data
--------------

Insert queries in `Native protocol <https://clickhouse.yandex/docs/en/single/index.html#native-interface-tcp>`_
are a little bit tricky because of ClickHouse columnar nature. And because we're using Python.

INSERT query is consist of two parts: query and query data. Query data is split into chunks that called blocks.
Each block is sent in binary columnar form.

As data in each block in send in binary form we should not serialize into string by
using substitution ``%(a)s`` and then deserialize it back into Python types.

This INSERT  will be completely inefficient when thousands rows are passed:

    .. code-block:: python

        client.execute(
            'INSERT INTO test (x) VALUES (%(a)s), (%(b)s), ...',
            {'a': 1, 'b': 2, ...}
        )


For inserting data must you specify insert query that ends with `VALUES` clause. Data must be specified separately.

    .. code-block:: python

        >>> client.execute(
        ...     'INSERT INTO test (x) VALUES',
        ...     [{'x': 1}, {'x': 2}, {'x': 3}, {'x': 100}]
        ... )
        >>> client.execute(
        ...     'INSERT INTO test (x) VALUES',
        ...     [[200]]
        ... )
        >>> client.execute(
        ...     'INSERT INTO test (x) VALUES',
        ...     ((x, ) for x in range(5))
        ... )

Data for insert can be list/tuple/generator of lists/tuples/dicts.

If data is not passed client will hang with further timeout.

    .. code-block:: python

        client.execute('INSERT INTO test (x) VALUES')  # will hang

Following example WILL NOT work:

    .. code-block:: python

        >>> client.execute(
        ...     'INSERT INTO test (x) VALUES (%(a)s), (%(b)s)',
        ...     {'a': 1, 'b': 2}
        ... )


Of course `INSERT ... SELECT` query will work without any data:

    .. code-block:: python

        >>> client.execute(
        ...     'INSERT INTO test (x) '
        ...     'SELECT * FROM system.numbers LIMIT %(limit)s',
        ...     {'limit': 5}
        ... )
        []

For ClickHouse server this query like usual `SELECT` query.

DDL
---

DDL queries can be executed in the same way SELECT queries are executed

    .. code-block:: python

        client.execute('DROP TABLE IF EXISTS test')
        client.execute('CREATE TABLE test (x Int32) ENGINE = Memory')


