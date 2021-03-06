# Migration

## 2.3.1 => 2.3.2

RocksDB is no longer installed as the default database for NodeJS.  If you've been using NanoSQL in NodeJS with the default `PERM` option you'll need to follow the steps below to restore access to your database.

### Step 1
Install the new RocksDB adapter from NPM.
```
npm i @nano-sql/adapter-rocksdb --save
```

### Step 2
Replace `PERM` in your create database call with `new RocksDB()`.

```ts
import { RocksDB } from "@nano-sql/adapter-rocksdb";

// what you had before...
nSQL().createDatabase({
    id: "my_db",
    mode: "PERM",
    tables: [..]
}).then...

// what you should have now
nSQL().createDatabase({
    id: "my_db",
    mode: new RocksDB(),
    tables: [..]
}).then...
```

That's it!  Everything should be working exactly as it was before the update.


## 2.2.5 => 2.26

The `connect()` method is now `createDatabase()`.

```ts
// old syntax
nSQL().connect({
    id: "my_db",
    mode: "PERM",
    tables: [..]
}).then...

// new syntax
nSQL().createDatabase({
    id: "my_db",
    mode: "PERM",
    tables: [..]
}).then...
```

The old `connect()` method will continue to work to prevent breaking changes, but the new method is now considered best practice.

## 2.2.4 => 2.2.5

### Preset Query API

The old preset query objects would look like this:
```ts
name: "getById",
args: {
    "id:uuid":{}
},
call: (db, args, onRow, complete, error) => {
    db.query("select").where(["id", "=", args.id]).stream(onRow, complete, error);
}
```

The `call` property function has changed.  the `onRow`, `complete`, and `error` callbacks have been removed and the `call` function now must return a nanoSQL query object.

The needed change is pretty simple, updating to conform to the new API looks like this:

```ts
name: "getById",
args: {
    "id:uuid":{}
},
call: (db, args) => {
    return db.query("select").where(["id", "=", args.id]).emit();
}
```

When calling `presetQuery` the usage is a little different as well.

```ts
// <= 2.2.4 syntax
nSQL("table").presetQuery("getById").promise({id: "123"}).then...
// or
nSQL("table").presetQuery("getById").stream({id: "123"}, onRow, onComplete, onError);

// >= 2.2.5 syntax
nSQL("table").presetQuery("getById", {id: "123"}).exec().then..
// or
nSQL("table").presetQuery("getById", {id: "123"}).stream(onRow, onComplete, onError)
// or
nSQL("table").presetQuery("getById", {id: "123"}).toCSV().then..
// ... or any other supported query result.
```

The new api allows you to take advantage of things like `.cache()` and any other future additions to the normal query exports.


## 1.X => 2.0 Migration

### Transfer

No 1.X databases will be compatible with 2.0.  You'll need to export your data from your 1.X tables and use `loadJS` to import them into 2.0 databases.

### Config

The `.confg(), .model(), .actions(), .views()` calls have all been merged into a single config object on the `.connect()` call. So if you had a previous setup section that looked like this:

```typescript
nSQL("table1").model(....);
nSQL("table1").actions(...);
nSQL("table1").views(...);
nSQL("table2").model(....);
nSQL().config({...}).connect().then...;
```

Version 2 setup will look like this:

```typescript
nSQL().connect({
    ...config items
    tables: [...tables config including actions and views]
}).then...
```

Let's look at a simple example: 

```typescript
// Version 1.X config
nSQL("posts").model([
    {key: "id" type: "int", props: ["pk", "ai"]},
    {key: "title", type: "string", props: ["idx"]},
    {key: "body", type: "string", default: ""}
])    
nSQL("posts").views([
    {
        name: 'get_post_by_title',
        args: ['title:string'],
        call: function(opts, db) {
            return db.query('select').where(['title', '=', opts.title]).exec();
        }
    }
]);
nSQL().config({
    id: "my_db",
    mode: "PERM"
}).connect().then...

// Identical Version 2 Config 
nSQL().connect({
    id: "my_db",
    mode: "PERM",
    tables: [
        {
            name: "posts",
            model: {
                "id:int": {pk: true, ai: true},
                "title:string": {},
                "body:string": {default: ""}
            },
            indexes: {
                "title:string": {}
            },
            views: [
                {
                    name: 'get_post_by_title',
                    args: ['title:string'],
                    call: function(opts, db) {
                        return db.query('select').where(['title', '=', opts.title]).exec();
                    }
                }
            ]
        }
    ]
}).then..
```

Alternatively, you can use a small helper function that will convert your data models, views, actions and a few other things from the v1 syntax into the v2 syntax.

```typescript
import { nSQLv1Config } from "@nano-sql/core/lib";
// or for <script> installs:
// const nSQLv1Config = window["@nano-sql"].core.nSQLv1Config;

// using the v1 config tool to migrate data models into v2
nSQL().connect(nSQLv1Config((nSQLv1) => {
    // just copy and paste v1 configs into this function
    // replace nSQL calls with nSQLv1
    nSQLv1("table1").model(....);
    nSQLv1("table1").actions(...);
    nSQLv1("table1").views(...);
    nSQLv1("table2").model(....);
    nSQLv1().config({
        id: "my_db",
        mode: "PERM"
    })
})).then...
```

The `nSQLv1Config` function only supports these v1 methods: `model`, `actions`, `views`, `table`, `config` and `rowFilter`.   If you use the v1 helper function some v2 features won't be available to you, like nested indexes.

Other v2 syntax adjustments include: indexes no longer work with props on the column definitions, they should be defined in the new `indexes` section.  The format in the `indexes` section is `"Index Column:Index Type": {props}`.  For example, if you wanted to index a nested value you could create an index like this: `"nested.column.value:string": {}` Supported index types are `int`, `float`and `string`; including any types supported as an array.  `geo` type can also be indexed but cannot be an array type.

### OrderBy & GroupBy

The syntax for OrderBy & GroupBy has changed.  The v1 syntax will still work, but it's recommended you switch to the newer syntax as it supports ordering by functions and correctly maintains the sequence of the columns to order by.

```typescript
// 1.X syntax
nSQL("users").query("select").orderBy({name: "asc"}).exec()

// 2.X syntax
nSQL("users").query("select").orderBy(["name ASC"]).exec()
```

### Connect Event

If you used the old connect event, you should replace it with ready.

```typescript
// 1.X Syntax
nSQL().on("connect", () => { ... });

// 2.X Syntax
nSQL().on("ready", () => { ... });
```

The connect callback still works, but it's called when the database backend is connected, not when the database tables are configured and ready to use.

### LoadJS & LoadCSV

These now use the normal table selector to perform the import.

```typescript
// 1.X syntax
nSQL().loadJS("tableToImport", [rows...]).then...

// 2.X syntax
nSQL("tableToImport").loadJS([rows...]).then..
```

### Drop

In nanoSQL 1.X if you used a drop query the table would just be emptied of rows but you could still use it.  Now the table is actually dropped from the database, meaning it isn't usable anymore unless you call a `create table` query to make it again.  To keep the same behavior, change your drop queries to delete queries.

```typescript
// 1.X syntax
nSQL("table").query("drop").exec()..

// 2.X syntax
nSQL("table").query("delete").exec()..
```

### Join

The syntax for the join command has changed slightly.  `table` is now `with` and `where` is now `on`.

```typescript
// 1.X Syntax
nSQL("users").query("select")
.join({
   type: "left",
   table: "orders",
   where: ["users.id","=","orders.userID"]
})
.exec().then..

// 2.X Syntax
nSQL("users").query("select")
.join({
   type: "left",
   with: {table: "orders"},
   on: ["users.id","=","orders.userID"]
})
.exec().then..
```

### Events

In nanoSQL v1.X, if you listened for events you would get one event per query.  Now, you will get one event per row.  

For example, let's say you have a `change` listener and an `upsert` query happens that adjusts 200 rows.  In v1.X you would get one event for that query, with all 200 rows in the event.  In 2.X you will get 200 events, one for each row.  Every event will contain the originating `query` object and every query has a unique uuid, so you can still reference the same changes to one query if you need to.

### ORM System

The old ORM system is no longer supported.  You'll need to reimplement the desired behavior using [foreign keys](/query/create-table.html#foreign-keys) and [graph queries](/query/select.html#graph).

### Observable Queries

The api for observable queries has changed dramatically, you can read about the new syntax [here](/query/select.html#observer-queries).

### HAVE
The `HAVE` where condition is now `INCLUDES`.  Change all instances of `HAVE` in your code with `INCLUDES`.

### Other Changes

* Plugins built for 1.X won't work with 2.X.  Read the new [plugin docs](/plugins/plugin-api.html).
* Query Functions for 1.X won't work with 2.X.  Read the [query function docs](/query/query-function-api.html).
* Adapters for 1.X won't work with 2.X.  Read the [adapter docs](/adapters/custom-adapter-api.html).

## 1.X - 2.0 Depreciated Features

### Trie Indexes

All `string` type secondary indexes support trie queries using `LIKE` in nanoSQL 2.  You can read about the new syntax [here](/query/create-table.html#indexes).

```ts
// 1.X syntax
nSQL("users").query("select").trieSearch("name", "bi").exec()...

// 2.X syntax
nSQL("users").query("select").where(["name", "LIKE", "bi%"]).exec()...
```

### Fuzzy Search

A new fuzzy search plugin handles this use case, read about it [here](/plugins/search.html).

You'll have to install and setup the plugin, once that's done changes will look like this:

```ts
// 1.X syntax
nSQL("posts").query("select").where(["search(title, body)", ">0.4", "some search term"]).exec();

/// 2.X syntax
nSQL("posts").query("select").where(["SEARCH(*, 'some search term')", "<=", 4]).exec();
```

### History

History isn't yet supported in version 2.0.  A history plugin for version 2 is planned to replace these features.

### Range Queries

Using the `.range(offset, limit)` is no longer supported, standard `.offset(x).limit(x)` queries will perform exactly the same as range queries provided modifiers like GroupBy and OrderBy aren't used.  You can use the new nested queries feature to search/query within the results of a query.

