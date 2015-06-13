---
layout: post
title: "Scanners and Valuers with Go"
date: 2015-06-12 20:00:00
categories: golang database
---

Data structure serialization in databases can be difficult with golang types.

For example, say we want to have a custom type as an attribute in a type.  How 
can we get this typed attribute inserted and read from a sql database.  Golang 
offers a mechanism to allow for this: Implement Scanner and Valuer database/sql 
interfaces.

Imagine we have the simplest representation of customers on the face of the 
planet, an identifier and a yes/no flag for active:

{% highlight go %}

type YesNoEnum bool

const (
	Yes YesNoEnum = true
	No            = false
)

type Customer struct {
	CustomerID int64
	Active     YesNoEnum
}

{% endhighlight %}

Now we have a dedicated YesNoEnum of which we have two constants, Yes and No. 
Lets try to start storing some Customers in a database:


{% highlight go %}
package main

import (
	"database/sql"
	"fmt"
	_ "github.com/mattn/go-sqlite3"
)

func main() {
	// open a database
	db, err := sql.Open("sqlite3", "./my.db")
	if err != nil {
		// fail!
		fmt.Println(err.Error())
		fmt.Println("ouch, database open failed")
		return
	}
	// close the database at the end of the function
	defer db.Close()

	//new Customer
	c := &Customer{
		CustomerID: 1,
		Active:     Yes,
	}
	// insert customer!
	_, err = db.Exec(
		"insert into customer (id, active) values (?, ?);",
		c.CustomerID, c.Active,
	)
	if err != nil {
		// fail!
		fmt.Println(err.Error())
		fmt.Println("I am sorry, your insert failed...better luck next time!")
		return
	}
}

{% endhighlight %}

Sweet!  Finally, we get to add some customers to our database, and set the to 
active!  Lets run it!

{% highlight sh %}
    go run main.go

sql: converting Exec argument #1's type: unsupported type main.YesNoEnum, a bool
I am sorry, your insert failed...better luck next time!

{% endhighlight %}

Okay, what is going on here.  Go's database/sql Exec is choking on our YesNoEnum custom
type.  But it is just a bool right?  No, by defining it as a Type, it is no longer a bool, it is a YesNoEnum type, and database/sql doesn't know what to do with it.  We can explain how database/sql should handle this type by implementing sql.Valuer.

Valuer is an interface where we can turn our type into a simpler type, a type the database will be able to understand, such as a boolean.  Here is what it looks like:

{% highlight go %}

import (
	"database/sql"
	"database/sql/driver"
	"fmt"
	_ "github.com/mattn/go-sqlite3"
)

type YesNoEnum bool

// Value - Implementation of valuer for database/sql
func (yne YesNoEnum) Value() (driver.Value, error) {
    // value needs to be a base driver.Value type
    // such as bool.
	return bool(yne), nil
}

const (
	Yes YesNoEnum = true
	No            = false
)
    
{% endhighlight %}

Now we when we run our application we should get a little closer:

{% highlight sh %}
    go run main.go

no such table: customer
I am sorry, your insert failed...better luck next time!
{% endhighlight %}

Muahahaha! Getting closer.  If only we had created that customer table in the database. Alright, after creating the table and re-running we have a customer id in the database.  Lets try to get it out of the database, and print the stored value to stdout.

{% highlight go %}

	// get the customer record
	rows, err := db.Query("select id, active from customer")
	if err != nil {
		fmt.Println(err.Error())
		fmt.Println("I am sorry, your query failed...better luck next time!")
		return
	}
	// close the rows at the end of the function
	defer rows.Close()
	for rows.Next() {
		foundCustomer := new(Customer)
		if err := rows.Scan(
			&foundCustomer.CustomerID, &foundCustomer.Active,
		); err != nil {
			fmt.Println(err.Error())
			fmt.Println("I am sorry, your scanning failed...better luck next time!")
			return
		}
		// time to print our customers!!
		fmt.Println(foundCustomer)
	}
	if err := rows.Err(); err != nil {
		fmt.Println(err.Error())
		fmt.Println("I am sorry, your rows failed...better luck next time!")
		return
	}
{% endhighlight %}

The above queries our customer table for all customers, then for each customer the program makes a new customer (foundCustomer) and tries to put the data from the database into our customer data structure.  When we run our program again we unsurprisingly get the following:

{% highlight sh %}
go run main.go

sql: Scan error on column index 1: unsupported driver -> Scan pair: int64 -> *main.YesNoEnum
I am sorry, your scanning failed...better luck next time!
{% endhighlight %}

Shucks.  I guess we need some glue to make database/sql understand how to put the value from the database into our data structure.  That glue is the scanner interface.  Much like the Valuer interface, we need to perform some conversions to get the data from the database into our structures.  Below is our implementation of a valuer for our YesNoEnum type:

{% highlight go %}

// Scan - Implement the database/sql scanner interface
func (yne *YesNoEnum) Scan(value interface{}) error {
	// if value is nil, false
	if value == nil {
		// set the value of the pointer yne to YesNoEnum(false)
		*yne = YesNoEnum(false)
		return nil
	}
	if bv, err := driver.Bool.ConvertValue(value); err == nil {
		// if this is a bool type
		if v, ok := bv.(bool); ok {
			// set the value of the pointer yne to YesNoEnum(v)
			*yne = YesNoEnum(v)
			return nil
		}
	}
	// otherwise, return an error
	return errors.New("failed to scan YesNoEnum")
}

{% endhighlight %}

So, moment of truth... do we get a customer record printed out?

Full gist [here][scanner-valuer-gist]

Hope this was helpful to anyone.

[scanner-valuer-gist]: https://gist.github.com/husobee/cac9cddbaacc1d3a7ae1
