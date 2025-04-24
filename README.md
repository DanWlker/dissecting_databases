# Dissecting Databases

## General (I will still use postgres as an example, but it should apply to most other DBs)

1. There are several restrictions on `GENERATED ALWAYS AS (expression) STORED`, below are some that occur usually, but the website is more exhaustive: https://www.postgresql.org/docs/current/ddl-generated-columns.html
    - Immutable functions only, no subqueries or reference to anything other than current row
    - Cannot reference other generated column (This applies to postgres but is different in for ex. [MySql](https://dev.mysql.com/doc/refman/8.4/en/create-table-generated-columns.html))
    - Cannot reference a system column EXCEPT `tableoid`
    - Cannot have a column default or identity definition (Mainly because for `DEFAULT` and `IDENTITY` the format is `GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY [ ( sequence_options )`, it is a completely different thing)
    - Cannot be part of a partition key
    - Generated columns are, conceptually, updated after `BEFORE` triggers have run. Therefore, changes made to base columns in a `BEFORE` trigger will be reflected in generated columns. But conversely, it is not allowed to access generated columns in `BEFORE` triggers.
  
1. [Immutable vs Stable vs Volatile functions](https://www.postgresql.org/docs/current/xfunc-volatility.html)
   - Immutable
       - cannot modify database
       - always returns the same result when given same argument values, forever
       - DOES NOT DO DATABASE LOOKUPS or use information not directly present in its argument list
       - optimizer can pre-evaluate the function when a query calls with constant arguments
   - Stable
       - cannot modify the database, but result depends on database lookups, param variables (ex. timezone)
       - within a single statement it will return the same result for the same argument values
       - optimizer will optimize multiple calls to a single call
       - note that `current_timestamp` family of functions is considered stable, as their values don't change within a transaction
   - Volatile
       - aka. go wild anything is permitted
       - can return different results on successive calls with same arguments in the same query
       - ex. random(), currval(), timeofday()
       - any functions that have side effects must be classified as volatile
       - will be evaluated at every row where its value is needed
       - optimizer will not do anything
    
    From the docs (yes i know its long but its good):
    
    > There is relatively little difference between STABLE and IMMUTABLE categories when considering simple interactive queries that are planned and
    > immediately executed: it doesn't matter a lot whether a function is executed once during planning or once during query execution startup.
    >
    > But there is a big difference if the plan is saved and reused later. Labeling a function IMMUTABLE when it really isn't might allow it to be
    > prematurely folded to a constant during planning, resulting in a stale value being re-used during subsequent uses of the plan. This is a hazard when
    > using prepared statements or when using function languages that cache plans (such as PL/pgSQL).
    >
    > For functions written in SQL or in any of the standard procedural languages, there is a second important property determined by the volatility
    > category, namely the visibility of any data changes that have been made by the SQL command that is calling the function. A VOLATILE function will see
    > such changes, a STABLE or IMMUTABLE function will not. This behavior is implemented using the snapshotting behavior of MVCC (see Chapter 13): STABLE
    > and IMMUTABLE functions use a snapshot established as of the start of the calling query, whereas VOLATILE functions obtain a fresh snapshot at the
    > start of each query they execute.
    >
    > Because of this snapshotting behavior, a function containing only SELECT commands can safely be marked STABLE, even if it selects from tables that
    > might be undergoing modifications by concurrent queries. PostgreSQL will execute all commands of a STABLE function using the snapshot established for
    > the calling query, and so it will see a fixed view of the database throughout that query.
    >
    > The same snapshotting behavior is used for SELECT commands within IMMUTABLE functions. It is generally unwise to select from database tables within an
    > IMMUTABLE function at all, since the immutability will be broken if the table contents ever change. However, PostgreSQL does not enforce that you do
    > not do that.
    >
    > A common error is to label a function IMMUTABLE when its results depend on a configuration parameter. For example, a function that manipulates
    > timestamps might well have results that depend on the TimeZone setting. For safety, such functions should be labeled STABLE instead.
    >
    > PostgreSQL requires that STABLE and IMMUTABLE functions contain no SQL commands other than SELECT to prevent data modification. (This is not a
    > completely bulletproof test, since such functions could still call VOLATILE functions that modify the database. If you do that, you will find that the
    > STABLE or IMMUTABLE function does not notice the database changes applied by the called function, since they are hidden from its snapshot.)

    More examples:
    - [`concat` is stable but not immutable](https://www.postgresql.org/message-id/3361.1410026366%40sss.pgh.pa.us)  because for stuff like `TIMESTAMPZ`, its output depends on the timezone. Unless you do something like [this](https://stackoverflow.com/questions/54372666/create-an-immutable-clone-of-concat-ws/54384767#54384767) where it only accepts text, then it is immutable.
    - [The same thing applies to `to_char`](https://dba.stackexchange.com/questions/77272/why-isnt-to-char-immutable-and-how-can-i-work-around-it)
  
1. Insert Into Select
   - Normally this is used to copy data from one table and insert it into another
     ```sql
     INSERT INTO Customers (CustomerName, City, Country)
     SELECT SupplierName, City, Country FROM Suppliers;
     ```
   - However it could also be repurposed to be similar to Upsert but without the index requirement (but there is a reason why it is required so you should know the tradeoffs).
   - Also has the advantage where you can Insert if not exist, making the operation atomic and preventing race conditions (you don't have to split finding and inserting into two queries, though upsert works as well)
   - Add another column `insert_success` to help determine if the row returned is from insert or existing
     ```sql 
		with insert_not_exist as (
			insert into your_table(col1, col2) 
			select :col1, :col2
			where not exists (
				select 1 
				from your_table 
				where
                 col1 = :col1 and
                 col2 = :col2
			)
			returning *, true as insert_success -- this tells you if the row returned is from insert or existing
		)
		select *
		from insert_not_exist
		union
		select *, false as insert_success
		from your_table
		where
             col1 = :col1 and
             col2 = :col2
     ```
1. You can create [partial unique indexes](https://www.postgresql.org/docs/current/indexes-partial.html) which is indexes that only apply when a condition satisfies

   > A partial index is an index built over a subset of a table; the subset is defined by a conditional expression (called the predicate of the partial index). The index contains entries only for those table rows that satisfy the predicate.

   This also helps when we want partial but unique indexes
   
   > We wish to ensure that there is only one “successful” entry for a given subject and target combination, but there might be any number of “unsuccessful” entries.

   ```sql
   CREATE TABLE tests (
   	subject text,
   	target text,
   	success boolean,
   	...
   );
	
   CREATE UNIQUE INDEX tests_success_constraint ON tests (subject, target)
   	WHERE success;
   ```

## Postgres specific

1. [There is no (or negligable) difference between varchar and text](https://stackoverflow.com/questions/4848964/difference-between-text-and-varchar-character-varying)
    - But, if you need to index it, it may bring some [concerns](https://stackoverflow.com/a/49774665)
    - [A benchmark](https://www.depesz.com/2010/03/02/charx-vs-varcharx-vs-varchar-vs-text/)
    - [Another benchmark with recommendation](https://stackoverflow.com/a/36806014)
    - [From the postgres wiki, see tip section](https://www.postgresql.org/docs/current/datatype-character.html)

1. [Single column value hard limit is 1GB](https://stackoverflow.com/a/39966079)

1. [Maximum size for row, table, db from postgres wiki](https://wiki.postgresql.org/wiki/FAQ#What_is_the_maximum_size_for_a_row.2C_a_table.2C_and_a_database.3F)

1. Upsert [On Conflict clause](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT)
   - `conflict_targets` can be:
       - One or more columns of the table (e.g., column1, column2)
       - Expressions on columns (e.g., LOWER(column1) or column1 + column2)
   - However, they must be unique indexes. PostgreSQL tries to automatically infer the unique index that applies to the `conflict_target`. It looks at all unique indexes on the table and finds one that matches the columns or expressions specified in the `conflict_target`
   - If `conflict_targes` is a partial unique index, it will only be used if the `index_predicate` matches the predicate of the index. Ex.
       - You have a partial index:
         ```sql
         CREATE UNIQUE INDEX username_active_unique ON users (username) WHERE active = true;
         ```
       - If you want to use this index in an ON CONFLICT clause, you would need to specify an `index_predicate`
         ```sql
         INSERT INTO users (username, email, active)
         VALUES ('user1', 'user@example.com', true)
         ON CONFLICT (username) WHERE active = true DO UPDATE SET email = EXCLUDED.email;
         ```
         Here, the index_predicate is WHERE active = true. PostgreSQL will use the partial index on username (but only when active = true) to handle the conflict
         
   - For `ON CONFLICT DO NOTHING`, it is **optional** to specify a `conflict_target`; when omitted, conflicts with all usable constraints (and unique indexes) are handled. It means that PostgreSQL will check for conflicts against all usable constraints and unique indexes on the table. So, if the table has multiple unique constraints or unique indexes, PostgreSQL will consider all of them for conflict detection.

1. Postgres Upsert with `RETURNING` clause will not return if the insert conflicts (aka. no insert is done). To return data, try [this](https://stackoverflow.com/a/42217872) 

1. [`NULL DISTINCT`](https://www.postgresql.org/docs/current/indexes-unique.html) is default when creating indexes, meaning null values in a unique column are not considered equal, allowing multiple nulls in the column. Use `NULLS NOT DISTINCT` to modify this behaviour
  
1. Use `SELECT current_database()` to see the name of the current database [1](https://dba.stackexchange.com/a/58332), `\conninfo` works as well [2](https://dba.stackexchange.com/a/78670). To see all databases use `SELECT * from pg_database`

1. Postgres can have multiple databases [1](https://www.postgresql.org/docs/current/manage-ag-overview.html)

1. [How to clone a postgres db](https://medium.com/preprintblog/how-to-copy-a-postgresql-database-to-a-new-one-in-the-same-instance-on-aws-rds-a632a1b7e83c)
