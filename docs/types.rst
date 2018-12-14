
Supported types
===============

Each ClickHouse type is deserialized to a corresponding Python type when SELECT queries are prepared.
When serializing INSERT queries, clickhouse-driver accepts a broader range of Python types.
The following ClickHouse types are supported by clickhouse-driver:


[U]Int8/16/32/64
----------------

INSERT types: :class:`int`, :class:`long`.

SELECT type: :class:`int`.


Float32/64
----------

INSERT types: :class:`float`, :class:`int`, :class:`long`.

SELECT type: :class:`float`.


Date
----

INSERT types: :class:`~datetime.date`.

SELECT type: :class:`~datetime.date`.


DateTime('timezone')
--------------------

*Timezone support is new in version 0.0.11.*

INSERT types: :class:`~datetime.datetime`, :class:`int`, :class:`long`.

Integers are interpreted as seconds without timezone (UNIX timestamps). Integers can be used when
insertion of datetime column is a bottleneck.

SELECT type: :class:`~datetime.datetime`.

Setting `use_client_time_zone <https://clickhouse.yandex/docs/en/single/#datetime>`_ is taken into consideration.


String/FixedString(N)
---------------------

INSERT types: :class:`str`/:func:`basestring <basestring>`, :class:`bytearray`, :class:`bytes`. See note below.

SELECT type: :class:`str`/:func:`basestring <basestring>`, :class:`bytes`. See note below.

String column is encoded/decoded using UTF-8 encoding.

String column can be returned without decoding. Return values are `bytes`:

    .. code-block:: python

        >>> settings = {'strings_as_bytes': True}
        >>> rows = client.execute(
        ...     'SELECT * FROM table_with_strings',
        ...     settings=settings
        ... )


If a column has FixedString type, upon returning from SELECT it may contain trailing zeroes
in accordance with ClickHouse's storage format. Trailing zeroes are stripped by driver for convenience.

During SELECT, if a string cannot be decoded with UTF-8 encoding, it will return as :class:`bytes`.

During INSERT, if ``strings_as_bytes`` setting is not specified and string cannot be encoded with ``UTF-8``,
a ``UnicodeEncodeError`` will be raised.


Enum8/16
--------

INSERT types: :class:`~enum.Enum`, :class:`int`, :class:`long`, :class:`str`/:func:`basestring <basestring>`.

SELECT type: :class:`str`/:func:`basestring <basestring>`.

    .. code-block:: python

        >>> from enum import IntEnum
        >>>
        >>> class MyEnum(IntEnum):
        ...     foo = 1
        ...     bar = 2
        ...
        >>> client.execute('DROP TABLE IF EXISTS test')
        []
        >>> client.execute('''
        ...     CREATE TABLE test
        ...     (
        ...         x Enum8('foo' = 1, 'bar' = 2)
        ...     ) ENGINE = Memory
        ... ''')
        []
        >>> client.execute(
        ...     'INSERT INTO test (x) VALUES',
        ...     [{'x': MyEnum.foo}, {'x': 'bar'}, {'x': 1}]
        ... )
        >>> client.execute('SELECT * FROM test')
        [(u'foo',), (u'bar',), (u'foo',)]

For Python 2.7, `enum34 <https://pypi.org/project/enum34>`_ package is used.

Currently clickhouse-driver can't handle empty enum value due to Python's `Enum` mechanics.
Enum member name must be not empty. See `issue`_ and  `workaround`_.

.. _issue: https://github.com/mymarilyn/clickhouse-driver/issues/48
.. _workaround: https://github.com/mymarilyn/clickhouse-driver/issues/48#issuecomment-412480613


Array(T)
--------

INSERT types: :class:`list`, :class:`tuple`.

SELECT type: :func:`tuple <tuple>`.


    .. code-block:: python

        >>> client.execute('DROP TABLE IF EXISTS test')
        []
        >>> client.execute(
        ...     'CREATE TABLE test (x Array(Int32)) '
        ...     'ENGINE = Memory'
        ... )
        []
        >>> client.execute(
        ...     'INSERT INTO test (x) VALUES',
        ...     [{'x': [10, 20, 30]}, {'x': [11, 21, 31]}]
        ... )
        >>> client.execute('SELECT * FROM test')
        [((10, 20, 30),), ((11, 21, 31),)]


Nullable(T)
-----------

INSERT types: :data:`~types.NoneType`, ``T``.

SELECT type: :data:`~types.NoneType`, ``T``.


UUID
----

INSERT types: :class:`str`/:func:`basestring <basestring>`, :class:`~uuid.UUID`.

SELECT type: :class:`~uuid.UUID`.


Decimal
-------

*New in version 0.0.16.*

INSERT types: :class:`~decimal.Decimal`, :class:`float`, :class:`int`, :class:`long`.

SELECT type: :class:`~decimal.Decimal`.
