## Akka.Persistence.PostgreSql

Akka Persistence journal and snapshot store backed by PostgreSql database.

**WARNING: Akka.Persistence.PostgreSql plugin is still in beta and it's mechanics described bellow may be still subject to change**.

### Configuration

Both journal and snapshot store share the same configuration keys (however they resides in separate scopes, so they are definied distinctly for either journal or snapshot store):

Remember that connection string must be provided separately to Journal and Snapshot Store.

```hocon
akka.persistence{
	journal {
		plugin = "akka.persistence.journal.postgresql"
		postgresql {
			# qualified type name of the PostgreSql persistence journal actor
			class = "Akka.Persistence.PostgreSql.Journal.PostgreSqlJournal, Akka.Persistence.PostgreSql"

			# dispatcher used to drive journal actor
			plugin-dispatcher = "akka.actor.default-dispatcher"

			# connection string used for database access
			connection-string = ""

			# default SQL commands timeout
			connection-timeout = 30s

			# PostgreSql schema name to table corresponding with persistent journal
			schema-name = public

			# PostgreSql table corresponding with persistent journal
			table-name = event_journal

			# should corresponding journal table be initialized automatically
			auto-initialize = off
			
			# timestamp provider used for generation of journal entries timestamps
			timestamp-provider = "Akka.Persistence.Sql.Common.Journal.DefaultTimestampProvider, Akka.Persistence.Sql.Common"
		
			# metadata table
			metadata-table-name = metadata

			# defines column db type used to store payload. Available option: BYTEA (default), JSON, JSONB
			stored-as = BYTEA
		}
	}

	snapshot-store {
		plugin = "akka.persistence.snapshot-store.postgresql"
		postgresql {
			# qualified type name of the PostgreSql persistence journal actor
			class = "Akka.Persistence.PostgreSql.Snapshot.PostgreSqlSnapshotStore, Akka.Persistence.PostgreSql"

			# dispatcher used to drive journal actor
			plugin-dispatcher = ""akka.actor.default-dispatcher""

			# connection string used for database access
			connection-string = ""

			# default SQL commands timeout
			connection-timeout = 30s

			# PostgreSql schema name to table corresponding with persistent journal
			schema-name = public

			# PostgreSql table corresponding with persistent journal
			table-name = snapshot_store

			# should corresponding journal table be initialized automatically
			auto-initialize = off
			
			# defines column db type used to store payload. Available option: BYTEA (default), JSON, JSONB
			stored-as = BYTEA
		}
	}
}
```

### Batching journal

Since version 1.1.3 an alternative, experimental type of the journal has been released, known as batching journal. It's optimized for concurrent writes made by multiple persistent actors, thanks to the ability of batching multiple SQL operations to be executed within the same database connection. In some of those situations we've noticed over an order of magnitude in event write speed.

To use batching journal, simply change `akka.persistence.journal.sql-server.class` to *Akka.Persistence.SqlServer.Journal.BatchingSqlServerJournal, Akka.Persistence.SqlServer*.

Additionally to the existing settings, batching journal introduces few more:

- `isolation-level` to define isolation level for transactions used withing event reads/writes. Possible options: *unspecified* (default), *chaos*, *read-committed*, *read-uncommitted*, *repeatable-read*, *serializable* or *snapshot*.
- `max-concurrent-operations` is used to limit the maximum number of database connections used by this journal. You can use them in situations when you want to partition the same ADO.NET pool between multiple components. Current default: *64*.
- `max-batch-size` defines the maximum number of SQL operations, that are allowed to be executed using the same connection. When there are more operations, they will chunked into subsequent connections. Current default: *100*.
- `max-buffer-size` defines maximum buffer capacity for the requests send to a journal. Once buffer gets overflown, a journal will call `OnBufferOverflow` method. By default it will reject all incoming requests until the buffer space gets freed. You can inherit from `BatchingSqlServerJournal` and override that method to provide a custom backpressure strategy. Current default: *500 000*.

### Table Schema

PostgreSql persistence plugin defines a default table schema used for journal, snapshot store and metadate table.

```SQL
CREATE TABLE {your_journal_table_name} (
	ordering BIGSERIAL NOT NULL PRIMARY KEY,
    persistence_id VARCHAR(255) NOT NULL,
    sequence_nr BIGINT NOT NULL,
    is_deleted BOOLEAN NOT NULL,
    created_at BIGINT NOT NULL,
    manifest VARCHAR(500) NOT NULL,
    payload BYTEA NOT NULL,
    tags VARCHAR(100) NULL,
    CONSTRAINT {your_journal_table_name}_uq UNIQUE (persistence_id, sequence_nr)
);

CREATE TABLE {your_snapshot_table_name} (
    persistence_id VARCHAR(255) NOT NULL,
    sequence_nr BIGINT NOT NULL,
    created_at BIGINT NOT NULL,
    manifest VARCHAR(500) NOT NULL,
    snapshot BYTEA NOT NULL,
    CONSTRAINT {your_snapshot_table_name}_pk PRIMARY KEY (persistence_id, sequence_nr)
);

CREATE TABLE {your_metadata_table_name} (
    persistence_id VARCHAR(255) NOT NULL,
    sequence_nr BIGINT NOT NULL,
    CONSTRAINT {your_metadata_table_name}_pk PRIMARY KEY (persistence_id, sequence_nr)
);
```

### Migration

#### From 1.0.6 to 1.1.0
```SQL
CREATE TABLE {your_metadata_table_name} (
    persistence_id VARCHAR(255) NOT NULL,
    sequence_nr BIGINT NOT NULL,
    CONSTRAINT {your_metadata_table_name}_pk PRIMARY KEY (persistence_id, sequence_nr)
);

ALTER TABLE {your_journal_table_name} DROP CONSTRAINT {your_journal_table_name}_pk;
ALTER TABLE {your_journal_table_name} ADD COLUMN ordering BIGSERIAL NOT NULL PRIMARY KEY;
ALTER TABLE {your_journal_table_name} ADD COLUMN tags VARCHAR(100) NULL;
ALTER TABLE {your_journal_table_name} ADD CONSTRAINT {your_journal_table_name}_uq UNIQUE (persistence_id, sequence_nr);
ALTER TABLE {your_journal_table_name} ADD COLUMN created_at_temp BIGINT NOT NULL;

UPDATE {your_journal_table_name} SET created_at_temp=extract(epoch from create_at);

ALTER TABLE {your_journal_table_name} DROP COLUMN create_at;
ALTER TABLE {your_journal_table_name} DROP COLUMN created_at_ticks;
ALTER TABLE {your_journal_table_name} RENAME COLUMN created_at_temp TO create_at;
```

### Tests

The PostgreSql tests are packaged and run as part of the default "All" build task.

In order to run the tests, you must do the following things:

1. Download and install PostgreSql from: http://www.postgresql.org/download/
2. Install PostgreSql with the default settings.  The default connection string uses the following credentials:
  1. Username: postgres
  2. Password: postgres
3. A custom app.config file can be used and needs to be placed in the same folder as the dll
