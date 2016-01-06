# xdb

For now, the readme contains the specification for a "wildcard" DB called `xdb`.

The goal of `xdb` is to offer a single API for any database in Python.

Once a module has been written for a particular database, it is as easy as changing the 1) [import statement](#importing) and 2) [server information](#server-information) to switch between backends. Let's say 2-5 lines of code, total.

**Disclaimer: This is my first attempt at writing a specification before actually writing code, so please bear with me. **

Collaboration: Note that I very much appreciate people collaborating on this specification (and of course, the actual writing of code) to make it better.

## Has this been done before?

Well, I know that I have been writing some kind of translators for my [sky](https://github.com/kootenpv/sky) project, see [here for an example](https://github.com/kootenpv/sky/blob/master/sky/crawler_services.py)

Also, [ZODB](http://www.zodb.org/en/latest/) provides support for different mechanisms such as file, MySQL and Postgres.

## Main motivation

- No need to know the intricacies of each database, learn 1 API, use any database
- Quick to mock using file as backend, then just switch to using a database.
- Helpful for frameworks that want to support multiple databases
- Could potentially really help with migrations between databases

## Difficulties

- Writing a single API that will be "good enough" to encapsulate ALL ("pragmatically" all) other APIs
- async (will have low first priority)

## Main objects

Mostly pseudo code here.

An `Item` class is named `Item` because it could be a `Document` (Document based type) or a `Record` (SQL type). `Item` seems neutral.

```python
class Item():
    """ Not sure yet what functionality """
    pass
```

Database is an object that allows typical CRUD.

```python
class Database(object):
    def __init__(self, server_information):
        pass

    def create_or_update(item):
        return "if not existing, create, otherwise update, and return status code"

    def create_or_update_bulk(items):
        return status_codes_for_each_item

    def get_item_bulk(conditions):
        return lots_of_items

    def get_item(conditions):
        """ Get item only under specific conditions
        if item is newest_item or item.id == X or etc.
        """
        pass

    def delete_item_if_exists(conditions):
        pass
```

## Modules:

There would be one base to inherit from, and then anyone could write a compatible binding, stored in a module. I'm thinking Cloudant, ElasticSearch, File based, ZODB and many more.

Core

```python
from xdb.base import Item
from xdb.base import Database
```

Cloudant

```python
from xdb.cloudant import XItem
from xdb.cloudant import XDatabase
```

File based

```python
from xdb.file import XItem
from xdb.file import XDatabase
```

Here the prepended X to the object shows that the prefix is replaceable; just like the wildcard "`*`". You could read it as `CloudantDatabase`, `FileDatabase` etc.

## Importing

End goal would be to just change a single import wherever database stuff is used.
Plus, the "`server_information.py`", which would contain information required for the specific database (see section below).

Import:

```python
import xdb.file as db
import xdb.cloudant as db
```

So you can use in your file :

```python
db.XDatabase
db.XItem
```

and still do not have to worry about the underlying APIs.

# Server information

In your project, you'd define a `server_information.py`, depending on what backend you use. It contains the information neccessary to make a connection to an XDatabase object

```python
def provide_server_information():
    """ For file based it would look like this """
    # Database is on toplevel path (here /local/path/db1/)
    # Table/ItemType is on sublevel path (here /local/path/db1/table1)
    # Items are files (here /local/path/db1/table1/record1.xdb)
    return {'path': '/local/path/'}

def provide_server_information():
    """ For cloudant it would look like this """
    from cloudant import account

    # or use token instead of username/password
    account = cloudant.Account(USERNAME)
    account.login(USERNAME, PASSWORD)

    return account

def provide_server_information():
    """ For elasticsearch it would look like this """
    import elasticsearch

    info = [{'host': 'localhost', 'port': 9200}]

    return elasticsearch.Elasticsearch(info)
```

## How you'd use it

Considering you'd have a script where you get docs, update, filter, sort, juggle with them, all you have to do is change the toplevel import and change provide_server_information with one of the others:

*/path/to/testing_mydb.py*

```python
import xdb.cloudant as db
from server_information import provide_server_information

mydb = db.XDatabase(provide_server_information())
items = mydb.get_bulk_items()
```

*/path/to/server_information.py*

```python
from cloudant import account

def provide_server_information():
    account = cloudant.Account(USERNAME)
    account.login(USERNAME, PASSWORD)
    return account
```

## Getting a superset of all functionality

This section tries to find out the best superset of functionality such that it can be used for all databases

|API            |Database|Topic|Item|Extras|
|---|---|---|---|---|
|File            |Path-0    |Path-1|Path-1-File||
|ElasticSearch   |Database    |doc_type|Document|Searching/plugins|
|Cloudant        |Database    ||Document|Views, async|
|ZODB            |Database|keys|objects|Transactional|
|SQL|Database|Table|Record|
