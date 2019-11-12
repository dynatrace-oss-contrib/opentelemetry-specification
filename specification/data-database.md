# Database (SQL and NoSQL) client calls

> 🚧 WORK IN PROGRESS

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [Connection-level attributes](#connection-level-attributes)
- [Call-level attributes](#call-level-attributes)
- [Notes for specific database types](#notes-for-specific-database-types)

<!-- tocstop -->

For database client call the `SpanKind` MUST be `Client`.

Span `name` should be set to low cardinality value representing the statement
executed on the database. It may be stored procedure name (without argument), SQL
statement without variable arguments, etc. When it's impossible to get any
meaningful representation of the span `name`, it can be populated using the same
value as `db.name`.

## Connection-level attributes

These attributes will usually be the same for all operations performed over the same database connection. Some database systems may allow a connection to switch to a different `db.user`, and other database systems may not even have the concept of a connection though.

| Attribute name | Notes and examples                                           | Required? |
| :------------- | :----------------------------------------------------------- | --------- |
| `component` | Always the string `"db"` | Yes       |
| `db.type` | The lower-case database category, e.g. `"sql"`, `"cassandra"`, `"hbase"`, or `"redis"`. | Yes       |
| `db.tech` | Name of the database technology used, e.g. "SQL Server" or "MySQL". | No     |
| `db.url` | The connection string used to connect to the database | No       |
| `db.user`      | Username for accessing database. E.g., `"readonly_user"` or `"reporting_user"` | No        |
| `db.portname` | A name that identifies the `net.peer.port` used to connect. This property is intended to be used for [MS SQL Server instances][] but can be used for any database system that provides a name to port mapping. | See below        |

[MS SQL Server instances]: https://docs.microsoft.com/en-us/sql/connect/jdbc/building-the-connection-url?view=sql-server-ver15

Additionally at least one of `net.peer.name` or `net.peer.ip` from the [network attributes][] is required and `net.peer.port` is recommended.
If using a non-standard port for the `db.tech`, at least one of `net.peer.port` or `db.portname` is required.

[network attributes]: data-span-general.md#general-network-connection-attributes

## Call-level attributes

These attributes may be different for each operation performed even if the same connection is used for multiple operations. Usually only one `db.name` will be used per connection though.

| Attribute name | Notes and examples                                           | Required? |
| :------------- | :----------------------------------------------------------- | --------- |
| `db.name`  | Database name. E.g., In Java, if the jdbc.url=`"jdbc:mysql://db.example.com:3306/customers"`, the name is `"customers"`. For commands that switch the database, this should be set to the target database (even if the command fails). | Yes (if applicable) |
| `db.statement` | A database statement for the given database type. Note that the value may be sanitized to exclude sensitive information. E.g., for `db.type="sql"`, `"SELECT * FROM wuser_table"`; for `db.type="redis"`, `"SET mykey 'WuValue'"`. | Yes (if applicable)       |
| `db.op` | The type of operation that is executed, e.g. the [MongoDB command name][] such as `findAndModify`. While it would semantically make sense to set this e.g. to an SQL keyword like `SELECT` or `INSERT`, it is *not* recommended to attempt any client-side parsing of `db.statement` just to get this property (the back end can do that if required). | If `db.statement` is not applicable.       |

[MongoDB command name]: https://docs.mongodb.com/manual/reference/command/#database-operations

## Notes for specific database types

In some **SQL** databases, `db.name` is called "schema name".

In **Cassandra**, `db.name` should be set to the keyspace name.

In **CouchDB**, `db.op` should be set to the HTTP method + the target REST route according to the API reference documentation.
For example, when retrieving a document, `db.op` would be set to (literally, i.e., without replacing the placeholders with concrete values): [`GET /{db}/{docid}`][CouchDB get doc].

[CouchDB get doc]: http://docs.couchdb.org/en/stable/api/document/common.html#get--db-docid

In **HBase**, `db.name` is the [namespace][hbase ns].

[hbase ns]: https://hbase.apache.org/book.html#_namespace

**Redis** does not have a database name.

[rpc]: #remote-procedure-calls