.. highlight:: haskell

More Concepts
=============

In this section we'll look at some further concepts using the Rel8 library that
you might need.

``newtype`` wrappers
--------------------

Haskell is a fantastic language for type safety in part to the ease of
introducing new types. In relational databases we often end up with lots of
different ``id`` columns that represent distinctly different domains, but are
all typed as ``int8`` (for example). Rel8 makes it easy to distinguish these
identifiers by using ``newtype`` just like ordinary haskell code.

Simply enable the ``GeneralizedNewtypeDeriving`` extension, specify your
``newtype`` over a base type, and derive the ``DBType`` and ``FromField``
classes::

  newtype ShopId = ShopId Int64
    deriving (Eq, Ord, Read, Show, FromField)

You can now use this ``newtype`` when defining base tables::

  data Shop f = Shop
    { shopId :: Col f "id" 'HasDefault ShopId
    , ..
    }

If you need to cast ``ShopId`` columns into ``Int64`` columns in a query, you
can do so with the (type-safe) ``coerceExpr`` function::

  coerceExpr :: Coercible a b => Expr a -> Expr b

In our case::

  coerceExpr :: Expr ShopId -> Expr Int64

because all ``ShopId`` s can be coerced into ``Int64``.


Custom Types
------------

In Haskell we often define algebraic data types for types that only have a few
values. For example, we might capture the different types of coffee by::

  data Coffee = PourOver | Espresso | FlatWhite
    deriving (Read, Show)

noting that latte is most certainly not a coffee.

Relational databases don't do a great job of capturing sum types, unfortunately,
but we can still marshal this data through Rel8. We'll need to provide instances
of ``DBType`` and ``FromField``, but this time we'll have to provide an
implementation for ``FromField``::

  instance DBType Coffee
  instance FromField Coffee where
    fromField f a =
      fmap readMay (fromField f a) >>=
      maybe (fail "Could not parse Coffee value") return

The default instance for ``DBType`` will use the ``show`` instance for
``Coffee``, so our ``FromField`` instance uses the ``Read`` instance.


Even more type inference
------------------------

In :doc:`tutorial` we used instances like ``instance Table (Part Expr) (Part
QueryResult)``. It's actually possible to get *even more* type inference using
the following variation::

  instance (expr ~ Expr, queryResult ~ QueryResult) =>
           Table (Part expr) (Part queryResult)

Any existing queries will continue to work, but with deeply nested queries that
can lead to even better inference. The same trick can be used with
``AggregateTable``. If you are using Rel8 seriously, I recommend this approach.


Predicates with ``NULL``
------------------------

If you've browsed the API for Rel8, you might have noticed that the types of
functions like ``filterQuery`` are more general than one might first expect::

  filterQuery :: Predicate bool => (a -> Expr bool) -> O.Query a -> O.Query a

There are two instances for the ``Predicate`` type class::

  instance Predicate Bool
  instance Predicate (Maybe Bool)

This may seem surprising, but in SQL, ``null`` takes on a special meaning in
predicates - it acts like ``false``. Ideally, in Haskell we could do away with
this nonsense and explicitly map ``null`` to false (e.g., ``nullable (lit False)
id myExpr``) but doing so results in SQL that won't hit database indexes (at
least at the time of writing). Let's see this in action with some SQL::

  CREATE TABLE numbers AS
  SELECT x FROM generate_series(1,1000000) x;

  CREATE INDEX numbers_x_idx ON numbers (x);

  EXPLAIN SELECT * FROM numbers WHERE x = 500;
                                    QUERY PLAN
  ----------------------------------------------------------------------------------
  Index Only Scan using numbers_x_idx on numbers  (cost=0.42..8.44 rows=1 width=4)
    Index Cond: (x = 500)
  (2 rows)


  EXPLAIN SELECT * FROM numbers WHERE x IS NOT DISTINCT FROM 500;
                        QUERY PLAN
  -----------------------------------------------------------
  Seq Scan on numbers  (cost=0.00..16925.00 rows=1 width=4)
    Filter: (NOT (x IS DISTINCT FROM 500))

Darn.

Indexes are extremely important when it comes to writing performant queries, so
I generally suggest that if you need to perform comparisons against ``NULL`` in
predicates you retain ``NULL`` and use the "``null`` lifted" operators (``==?``,
``&&?``) along with ``toNullable``. While unfortunate, these queries will often
compile down to considerably more performant queries.

.. _antijoins:

Antijoins
---------

`Wikipedia describes antijoins <https://en.wikipedia.org/wiki/Relational_algebra#Antijoin_.28.E2.96.B7.29>`_ as

  The **antijoin**, written as R ▷ S where R and S are relations, is similar to
  the semijoin, but the result of an antijoin is only those tuples in R for
  which there is no tuple in S that is equal on their common attribute names.

To be a little less technical, an antijoin is a way to restricting the rows in
one query by the absence of rows in another query. For example, we might want to
query all rows in the ``user`` table where those users haven't placed an order.

You can achieve this with a left-join, but PostgreSQL has specific support for
antijoins via the ``NOT EXISTS`` operator, and using this can lead to increased
performance. Rel8 wraps ``EXISTS`` and ``NOT EXISTS`` with the ``exists`` and
``notExists`` arrow commands. Considering on from the above example, we can
write::

  usersNotOrdered :: Query (User Expr)
  usersNotOrdered = proc _ -> do
    user <- queryTable -< () -- Select all users
    (| notExists (do order <- queryTable -< () -- all orders
                     where_ -< userId user ==. orderUserId order) |)


``RecordWildCards`` and ``NamedFieldPuns``
------------------------------------------

The ``RecordWildCards`` and ``NamedFieldPuns`` extensions can lead to some very
readable queries, that are a lot like taking projections with ``SELECT``. For
example, we could join the ``Supplier`` and ``Part`` tables (from
:doc:`tutorial`) on city::

  proc _ -> do
    supplier <- queryTable -< ()
    part <- queryTable -< ()
    where_ -< supplierCity supplier ==. partCity part
    ...

However the ``where_`` clause is a little hard to quickly scan. With
``NamedFieldPuns``, we can write::

  proc _ -> do
    Supplier{ supplierCity } <- queryTable -< ()
    Part{ partCity } <- queryTable -< ()
    where_ -< supplierCity ==. partCity
    ...

This has the advantage of:

1. Making it clear which tables we are selecting from.
2. Making it clear which columns we require from these tables.
3. Allowing us to write a more concise predicate that is easily to understand
   when we are skim reading code.

If you use ``RecordWildCards`` (binding with ``{..}``) you can mimic the
behavior of ``SELECT *``.
