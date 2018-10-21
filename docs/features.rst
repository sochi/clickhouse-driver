.. _features:

Features
========

- Compression support:

  * `LZ4/LZ4HC <http://www.lz4.org/>`_
  * `ZSTD <https://facebook.github.io/zstd/>`_

- TLS support (since server version 1.1.54304).


.. _external-tables:

External data for query processing
----------------------------------

You can pass `external data <https://clickhouse.yandex/docs/en/single/index.html#external-data-for-query-processing>`_
alongside with query:

    .. code-block:: python

        tables = [{
            'name': 'ext',
            'structure': [('x', 'Int32'), ('y', 'Array(Int32)')],
            'data': [
                {'x': 100, 'y': [2, 4, 6, 8]},
                {'x': 500, 'y': [1, 3, 5, 7]},
            ]
        }]
        rv = client.execute('SELECT sum(x) FROM ext', external_tables=tables)
        print(rv)


Settings
--------

There are a lot of ClickHouse server `settings <https://clickhouse.yandex/docs/en/single/index.html#server-settings>`_.
Settings can be specified during Client's initialization:

    .. code-block:: python

        # Set max number threads for all queries execution.
        settings = {'max_threads': 2}
        client = Client('localhost', settings=settings)

Each setting can be overridden in any `execute` statement:

    .. code-block:: python

        # Set lower priority to query and limit max number threads to execute the request.
        settings = {'max_threads': 2, 'priority': 10}
        print(client.execute('SHOW TABLES', settings=settings))


Compression
-----------

Native protocol supports two types of compression: `LZ4 <http://www.lz4.org/>`_ and
`ZSTD <https://facebook.github.io/zstd/>`_. When compression is enabled compressed data
should be hashed using `CityHash algorithm <https://github.com/google/cityhash>`_.
Additional packages should be install in order by enable compression suport, see :ref:`installation-pypi`.
Enabled client-side compression can save network traffic.

Clients with compression support can be constructed in following way:

    .. code-block:: python

        >>> from clickhouse_driver import Client
        >>> client_with_lz4 = Client('localhost', compression=True)
        >>> client_with_lz4 = Client('localhost', compression='lz4')
        >>> client_with_zstd = Client('localhost', compression='zstd')


.. _compression-cityhash-notes:

CityHash algorithm notes
~~~~~~~~~~~~~~~~~~~~~~~~

Unfortunately ClickHouse server comes with built-in old version of CityHash
hashing algorithm (1.0.2). That's why we can't use original
`CityHash <https://pypi.org/project/cityhash>`_ package. Downgraded version of
this algorithm is placed at `PyPI <https://pypi.org/project/clickhouse-cityhash>`_.


Secure connection
-----------------

    .. code-block:: python

        from clickhouse_driver import Client

        client = Client('localhost', secure=True)
        # Using self-signed certificate.
        self_signed_client = Client('localhost', secure=True, ca_certs='/etc/clickhouse-server/server.crt')
        # Disable verification.
        no_verifyed_client = Client('localhost', secure=True, verify=False)

        # Example of secured client with Let's Encrypt certificate.
        import certifi

        client = Client('remote-host', secure=True, ca_certs=certifi.where())


Specifying query id
-------------------

You can manually set query identificator for each query. UUID for example:

    .. code-block:: python

        from uuid import uuid1

        query_id = str(uuid1())
        print(client.execute('SHOW TABLES', query_id=query_id))


Retrieving results in columnar form
-----------------------------------

Columnar form sometimes can be more useful.

    .. code-block:: python

        print(client.execute('SELECT arrayJoin(range(3))', columnar=True))


Data types checking on INSERT
-----------------------------

Data types check is disabled for performance on ``INSERT`` queries.
You can turn it on by `types_check` option:

    .. code-block:: python

        client.execute('INSERT INTO test (x) VALUES', [('abc', )], types_check=True)


Reading query profile info
--------------------------

Last query's profile info can be examined. `rows_before_limit` examine example:

    .. code-block:: python

        rows = client.execute('SELECT * FROM test ORDER BY foo LIMIT 5')
        total_rows_count = client.last_query.profile_info.rows_before_limit


Receiving logs
--------------

Query logs can be received from server by using `send_logs_level` setting:

    .. code-block:: python

        settings = {'send_logs_level': 'debug'}
        self.client.execute('SELECT 1', settings=settings)
