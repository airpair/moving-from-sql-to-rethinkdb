I have been a software engineer for coming up to 10 years now. Throughout academia and my professional career the vast majority of my interactions with data storage layers have been with SQL systems - primarily MySQL and PostgreSQL.

I have built up a strong understanding of SQL and its capabilties. I am a big fan of it and its declarative nature. However, I have often come up against scenarios where SQL and RDBMS have struggled and I understand that SQL is not right for everything. The relatively recent rise and proliferation of NoSQL databases have given developers an increasingly large amount of choice on what to use for their persistance layer.

When we began working on [Workshape.io](http://www.workshape.io) we needed to choose a data storage layer that would allow us to rapidly iterate and evolve our data structure over time. We needed something that let us focus on our application and prototyping over data modelling. I saw RethinkDB as a great choice, moving away from my traditional routes of working with SQL. What begun as something of an experimental choice, and exploration of working with NoSQL, quickly established itself as a very impressive option that formed the basis of our application. 

This post is about using [RethinkDB](http://www.rethinkdb.com/), a NoSQL database, as an engineer coming from an SQL background. It is a comprehensive comparison of the main tasks you may find yourself performing when using either in a web development environment.

## Introduction to RethinkDB

[RethinkDB](http://www.rethinkdb.com/) is an open-source distributed database that stores JSON documents. It is a NoSQL solution to data storage.

It is both developer and operations oriented - combining an easy-to-use powerful query language with simple controls for operating at scale with high availability. Drawing upon learnings from NoSQL databases like MongoDb, CouchDB, Riak and Cassandra, RethinkDB is focused on bringing the developer the best of both worlds as a foundation for their applications. 

Founded in 2009 RethinkDB came out of [YCombinator](https://www.ycombinator.com/) and is relatively new player on the data storage scene. The database itself has been available for a little over a year now and it is still undergoing heavy development. Their [stability page](http://www.rethinkdb.com/stability/) outlines their current status on most common usage patterns of RethinkDB and their stability.

RethinkDB have a great video that introduces the main features [here](http://www.rethinkdb.com/videos/what-is-rethinkdb/).

## Choosing RethinkDB over SQL

**SQL and NoSQL are not silver bullets**. It is important to understand that each approach, and their specific implementations, have use cases that they lend themselves to. 

RethinkDB is a very good solution when flexibility and rapid iteration are primary factors. It's other big strength is its ability to scale horizontally, with very little effort or changes required to how you interact with the database.

It is not a good solution if you need a fixed schema, or require transaction support. In this case you should still look to SQL for these use cases.

It is also not a good solution if you intend to use it for intense data analysis. This use case would be better suited to another NoSQL system like Hadoop.

## Comparing Usage

For the purpose of this blog post I shall compare the various activities involved in setting up, using, deploying and scaling a data storage layer.

I shall compare the set up of PostgreSQL and MySQL to RethinkDB on Ubuntu 14.04. Where appropriate I shall speak of SQL generically, ignoring the subtleties of each RBDMS and it's implementation of SQL.

I am using the following `Vagrantfile` for setting up an environment inside VirtualBox:

```bash,linenums=true
# Basic Vagrantfile for quickly spinning up 2 VMs
Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu/trusty64"
  config.vm.box_check_update = true

  config.vm.define "s1" do |s1|
    s1.vm.box = "ubuntu/trusty64"
    s1.vm.hostname = "thorium"

    s1.vm.provision "shell", path: "provision.sh"

    s1.vm.network "forwarded_port", guest: 8080, host: 8083
    s1.vm.network "private_network", ip: "192.168.33.11"

    s1.vm.synced_folder "./data", "/vagrant_data", type: "nfs"

    s1.vm.provider "virtualbox" do |v|
      v.memory = 1024
      v.cpus = 2
    end

    s1.vm.post_up_message = " [x] Development Environment 1 is FIRED UP!!"

    s1.ssh.forward_agent = true
  end

  config.vm.define "s2" do |s2|
    s2.vm.box = "ubuntu/trusty64"
    s2.vm.hostname = "thallium"

    s2.vm.provision "shell", path: "provision.sh"

    s2.vm.network "forwarded_port", guest: 8080, host: 8084
    s2.vm.network "private_network", ip: "192.168.33.12"

    s2.vm.synced_folder "./data", "/vagrant_data", type: "nfs"

    s2.vm.provider "virtualbox" do |v|
      v.memory = 1024
      v.cpus = 2
    end

    s2.vm.post_up_message = " [x] Development Environment 2 is FIRED UP!!"

    s2.ssh.forward_agent = true
  end

end
```

This creates two virtual machines with Ubuntu and installs the dependencies required onto each box.

### Installation

The installation of MySQL and PostgreSQL servers on Ubuntu is a one-line operation as Ubuntu includes both in its source list by default.

```bash,linenums=true
# PostgresSQL
sudo apt-get install postgresql
```

```bash,linenums=true
# MySQL
sudo apt-get install mysql-server
```

RethinkDb requires you add a source list, run `apt-get update` and then use `apt-get` to install RethinkDB. Their website provides this in one command.

```bash,linenums=true
# RethinkDB
source /etc/lsb-release && echo "deb http://download.rethinkdb.com/apt $DISTRIB_CODENAME main" | sudo tee /etc/apt/sources.list.d/rethinkdb.list
wget -qO- http://download.rethinkdb.com/apt/pubkey.gpg | sudo apt-key add -
sudo apt-get update
sudo apt-get install rethinkdb
```

Both MySQL and PostgreSQL daemons run straight after installation, however RethinkDB does not. To get RethinkDB up and running with init.d you need to do this:

```bash,linenums=true
# RethinkDB
sudo cp /etc/rethinkdb/default.conf.sample /etc/rethinkdb/instances.d/instance1.conf
sudo /etc/init.d/rethinkdb restart
```
*You may want to modify the settings in instance1.conf to suit you*

Although there are more steps to getting RethinkDB set up right now in comparison to PostgreSQL and MySQL it is most likely due to the relative age of RethinkDB and its non-presence in the Ubuntu distribution. With that said, the RethinkDB site does a great job of documenting how to install the server on a variety of operating systems so installation is still relatively simple. *Windows is not currently supported as a base OS*

### Management Tools

Out of the box RethinkDB comes with a browser-based GUI that is available on `http://localhost:8080` (in our Vagrantfile we bound localhost:8080 to the VMs port 8080). 

![RethinkDB GUI](http://i.imgur.com/98ChMa7.png)

This tool provides:

 - real-time performance and health status
 - a management tool for each server and database in your cluster
 - a data explorer for running queries

In order to test-drive the RethinkDB you can do it entirely within your browser.

MySQL and PostgreSQL do not come with GUIs and instead ship with command line utilities, `mysql` and `psql` respectively. You can perform database management, querying and query analysis through these command line tools.

If you would like you can also access RethinkDB on the command line through the Python REPL by doing the following:

```python
# RethinkDB
import rethinkdb as r
r.connect('localhost', 28015).repl()

# To get a set of results
list(r.db('dragonball').table('characters').limit(3).run())
```

We at [Workshape.io](http://www.workshape.io) have also built a [command line](https://github.com/Workshape/reql-cli) interface written in ES6 that you can use too. (Contributions to the project are welcome!)

### Creating a database

This is where you begin to notice the difference between SQL and RethinkDB, with RethinkDB having its own query language [ReQL](http://rethinkdb.com/docs/introduction-to-reql/). ReQL has official native bindings in Javascript, Ruby and Python (A number of unofficial bindings are also [available](http://www.rethinkdb.com/docs/install-drivers/)). With this chainable query language you are able to 'talk' to your database in a language you are already comfortable with. One of the biggest benefits to arise from this approach is that **query injection** is now something of the past as you are no longer manipulating strings when querying a database.

When using ReQL inside of the GUI the format conforms to that of the native Javascript API. The GUI comes with a code completion suggestor built in, so it is a great tool to begin when you are starting out.

![Code Completion on RethinkDB Data Explorer GUI](http://i.imgur.com/GgoCBj3.png)

```javascript,linenums=true
// RethinkDB
r.dbCreate('dragonball');
```

```sql,linenums=true
# SQL
CREATE DATABASE dragonball;
```

### Creating a table

The creation of a table in SQL and RethinkDB brings forward the issue of requiring a schema. In the world of RethinkDB you need not concern yourself with confirming the structure of your data prior to creating a table, only knowing that you want to store a certain type of something. 

```javascript,linenums=true
// RethinkDB
r.db('dragonball').tableCreate('characters');
```

In SQL however, data modelling has to be considered as a step prior to creating your tables. This is important to flag up as in agile development, SQL puts up a significant road-block here. There is certain commitment that needs to be made by the developer up-front in order to cement the schema of a database. 

In our example, a character may originate from one or more species in the Dragonball world. In RethinkDB we have not committed, or made any decisions, or how we are going to store this information, whereas with SQL we have made the decision to normalise our data set and use a *link table* to turn a MANY-to-MANY relationship into ONE-to-MANY-to-ONE.

```sql,linenums=true
# SQL
CREATE TABLE characters (
     id int NOT NULL AUTO_INCREMENT,
     name varchar(50) NOT NULL,
     max_strength int NULL,
     created timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
     PRIMARY KEY (id)
);

CREATE TABLE species (
    id int NOT NULL AUTO_INCREMENT,
    name varchar(50) NOT NULL,
    created timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id)
);

CREATE TABLE character_species (
    character_id int NOT NULL,
    species_id int NOT NULL,
    PRIMARY KEY (character_id, species_id)
);

ALTER TABLE character_species 
ADD CONSTRAINT character_fk_idx
FOREIGN KEY (character_id) REFERENCES characters(id) 
ON UPDATE CASCADE
ON DELETE CASCADE;

ALTER TABLE character_species 
ADD CONSTRAINT species_fk_idx
FOREIGN KEY (species_id) REFERENCES species(id) 
ON UPDATE CASCADE
ON DELETE CASCADE;
```

*By default, in RethinkDB a field called `id` is created that acts as an auto-assigned primary key - these are [UUIDs](http://en.wikipedia.org/wiki/Universally_unique_identifier) as opposed to integers used in most RDBMS systems. This is because RethinkDB is built to be highly scalable.*

### Insertion

With insertion you notice that RethinkDB works with, and stores documents as JSON. You can use the insert command to insert one or more records at a time, and each document can have a varying structure and also contain nested documents. Any valid JSON document, is valid in RethinkDB.

When it comes to storing information about each character's species it is as simple as storing an array inside each document if the species is known, and omitting that field if not.

```javascript,linenums=true
// RethinkDB
r
  .db('dragonball')
  .table('characters')
  .insert([
    {name: 'Goku', species: ['Saiyan'], maxStrength: 1000000, created: r.now() },
    {name: 'Gohan', species: ['Human','Saiyan'], maxStrength: 700000, created: r.now() },
    {name: 'Vegata', species: ['Saiyan'], maxStrength: 900000, created: r.now() },
    {name: 'Piccolo', species: ['Nemek'], maxStrength: 800000, created: r.now() },
    {name: 'Android 16', maxStrength: 920000, created: r.now() },
    {name: 'Bulma', species: ['Human'], created: r.now() },
    {name: 'Trunks', species: ['Human','Saiyan'], maxStrength: 650000, created: r.now() }
  ]);
```

In SQL we first create the characters, and then species and then insert the link records.

```sql,linenums=true
# SQL
INSERT INTO characters (name, max_strength)
    VALUES
        ('Goku', 1000000),
        ('Gohan', 700000),
        ('Vegata', 900000),
        ('Piccolo', 800000),
        ('Android 16', 920000),
        ('Bulma', null),
        ('Trunks', 650000);
        
INSERT INTO species (name)
    VALUES
        ('Saiyan'),
        ('Human'),
        ('Nemek');

INSERT INTO character_species (character_id, species_id)
    VALUES
        (1,1),
        (2,1),
        (2,2),
        (3,1),
        (4,3),
        (6,2),
        (7,1),
        (7,2);
```

*In PostgreSQL 9.4 it is possible to store and search upon arrays and so it is not compulsary that the data is modelled in such a conventional manner*

### Basic Querying

When searching for equality on one of more fields there is not too much to differentiate the two approaches. 

```javascript,linenums=true
// RethinkDB
r
    .db('dragonball')
    .table('characters')
    .filter({name: 'Goku'});
```

![RethinkDB Query Results](http://i.imgur.com/RAQc3r4.png)

```sql,linenums=true
# SQL
SELECT * 
FROM characters 
WHERE name = 'Goku';
```

![MySQL Query Results](http://i.imgur.com/zZK2Yqw.png)

When looking for something other than equality you begin to be introduced to other aspects of the RethinkDB Query DSL. You can use functions as filters on your data. The documentation into the [RethinkDB API](http://rethinkdb.com/api/javascript/) provides a great resource for familiarising yourself with the things you can do with it.

```javascript,linenums=true
// RethinkDB
r
    .db('dragonball')
    .table('characters')
    .filter(function(row){
        return row('maxStrength').gt(700000).and(row('species').contains('Saiyan'));
    })
    .orderBy(r.desc('maxStrength'));
```

```sql,linenums=true
# SQL
SELECT c.* 
FROM characters c
INNER JOIN character_species cs ON c.id = cs.character_id
INNER JOIN species s ON cs.species_id = s.id
WHERE max_strength > 700000
AND s.name = 'Saiyan'
ORDER BY max_strength DESC;
```

With ReQL grouping is something that is less fraught with peril. In order to group characters in our database by species we can achieve this very simply by applying a group by species.

```javascript,linenums=true
// RethinkDB
r
    .db('dragonball')
    .table('characters')
    .group('species');
```

To achieve a similar result in SQL requires the use of LEFT JOINs (to ensure Android 16) is included in the result set, plus the use of `group_concat()` to make sure we do not lose information from the grouping.

*`group_concat()` des not exist in PostgreSQL, you'd have to `use array_agg()`*

```sql,linenums=true
# SQL
SELECT c.*, group_concat(s.name) as species 
FROM characters c 
LEFT JOIN character_species cs ON c.id = cs.character_id 
LEFT JOIN species s ON cs.species_id = s.id
GROUP BY c.id
ORDER BY species
```

The grouping could also be achieved in a slightly more equivalent manner to the ReQL query with a slight loss of information around each character. This will also only group characters by individual species not by a compound species. Trunks and Gohan would appear in both the Human and Saiyan groupings.

```sql,linenums=true
# SQL
SELECT s.name, group_concat(c.name) as characters
FROM characters c 
LEFT JOIN character_species cs ON c.id = cs.character_id 
LEFT JOIN species s ON cs.species_id = s.id
GROUP BY s.id
```

### Updating

Updating in RethinkDB is similar in many regards to SQL when working with existing fields. You can update all entries in a table, or a filtered sub-set of entries
```javascript,linenums=true
// RethinkDB
r
    .db('dragonball')
    .table('characters')
    .filter({name:'Goku'})
    .update({maxStrength: 2500000});
```

```sql,linenums=true
# SQL
UPDATE characters 
SET max_strength = 2500000
WHERE name = 'Goku';
```

With SQL you are restricted to updating records with the data type for that field. Setting it to another type will result in an error. In RethinkDB you are not restricted here. You can make wholesale changes to records with the update command - changing data types.

In a similar fashion you can add properties from documents in a table with the update command. In the SQL world you would have to consider a data migration, whereas here you can immediately apply a change to a collection.

```javascript,linenums=true
// RethinkDB
r
    .db('dragonball')
    .table('characters')
    .filter({name:'Goku'})
    .update({age: 50});
    
```

```sql,linenums=true
# SQL
ALTER TABLE characters 
ADD COLUMN age int NULL;

UPDATE characters
SET age = 50
WHERE name = 'Goku';
```

If you would like to remove properties from a RethinkDB table you need to use to the `replace()` function. Combining this with `row.without(...)` enables you to replace each row in the table without a field, or fields.

```javascript,linenums=true
// RethinkDB
r
    .db('dragonball')
    .table('characters')
    .replace(r.row.without('maxStrength'));
```

In SQL to drop a column may seem slightly more intuitive by altering the table structure.

```sql,linenums=true
# SQL
ALTER TABLE characters 
DROP COLUMN max_strength;
```

### Deleting

Deleting any records from a table is, unsurprisingly, not any different in terms of functionality. You can either empty a whole table or delete the resulting set from a filter.

```javascript,linenums=true
// RethinkDB
r
    .db('dragonball')
    .table('characters')
    .filter({name: 'Goku')
    .delete()
```

```sql,linenums=true
# SQL
DELETE FROM characters
WHERE name = 'Goku';
```

### Indexes

RethinkDB supports primary and secondary indexes. Primary indexes enforce uniqueness but secondary indexes do not. [Secondary indices](http://rethinkdb.com/api/python/index_create/) can be a single, compound, multi or geospatial index. 

Like indexes in SQL databases adding them improves the speed of many read queries at the slight cost of increased storage space and decreased write performance.

In MySQL and PostgreSQL there are slightly different options for indexes. In both of these, you are able to create single, compound, and geospatial indexes (with an extension in MySQL). You can also choose which underlying indexing algorithm to use.

You are also able to enforce uniqueness on any field in SQL database. This is something you have to sacrifice with RethinkDB to allow for it's distributed qualities.

### Data Modelling

With RDBMS systems data modelling requires the use of foreign keys and related data. By using this technique you *usually* aim to remove any redundant data from your model - this helps ensure ensure consistency and integrity in your database. The process of modelling a suitable schema usually involves normalisation moving from the first normal form through to the fifth, or sixth. In fast moving environments this up-front effort *could* be seen as a hinderance, and this is where RethinkDB and it's NoSQL counter parts lend itself to agile and iterative development. You can favour using **nested arrays** over composing a suitable schema that support your needs. This is the typical approach most document based databases provide.

However, RethinkDB also offers the ability to link data across multiple tables with the use of distributed joins. This is somewhat similar to joins in SQL. You are able to model relations between two seperate tables. This addition to RethinkDB gives you a choice which is covered in their [data modelling](http://rethinkdb.com/docs/data-modeling/) section. In summary:

 - nested arrays offer a convenient way to store a small amount (~ 200 entries) of associated data that is only usually queried in conjunction with its parent
 - joined data is better suited to larger amounts of data where that data maybe be accessed independent of its associate.


*There is an exception with PostgreSQL 9.4 where you can store [JSON inside a table](http://clarkdave.net/2013/06/what-can-you-do-with-postgresql-and-json/). This offers similar benefits to RethinkDB in this regard where you can defer modelling decisions and store/query json data.*

### Using RethinkDB in your application

So far we have look at how RethinkDB differs from working with SQL and RDBMS databases but we have not covered what it is like using each of the options inside your application code. In this example I will be using Node.js to demonstrate the various differences between the two.

Below are very basic examples of using rethinkdb, mysql and postgres in Node.js to print the results of the group query.

```javascript,linenums=true
// RethinkDB
var r = require('rethinkdb');
r
.connect( {host: 'localhost', port: 28015, db: 'dragonball'})
.then(function(conn){
  r
  .table('characters')
  .group('species')
  .run(conn)
  .then(function(cursor) {
      return cursor.toArray()
      .then(function(result) {
          console.log(JSON.stringify(result, null, 2));
          return conn.close();
      });
  });
})
.error(function(err) {
  console.log('Something went wrong!', err);
});
```

```javascript,linenums=true
// MySQL
var mysql      = require('mysql');
var connection = mysql.createConnection({
  host     : 'localhost',
  user     : 'root',
  database : 'dragonball'
});

connection.connect();

var groupQuery = 'SELECT c.*, group_concat(s.name) as species \
FROM characters c \
LEFT JOIN character_species cs ON c.id = cs.character_id \
LEFT JOIN species s ON cs.species_id = s.id \
GROUP BY c.id \
ORDER BY species';

connection.query(groupQuery, function(err, rows, fields) {
  if (err) throw err;

  console.log(JSON.stringify(rows, null, 2));
});

connection.end();
```

```javascript,linenums=true
// PostgreSQL
var pg = require('pg');
var conString = "postgres://vagrant:awesome@localhost/dragonball";

var groupQuery = 'SELECT c.*, array_agg(s.name) as species \
FROM characters c \
LEFT JOIN character_species cs ON c.id = cs.character_id \
LEFT JOIN species s ON cs.species_id = s.id \
GROUP BY c.id \
ORDER BY species';

pg.connect(conString, function(err, client, done) {
  if(err) {
    return console.error('error fetching client from pool', err);
  }
  client.query(groupQuery, function(err, result) {
    //call `done()` to release the client back to the pool
    done();

    if(err) {
      return console.error('error running query', err);
    }
    console.log(JSON.stringify(result.rows, null, 2));
    //output: 1
    client.end();
  });
});
```

The most noticeable difference, that was mentioned earlier, is that RethinkDB's query language is embedded into Node.js and therefore there is no context switching between querying your database and performing other tasks in your application. From an engineering point of view this could be seen to help with productivity for the individual and the team. You could also argue that your source code is more readable.

In SQL systems all queries are sent as strings to the database server for processing. In both MySQL and PostgreSQL examples the specific npm modules used provide parameterisation support to add against SQL injection, which is something you need not concern yourself with with RethinkDB.

In larger applications and/or when working frameworks it is quite common to work with an ORM to abstract the need to work with SQL to work with your database. When using these in conjunction with MySQL or PostgreSQL you may feel that RethinkDB's API is nothing too special. But in this case the important thing to remember is that various ORMs make assumptions about how you want to work with your data, and depending upon its underlying implementation (Active Record or Data Mapper) it can limit and hinder the performance you get from your RDBMS. The full power of RethinkDB and its query language is available out of the box. You can ORM convenience with it's embedded API without any assumptions limiting which queries will run efficiently.

In the world of RethinkDB you can also use an ORM if you so desire. If you would like to have some enforcement of schema you can work with [Thinky](http://thinky.io), a light weight ORM specifically built for RethinkDB.

### Operating at scale

The ability to scale is considered one of the main benefits to using NoSQL over SQL. RethinkDB provides the ability to almost effortlessly scale to a multi-node cluster just by running a couple of commands. Configuration could not be simpler!

We can create a RethinkDB cluster with two commands. On our first VM run:

```bash,linenums=true
# RethinkDB
rethinkdb --bind all
```

And on our second machine we run:

```bash,linenums=true
# RethinkDB
rethinkdb --join 192.168.33.11:29015 --bind all
```

That is it. We now have a cluster with 2 servers. **Wow**. The GUI will reflect the status of the cluster.

![RethinkDB with 2 servers](http://i.imgur.com/ykuHIbu.png)

It will also inform you when one of the servers has gone away and give you instructions on what to do to resolve the situation. 

![RethinkDB node down](http://i.imgur.com/JX1Xcnm.png)

In the SQL world scaling is much more of a daunting tasks with a [number of strategies](http://www.slideshare.net/ScaleBase/strategies-for-scaling-my-sql) to be considered dependent upon your specific use case and needs. To begin with the common approach is to beef up the box on which you are running your database server, known as vertical scaling. There is also Master-Slave replication and possibility of directing writes to one db server and reads to another.

Whichever strategy you adopt when working with an SQL database it is most definitely a more challenging task to complete than in RethinkDB with many more variables. In RethinkDB it is incredibly simple to scale, and no extra effort to utilise with ReQL automatically parallelising queries across your cluster.

If you are interested in seeing an example check out this [blog](https://www.digitalocean.com/community/tutorials/how-to-set-up-master-slave-replication-in-mysql) by Digital Ocean about setting up MySQL Master-Slave replication. I would include it in this blog post, but the process has a large number of steps in comparison.

### Powering a real-time application

In version 1.16 of RethinkDB the team behind the database launched the [changefeeds API](http://rethinkdb.com/docs/changefeeds/javascript/). This was initially intended to improve integrations with other realtime systems like message queues. Developers were drawn to this feature for another reason - the possibility of powering real-time applications on the web. 

The changefeed API offers a much simpler way to monitor a database for updates and respond accordingly. Previous to this a number of steps would be required to efficiently and accurately update users in real-time of any changes in your application. 

A common approach with SQL systems would be to monitor the replication logs for changes on individual records giving you the capability to push updated documents to clients based on changes. With RethinkDB's changefeeds though you are not only able to push document based changes, but also any changes made on documents contained in a query. Like every other aspect of RethinkDB this functionality does not change with scale, and the developer does not have to do any different. In a SQL environment the complexity of monitoring multiple replication logs would have to be dealt with.

The below example shows us monitoring for changes any Saiyans in our characters table.

```javascript,linenums=true
// RethinkDB
var r = require('rethinkdb');
r
.connect({host: 'localhost', port: 28015, db: 'dragonball'})
.then(function(conn){
  r
  .table('characters')
  .filter(function(row) {
    return row('species').contains('Saiyan');
  })
  .changes()
  .run(conn, function(err, cursor) {
      cursor.each(console.log);
  });
})
.error(function(err) {
  console.log('Something went wrong!', err);
});
```

That's it. Any changes made to existing records, or new ones added that match the filter criteria will logged to the console. The output for a change contains an old and new value, as shown below.

```javascript,linenums=true
{  
   new_val: { 
     created: Mon Mar 30 2015 15:36:46 GMT+0000 (UTC),
     id: '4b9c9b9c-9c82-4e22-8d43-4afe66e2fa2d',
     maxStrength: 800000,
     name: 'Goku',
     species: [ 'Saiyan' ] 
   },
   old_val: { 
     created: Mon Mar 30 2015 15:36:46 GMT+0000 (UTC),
     id: '4b9c9b9c-9c82-4e22-8d43-4afe66e2fa2d',
     maxStrength: 1000000,
     name: 'Goku',
     species: [ 'Saiyan' ] 
   } 
}
```

By using this feature in conjunction with Web Sockets it is refreshingly easy to push changes to a client.

## Conclusions

In this post we have covered the basics of working with RethinkDB in relation to it's SQL counterparts. I have tried to provide a comprehensive comparative usage to allow readers to see the two side by side and gain more of a hands on understanding of RethinkDB and it's relative merits. As we moved from the basic commands to real-world usage the difference in developer experience should be clearer, with RethinkDB building out their product with you, the developer, at the centre of it.

If you are working on a new web or mobile project I would strongly advise the consideration of RethinkDB. It is incredibly developer friendly with very powerful features for building web applications. The overwhelming experience for us has been that RethinkDB has allowed me and my team at Workshape.io to focus more on our product rather than low-level engineering oriented tasks that may detract from our current velocity. It has helped us stay focussed and efficient. 

## Source Code

If you would like to run through the example I have created a [Github repository](https://github.com/GordyD/rethinkdb-vs-sql-demo) with the source code and instructions for set up.

