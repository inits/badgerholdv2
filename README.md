# BadgerHold
[![Build Status](https://travis-ci.org/timshannon/badgerhold.svg?branch=master)](https://travis-ci.org/timshannon/badgerhold) [![GoDoc](https://godoc.org/github.com/timshannon/badgerhold?status.svg)](https://godoc.org/github.com/timshannon/badgerhold) [![Coverage Status](https://coveralls.io/repos/github/timshannon/badgerhold/badge.svg?branch=master)](https://coveralls.io/github/timshannon/badgerhold?branch=master) [![Go Report Card](https://goreportcard.com/badge/github.com/timshannon/badgerhold)](https://goreportcard.com/report/github.com/timshannon/badgerhold)


BadgerHold is a simple querying and indexing layer on top of a [Badger](https://github.com/dgraph-io/badger) instance. 
The goal is to create a simple, higher level interface on top of Badger DB that simplifies dealing with Go Types and finding data, but exposes the underlying
Badger DB for customizing as you wish.  By default the encoding used is Gob, so feel free to use the GobEncoder/Decoder
interface for faster serialization.  Or, alternately, you can use any serialization you want by supplying encode / decode
funcs to the `Options` struct on Open.

One Go Type will be prefixed with it's type name, so you can store multiple types in a single Badger database with
conflicts.

This project is a rewrite of the [BoltHold](https://github.com/timshannon/bolthold) project on the Badger KV database
instead of [Bolt](https://github.com/etcd-io/bbolt).  For a performance comparison between bolt and badger, see 
https://blog.dgraph.io/post/badger-lmdb-boltdb/.  I've written up my own comparison of the two focusing on 
characteristics *other* than performance here: https://tech.townsourced.com/post/boltdb-vs-badger/.

## Use badger aes example
```
package main

import (
	"fmt"
	"time"

	badgerhold "github.com/inits/badgerholdv2"

	badger "github.com/dgraph-io/badger/v2"
)

// Item ...
type Item struct {
	ID       int
	Category string `badgerholdIndex:"Category"` //建立索引，不知道实际效果是不是更快
	Created  time.Time
	Qiaos    string `badgerhold:"unique"` // 设定这个字段的值的唯一性
}

var data = []Item{
	{
		ID:       0,
		Category: "blue",
		Created:  time.Now().Add(-4 * time.Hour),
		Qiaos:    "puqiaoming3",
	},
	{
		ID:       1,
		Category: "red",
		Created:  time.Now().Add(-3 * time.Hour),
		Qiaos:    "puqiaoming",
	},
	{
		ID:       2,
		Category: "blue",
		Created:  time.Now().Add(-2 * time.Hour),
		Qiaos:    "puqiaoming2",
	},
	{
		ID:       3,
		Category: "blue",
		Created:  time.Now().Add(-20 * time.Minute),
		Qiaos:    "puqiaoming1",
	},
}

func main() {

	opts := badgerhold.DefaultOptions
	opts.Options = badger.DefaultOptions("/tmp/badgerhold/qms").WithEncryptionKey([]byte("1234567890123456"))
	//opts.Options = badger.DefaultOptions("/tmp/badgerhold/qms")

	store, err := badgerhold.Open(opts)
	if err != nil {
		fmt.Println(err)
	}

	defer store.Close()

	// TxInsert 写入数据库
	err = store.Badger().Update(func(tx *badger.Txn) error {
		for i := range data {
			fmt.Println(data[i].ID, data[i])
			err := store.TxInsert(tx, badgerhold.NextSequence(), data[i]) //badgerhold.NextSequence() 自增id
			if err != nil {
				fmt.Println("inset error:", err)
				return err
			}
		}
		return err
	})
	if err != nil {
		fmt.Println(err)
	}

	// Find 查找给定数据
	var result []Item
	err = store.Find(&result, badgerhold.Where("Category").Ne(""))
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(result)

	// TxDeleteMatching 删除匹配
	err = store.Badger().Update(func(tx *badger.Txn) error {
		err = store.TxDeleteMatching(tx, &Item{}, badgerhold.Where("Category").Eq("red"))
		if err != nil {
			fmt.Println(err)
			return err
		}
		return nil
	})
	if err != nil {
		fmt.Println("delete error:", err)
	}

	// TxUpdateMatching 删除匹配
	err = store.Badger().Update(func(tx *badger.Txn) error {
		store.TxUpdateMatching(tx, &Item{}, badgerhold.Where("Qiaos").In("puqiaoming2", "puqiaoming3"), func(record interface{}) error {
			update, ok := record.(*Item)
			if !ok {
				return fmt.Errorf("Record isn't the correct type!  Wanted Item, got %T", record)
			}
			fmt.Println("update struct:", update)
			update.Category = "yellows"
			return nil
		})

		return nil
	})

	if err != nil {
		fmt.Println("update1 error:", err)
	}

	// find all items in
	var result1 []Item
	err = store.Find(&result1, badgerhold.Where("Qiaos").Ne(""))
	if err != nil {
		fmt.Println(err)
	}

	fmt.Println(result1)
}


```

## Indexes
Indexes allow you to skip checking any records that don't meet your index criteria.  If you have 1000 records and only
10 of them are of the Division you want to deal with, then you don't need to check to see if the other 990 records match
your query criteria if you create an index on the Division field.  The downside of an index is added disk reads and writes
on every write operation.  For read heavy operations datasets, indexes can be very useful.

In every BadgerHold store, there will be a reserved bucket *_indexes* which will be used to hold indexes that point back
to another bucket's Key system.  Indexes will be defined by setting the `badgerhold:"index"` struct tag on a field in a type.

```Go
type Person struct {
	Name string
	Division string `badgerhold:"index"`
}

// alternate struct tag if you wish to specify the index name
type Person struct {
	Name string
	Division string `badgerholdIndex:"IdxDivision"`
}

```

This means that there will be an index created for `Division` that will contain the set of unique divisions, and the
main record keys they refer to. 

Optionally, you can implement the `Storer` interface, to specify your own indexes, rather than using the `badgerHoldIndex`
struct tag.

## Queries
Queries are chain-able constructs that filters out any data that doesn't match it's criteria. An index will be used if
the `.Index()` chain is called, otherwise BadgerHold won't use any index.

Queries will look like this:
```Go
s.Find(badgerhold.Where("FieldName").Eq(value).And("AnotherField").Lt(AnotherValue).Or(badgerhold.Where("FieldName").Eq(anotherValue)))

```

Fields must be exported, and thus always need to start with an upper-case letter.  Available operators include:
* Equal - `Where("field").Eq(value)`
* Not Equal - `Where("field").Ne(value)`
* Greater Than - `Where("field").Gt(value)`
* Less Than - `Where("field").Lt(value)`
* Less than or Equal To - `Where("field").Le(value)`
* Greater Than or Equal To - `Where("field").Ge(value)`
* In - `Where("field").In(val1, val2, val3)`
* IsNil - `Where("field").IsNil()`
* Regular Expression - `Where("field").RegExp(regexp.MustCompile("ea"))`
* Matches Function - `Where("field").MatchFunc(func(ra *RecordAccess) (bool, error))`
* Skip - `Where("field").Eq(value).Skip(10)`
* Limit - `Where("field").Eq(value).Limit(10)`
* SortBy - `Where("field").Eq(value).SortBy("field1", "field2")`
* Reverse - `Where("field").Eq(value).SortBy("field").Reverse()`
* Index - `Where("field").Eq(value).Index("indexName")`


If you want to run a query's criteria against the Key value, you can use the `badgerhold.Key` constant:
```Go

store.Find(&result, badgerhold.Where(badgerhold.Key).Ne(value))

```

You can access nested structure fields in queries like this:

```Go
type Repo struct {
  Name string
  Contact ContactPerson
}

type ContactPerson struct {
  Name string
}

store.Find(&repo, badgerhold.Where("Contact.Name").Eq("some-name")
```

Instead of passing in a specific value to compare against in a query, you can compare against another field in the same
struct.  Consider the following struct:

```Go
type Person struct {
	Name string
	Birth time.Time
	Death time.Time
}

```

If you wanted to find any invalid records where a Person's death was before their birth, you could do the following:

```Go

store.Find(&result, badgerhold.Where("Death").Lt(badgerhold.Field("Birth")))

```

Queries can be used in more than just selecting data.  You can delete or update data that matches a query.

Using the example above, if you wanted to remove all of the invalid records where Death < Birth:

```Go

// you must pass in a sample type, so BadgerHold knows which bucket to use and what indexes to update
store.DeleteMatching(&Person{}, badgerhold.Where("Death").Lt(badgerhold.Field("Birth")))

```

Or if you wanted to update all the invalid records to flip/flop the Birth and Death dates:
```Go

store.UpdateMatching(&Person{}, badgerhold.Where("Death").Lt(badgerhold.Field("Birth")), func(record interface{}) error {
	update, ok := record.(*Person) // record will always be a pointer
	if !ok {
		return fmt.Errorf("Record isn't the correct type!  Wanted Person, got %T", record)
	}

	update.Birth, update.Death = update.Death, update.Birth

	return nil
})
```

### Keys in Structs

A common scenario is to store the badgerhold Key in the same struct that is stored in the badgerDB value.  You can
automatically populate a record's Key in a struct by using the `badgerhold:"key"` struct tag when running `Find` queries.

Another common scenario is to insert data with an auto-incrementing key assigned by the database.
When performing an `Insert`, if the type of the key matches the type of the `badgerhold:"key"` tagged field,
the data is passed in by reference, **and** the field's current value is the zero-value for that type,
then it is set on the data _before_ insertion.

```Go
type Employee struct {
	ID uint64 `badgerhold:"key"`
	FirstName string
	LastName string
	Division string
	Hired time.Time
}

// old struct tag, currenty still supported but may be deprecated in the future
type Employee struct {
	ID uint64 `badgerholdKey`
	FirstName string
	LastName string
	Division string
	Hired time.Time
}
```
Badgerhold assumes only one of such struct tags exists. If a value already exists in the key field, it will be overwritten.

If you want to insert an auto-incrementing Key you can pass the `badgerhold.NextSequence()` func as the Key value.

```Go
err := store.Insert(badgerhold.NextSequence(), data)
```

The key value will be a `uint64`.

If you want to know the value of the auto-incrementing Key that was generated using `badgerhold.NextSequence()`,
then make sure to pass your data by value and that the `badgerholdKey` tagged field is of type `uint64`.

```Go
err := store.Insert(badgerhold.NextSequence(), &data)
```


### Unique Constraints

You can create a unique constraint on a given field by using the `badgerhold:"unique"` struct tag:

```Go
type User struct {
  Name string 
  Email string `badgerhold:"unique"` // this field will be indexed with a unique constraint
}
```

The example above will only allow one record of type `User` to exist with a given `Email` field.  Any insert, update
or upsert that would violate that constraint will fail and return the `badgerhold.ErrUniqueExists` error.


### Aggregate Queries

Aggregate queries are queries that group results by a field.  For example, lets say you had a collection of employees:

```Go
type Employee struct {
	FirstName string
	LastName string
	Division string
	Hired time.Time
}
```

And you wanted to find the most senior (first hired) employee in each division:

```Go

result, err := store.FindAggregate(&Employee{}, nil, "Division") //nil query matches against all records
```

This will return a slice of `Aggregate Result` from which you can extract your groups and find Min, Max, Avg, Count,
etc.

```Go
for i := range result {
	var division string
	employee := &Employee{}

	result[i].Group(&division)
	result[i].Min("Hired", employee)

	fmt.Printf("The most senior employee in the %s division is %s.\n",
		division, employee.FirstName + " " + employee.LastName)
}
```

Aggregate queries become especially powerful when combined with the sub-querying capability of `MatchFunc`.


Many more examples of queries can be found in the [find_test.go](https://github.com/timshannon/badgerhold/blob/master/find_test.go)
file in this repository.

## Comparing

Just like with Go, types must be the same in order to be compared with each other.  You cannot compare an int to a int32.
The built-in Go comparable types (ints, floats, strings, etc) will work as expected.  Other types from the standard library
can also be compared such as `time.Time`, `big.Rat`, `big.Int`, and `big.Float`.  If there are other standard library
types that I missed, let me know.

You can compare any custom type either by using the `MatchFunc` criteria, or by satisfying the `Comparer` interface with
your type by adding the Compare method: `Compare(other interface{}) (int, error)`.

If a type doesn't have a predefined comparer, and doesn't satisfy the Comparer interface, then the types value is converted
to a string and compared lexicographically.

## Behavior Changes
Since BadgerHold is a higher level interface than Badger DB, there are some added helpers.  Instead of *Put*, you
have the options of:
* *Insert* - Fails if key already exists.
* *Update* - Fails if key doesn't exist `ErrNotFound`.
* *Upsert* - If key doesn't exist, it inserts the data, otherwise it updates the existing record.

When getting data instead of returning `nil` if a value doesn't exist, BadgerHold returns `badgerhold.ErrNotFound`, and
similarly when deleting data, instead of silently continuing if a value isn't found to delete, BadgerHold returns
`badgerhold.ErrNotFound`.  The exception to this is when using query based functions such as `Find` (returns an empty slice),
`DeleteMatching` and `UpdateMatching` where no error is returned.


## When should I use BadgerHold?
BadgerHold will be useful in the same scenarios where BadgerDB is useful, with the added benefit of being able to retire
some of your data filtering code and possibly improved performance.

You can also use it instead of SQLite for many scenarios.  BadgerHold's main benefit over SQLite is its simplicity when
working with Go Types.  There is no need for an ORM layer to translate records to types, simply put types in, and get
types out.  You also don't have to deal with database initialization.  Usually with SQLite you'll need several scripts
to create the database, create the tables you expect, and create any indexes.  With BadgerHold you simply open a new file
and put any type of data you want in it.

```Go
store, err := badgerhold.Open(filename, 0666, nil)
if err != nil {
	//handle error
}
err = store.Insert("key", &Item{
	Name:    "Test Name",
	Created: time.Now(),
})

```

That's it!

Badgerhold currently has over 80% coverage in unit tests, and it's backed by BadgerDB which is a very solid and well built
piece of software, so I encourage you to give it a try.

If you end up using BadgerHold, I'd love to hear about it.
