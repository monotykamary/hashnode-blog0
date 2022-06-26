## A merge upsert pattern for TimescaleDB

*This post was motivated after a migration to TimescaleDB to support higher ingest rates for our event stores, logs, and journals. We normally avoid and set constraints to prevent updates to hypertables, except for one table we use to show to the user. This post re-explores our story in the need for generalizing a safe upsert pattern on TimescaleDB. The implementations mentioned in this article related to SQL and Golang.*

## Introduction

Single upserts in SQL are pretty boring, and that's good. Currently, the easiest way to do an upsert on PostgreSQL is to use `INSERT... ON CONFLICT... DO UPDATE`.

```sql
/* https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-upsert/ */
INSERT INTO customers (name, email)
VALUES('Microsoft','hotline@microsoft.com') 
ON CONFLICT (name) 
DO 
   UPDATE SET email = EXCLUDED.email || ';' || customers.email;
```

You get a similar interface with MySQL when you use `INSERT... ON DUPLICATE KEY UPDATE`. In both cases, you set up unique constraints on the columns that need uniqueness, run the query, and you're good to go.

## Our problem

It wasn't until fairly recently that TimescaleDB supported upserts using PostgreSQL's `INSERT... ON CONFLICT` statement and added documentation for it:

https://github.com/timescale/timescaledb/issues/1094
https://github.com/timescale/timescaledb/pull/4405
https://docs.timescale.com/timescaledb/latest/how-to-guides/write-data/upsert/#upsert-data

The additional requirement for upserts in TimescaleDB requires us to create a unique constraint that includes the partitioning column (in this case our `time` column).

```sql
/* https://docs.timescale.com/timescaledb/latest/how-to-guides/write-data/upsert/#create-a-table-with-a-unique-constraint */
ALTER TABLE conditions
  ADD CONSTRAINT conditions_time_location
    UNIQUE (time, location);
```

```sql
/* https://docs.timescale.com/timescaledb/latest/how-to-guides/write-data/upsert/#insert-or-do-nothing-to-a-table-with-a-unique-constraint */
INSERT INTO conditions
  VALUES ('2017-07-28 11:42:42.846621+00', 'office', 70.2, 50.1)
  ON CONFLICT (time, location) DO UPDATE
    SET temperature = excluded.temperature,
        humidity = excluded.humidity;
```

This looks great on paper, until you realize this **doesn't work for inserting records with the latest time.** For that, you would need to use **two models to read and update**: the first model to find a record with the old time, with the other model used to insert the new time when the old time isn't found. However, using two models here is simply not possible with `INSERT... ON CONFLICT` as there is nothing in the statement that allows us to specify a separate read condition.

## The `MERGE` statement

The `MERGE` statement is upsert on steroids. It allows us to specify what conditions to match and find before an `UPDATE` or `INSERT` (+ `DELETE`). `MERGE` would handle all the concurrency quirks of an upsert for us and we would just need to specify a few conditions and outputs.

```sql
/* https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=08ea7a2291db21a618d19d612c8060cda68f1892 */
MERGE INTO target AS t
USING source AS s
ON t.tid = s.sid
WHEN MATCHED AND t.balance > s.delta THEN
  UPDATE SET balance = t.balance - s.delta
WHEN MATCHED THEN
  DELETE
WHEN NOT MATCHED AND s.delta > 0 THEN
  INSERT VALUES (s.sid, s.delta)
WHEN NOT MATCHED THEN
  DO NOTHING;
```
The thing is, `MERGE` isn't actually supported on PostgreSQL yet 🤦. We **almost** got `MERGE` in 2018, before it was reverted due to a design issue: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=08ea7a2291db21a618d19d612c8060cda68f1892.

But now it came back and had a redemption arc; it is now reimplemented in PostgreSQL 15 🙌: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=7103ebb7aae8ab8076b7e85f335ceb8fe799097c.

### And another problem

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1656152169843/HxRH7yP1Y.png align="left")

PostgreSQL 15 isn't generally available (GA) yet 🤦. On top of that, we would still have to wait for it to be ported to TimescaleDB and then have the `MERGE` feature also be supported on hypertables. This is going to be a long one bois.

## Doing it ourselves?

Although, at first glance, it seems rather trivial to do an upsert. Our only quirk here is that our `time` column is determined by time, which makes it subtly non-deterministic from the perspective of the developer. We would need to do the following to achieve our upsert:

- Create an object with our model struct with our updated fields
- Query the record with the unique IDs other than our partitioning column
  -  If the record exists:
    - Merge our model object with the record (we take the query’s value for our partitioning column)
    - UPDATE the record with the updated fields
  - If the record does not exist:
    - INSERT the record with the updated fields

Doing this on the application side, and even on the SQL side comes with some caveats. We can't use this approach without considering strategies that prevent us from reaching concurrency issues or race conditions. You can see a handful mentioned in this article: https://www.depesz.com/2012/06/10/why-is-upsert-so-complicated/. The strategies mentioned there are inefficient (maybe except for advisory locks), don’t consider rollbacks, and have nuanced tradeoffs that we can’t afford to discover during production.

## Further research

Reading a bit more, there are a few mentions on using a staging table as a method to perform a merge upsert.

https://stackoverflow.com/questions/17267417/how-to-upsert-merge-insert-on-duplicate-update-in-postgresql
https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-upsert.html

Although it's mentioned on both sources that it's focused on bulk upserts, it's also good for our single upsert case, albeit being much slower. This method avoids us having to query the record with `SELECT` to create our conditions by implying the conditions on matching IDs in our `WHERE` clauses, avoiding a whole host of mental overhead surrounding concurrency.

This was probably the best alternative so far.

## Implementation

Creating a staging table for merge sounds pretty easy, as we also have instructions on our sources as well. However, we have a few extra small cases in our project that we also need to include in the implementation.

### Staging upserts with locks

Along with the detail that we use IDENTITY columns to reference our IDs, our implementation would look something like this: 

- `BEGIN` our transaction
- `CREATE` a `TEMPORARY` table mimicking the target table (and exclude constraints)
- `LOCK` the target table `IN EXCLUSIVE MODE`. This permits other transactions to `SELECT`, but not make any changes to the table.
- (Set the identity sequence to mimic the target table)
- Copy or bulk-insert the new data into the temp table
- Do an `UPDATE ... FROM` of existing records using the values in the temp table;
- Do an `INSERT` of rows that don't already exist in the target table;
- `COMMIT`, releasing the lock.

This is what it would look like in SQL:

```sql
BEGIN;

CREATE TEMPORARY TABLE staging_transaction_log (LIKE transaction_log EXCLUDING CONSTRAINTS) ON COMMIT DROP;

LOCK TABLE transaction_log IN EXCLUSIVE MODE;

SELECT
    setval(
        pg_get_serial_sequence('staging_transaction_log', 'id'),
        (
            SELECT
                last_value
            FROM
                transaction_log_id_seq
        )
    );
    
INSERT INTO
    "staging_transaction_log" (
        "type",
        "type_name",
        "owner_address",
        "t1_amount",
        "t2_amount",
        "version",
        "status",
        "request_id",
        "hash",
        "time"
    )
VALUES
    (
        'salary_claimed',
        'CLAIM_SALARY',
        '....',
        '0',
        '1723.68',
        0,
        'processsing',
        75,
        '',
        '2022-05-31 19:31:59.72'
    ) RETURNING "id";

UPDATE transaction_log
SET request_id = staging_transaction_log.request_id
FROM staging_transaction_log
WHERE staging_transaction_log.id = transaction_log.id;

INSERT INTO
    transaction_log(
        type,
        type_name,
        owner_address,
        t1_amount,
        t2_amount,
        version,
        status,
        request_id,
        hash,
        time
    )
SELECT
    staging_transaction_log.type,
    staging_transaction_log.type_name,
    staging_transaction_log.owner_address,
    staging_transaction_log.t1_amount,
    staging_transaction_log.t2_amount,
    staging_transaction_log.version,
    staging_transaction_log.status,
    staging_transaction_log.request_id,
    staging_transaction_log.hash,
    staging_transaction_log.time
FROM staging_transaction_log
LEFT OUTER JOIN transaction_log ON (
    transaction_log.id = staging_transaction_log.id
)
WHERE transaction_log.id IS NULL;

COMMIT;
```

### Implementing it on Golang (with Gorm)

*Note: The details outside of the implementation do not include everything. If you wish to implement this in your own project, YMMV.*

Our project uses Gorm, and it's pretty fun to use. Implementing this pattern on Golang and Gorm is pretty easy. We do need to be careful of column name and table names as well as syntax errors when composing strings.

We have a common struct to set our merge options to check the following:
- **identity**: special for our case, check whether the `id` column is a `SERIAL`, `IDENTITY`, or a column that `AUTO INCREMENTS`.
- **constraints**: the column(s) we're searching for.
- **columns**: the column(s) we're updating.

```go
package model

type MergeOptions struct {
	/*
		Strictly for the `id` column; this bool identifies whether we have a SERIAL, IDENTITY,
		or a column that AUTO INCREMENTS.
	*/
	Identity bool
	/*
		A list of constraints to be used as conditions to find records that have matching values
		respective to the matching columns specified in our constraints. If no constraint is
		specified, `id` will be used by default to compare uniqueness.
	*/
	Constraints []string
	/*
		A list of columns to be updated.
	*/
	Columns []string
}

```

```go
package pg

import (
	"fmt"
	"reflect"
	"regexp"
	"strings"

	"github.com/.../api/model"
	"gorm.io/gorm"
)

// used to get JSON fields from our model which are tied to our table column names
func getJSONFields(param interface{}, omittedFields map[string]interface{}) []string {
	fields := []string{}
	val := reflect.ValueOf(param)
	for i := 0; i < val.Type().NumField(); i++ {
		field := val.Type().Field(i).Tag.Get("json")
		_, exists := omittedFields[field]
		if !exists {
			fields = append(fields, field)
		}
	}
	return fields
}

func composeMerge(db *gorm.DB, tableName string, param interface{}, options model.MergeOptions) string {
	// create a staging table
	stagingTableName := fmt.Sprintf("staging_%v", tableName)
	createStagingTable := fmt.Sprintf(
		`CREATE TEMPORARY TABLE %v (LIKE %v EXCLUDING CONSTRAINTS) ON COMMIT DROP;`,
		stagingTableName,
		tableName,
	)

	// lock the target table in exclusive mode
	lockTable := fmt.Sprintf("LOCK TABLE %v IN EXCLUSIVE MODE;", tableName)

	// set the increment identity value of the staging id to the target id
	setStagingIdentity := ""
	if options.Identity {
		setStagingIdentity = fmt.Sprintf(`
			SELECT setval(pg_get_serial_sequence('%v', 'id'), (SELECT last_value FROM %v_id_seq));
		`,
			stagingTableName,
			tableName,
		)
	}

	// insert our values to the staging table
	re := regexp.MustCompile(tableName)
	insertToStagingTable := db.ToSQL(func(tx *gorm.DB) *gorm.DB {
		return tx.Create(param)
	})
	insertToStagingTable = re.ReplaceAllString(insertToStagingTable, stagingTableName) + ";"

	// compose our constraint values
	updateConstraints := fmt.Sprintf("%v.id = %v.id", stagingTableName, tableName)
	insertConstraints := fmt.Sprintf("%v.id IS NULL", tableName)
	insertJoinConstraints := fmt.Sprintf("%v.id = %v.id", tableName, stagingTableName)

	if len(options.Constraints) > 0 {
		updateConstraints = ""
		insertConstraints = ""
		insertJoinConstraints = ""

		for _, constraint := range options.Constraints {
			updateConstraints += fmt.Sprintf("%v.%v = %v.%v AND ", stagingTableName, constraint, tableName, constraint)
			insertConstraints += fmt.Sprintf("%v.%v IS NULL AND ", tableName, constraint)
			insertJoinConstraints = fmt.Sprintf("%v.%v = %v.%v AND ", tableName, constraint, stagingTableName, constraint)
		}
		updateConstraints = strings.TrimSuffix(updateConstraints, " AND ")
		insertConstraints = strings.TrimSuffix(insertConstraints, " AND ")
		insertJoinConstraints = strings.TrimSuffix(insertJoinConstraints, " AND ")
	}

	// attempt to update the target table or ignore if there are no set columns
	setColumns := ""
	attemptUpdateTable := ""
	if len(options.Columns) > 0 {
		for _, column := range options.Columns {
			setColumns += fmt.Sprintf("%v = %v.%v,", column, stagingTableName, column)
		}
		setColumns = strings.TrimSuffix(setColumns, ",")

		attemptUpdateTable = fmt.Sprintf(`
			UPDATE %v
			SET %v
			FROM %v
			WHERE %v;
		`,
			tableName,
			setColumns,
			stagingTableName,
			updateConstraints,
		)
	}

	// attempt to insert to the target table; ignore id to allow generation
	insertJSONFields := getJSONFields(param, map[string]interface{}{"id": true})
	insertFields := ""
	for _, insertField := range insertJSONFields {
		insertFields += fmt.Sprintf("%v.%v,", stagingTableName, insertField)
	}
	insertFields = strings.TrimSuffix(insertFields, ",")
	tableInsertColumns := fmt.Sprintf("%v(%v)", tableName, strings.Join(insertJSONFields, ","))

	attemptInsertTable := fmt.Sprintf(`
		INSERT INTO %v
		SELECT %v
		FROM %v
		LEFT OUTER JOIN %v ON (%v)
		WHERE %v;
	`,
		tableInsertColumns,
		insertFields,
		stagingTableName,
		tableName,
		insertJoinConstraints,
		insertConstraints,
	)

	return createStagingTable + lockTable + setStagingIdentity + insertToStagingTable + attemptUpdateTable + attemptInsertTable
}
```

We then create a common `Upsert` function linked to our repo:

```go
package pg

import (
	"github.com/.../api/model"
	"github.com/.../api/model/errors"
	"github.com/.../api/repo"
	"gorm.io/gorm"
)

type transactionLogRepo struct{}

func NewTransactionLogRepo() repo.TransactionLogRepo {
	return &transactionLogRepo{}
}

func (ot *transactionLogRepo) Upsert(s repo.DBRepo, param model.TransactionLog, options model.MergeOptions) error {
	db := s.DB()

    // check that we're inside a transaction
	committer, ok := db.Statement.ConnPool.(gorm.TxCommitter)
	if ok && committer == nil {
		return errors.ErrInternalServerError
	}

	tableName := param.TableName()
	query := composeMerge(db, tableName, param, options)

	return db.Exec(query).Error
}
```

and then we run the function through our service and wrap it inside a transaction:

```go
...
	ctx, cancel := context.WithTimeout(context.Background(), consts.DefaultLockTime*time.Second)
	defer cancel()

	tx, done := uc.store.NewTransactionWithContext(ctx)

	err = uc.repo.TransactionLog.Upsert(tx, model.TransactionLog{
		Type:         consts.ActionDepositToken,
		TypeName:     "DEPOSIT",
		OwnerAddress: param.OwnerAddress,
		T1Amount:     amountT1,
		T2Amount:     amountT2,
		Version:      0,
		Status:       consts.TransactionLogStatusProcessing,
		Hash:         param.Hash,
		RequestId:    request.Id,
		Time:         time.Now(),
	}, model.MergeOptions{
		Identity:    true,
		Constraints: []string{"hash"},
		Columns:     []string{"request_id", "status"},
	})
	if err != nil {
		zap.L().Sugar().Errorf("uc.repo.TransactionLog.Upsert(): %v", err)
		return done(errors.ErrInternalServerError)
	}

	return done(nil)
...
```

And now we have a pretty ergonomic upsert function, abstracted in a way that will also allow future implementation of the `MERGE` statement in the future, if we ever get there.

### Tradeoffs

Of course, no implementation is without its tradeoffs.

**On the application side**, the function we've created to do upserts does single record upserts pretty safely. The `composeMerge` function currently doesn't allow batch upserts, but that can be easily updated or separated into a separate compose function. The `MergeOptions` are actually a gateway for a pretty nasty SQL injection, but we currently don't allow any outside user inputs for these options. Although, we should get around to sanitizing them.

**On the SQL side**, we've significantly slowed down our upsert by spending CPU and memory in creating a staging table, merging 2 tables through an implicit `WHERE` diff, and wiping the staging table `ON COMMIT`. It might be a different story for bulk upserts, but we're not there yet.

Our implementation also means **we don't actually need unique constraints for this upsert to work**, meaning it won't fail if we don't have unique constraints on the columns we want to match and compare. This will allow multiple similar record upserts; which could be a good thing depending on your use case. It's up to the developer or database administrator to include constraints to maintain the consistency of the operation of this upsert function.

## Conclusion

This is one of stories we have in our project when working and migrating to TimescaleDB. The `INSERT... ON CONFLICT... DO UPDATE` statement we use quite often on PostgreSQL, became a bit tricky to use here, which motivated this whole research and implementation journey. There's still a bit to improve here on the app side, but so far we're pretty happy with the implementation and we haven't met a case where this has failed us (yet).

## Other related sources

https://www.cybertec-postgresql.com/en/postgresql-15-using-merge-in-sql/