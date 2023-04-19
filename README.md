Object Relational Tutorial
The SQLAlchemy Object Relational Mapper presents a method of associating user-defined Python classes with database tables, and instances of those classes (objects) with rows in their corresponding tables. It includes a system that transparently synchronizes all changes in state between objects and their related rows, called a unit of work, as well as a system for expressing database queries in terms of the user defined classes and their defined relationships between each other.

The ORM is in contrast to the SQLAlchemy Expression Language, upon which the ORM is constructed. Whereas the SQL Expression Language, introduced in SQL Expression Language Tutorial, presents a system of representing the primitive constructs of the relational database directly without opinion, the ORM presents a high level and abstracted pattern of usage, which itself is an example of applied usage of the Expression Language.

While there is overlap among the usage patterns of the ORM and the Expression Language, the similarities are more superficial than they may at first appear. One approaches the structure and content of data from the perspective of a user-defined domain model which is transparently persisted and refreshed from its underlying storage model. The other approaches it from the perspective of literal schema and SQL expression representations which are explicitly composed into messages consumed individually by the database.

A successful application may be constructed using the Object Relational Mapper exclusively. In advanced situations, an application constructed with the ORM may make occasional usage of the Expression Language directly in certain areas where specific database interactions are required.

The following tutorial is in doctest format, meaning each >>> line represents something you can type at a Python command prompt, and the following text represents the expected return value.

Version Check
A quick check to verify that we are on at least version 1.3 of SQLAlchemy:

>>> import sqlalchemy
>>> sqlalchemy.__version__ 
1.3.0
Connecting
For this tutorial we will use an in-memory-only SQLite database. To connect we use create_engine():

>>> from sqlalchemy import create_engine
>>> engine = create_engine('sqlite:///:memory:', echo=True)
The echo flag is a shortcut to setting up SQLAlchemy logging, which is accomplished via Python’s standard logging module. With it enabled, we’ll see all the generated SQL produced. If you are working through this tutorial and want less output generated, set it to False. This tutorial will format the SQL behind a popup window so it doesn’t get in our way; just click the “SQL” links to see what’s being generated.

The return value of create_engine() is an instance of Engine, and it represents the core interface to the database, adapted through a dialect that handles the details of the database and DBAPI in use. In this case the SQLite dialect will interpret instructions to the Python built-in sqlite3 module.

Lazy Connecting

The Engine, when first returned by create_engine(), has not actually tried to connect to the database yet; that happens only the first time it is asked to perform a task against the database.

The first time a method like Engine.execute() or Engine.connect() is called, the Engine establishes a real DBAPI connection to the database, which is then used to emit the SQL. When using the ORM, we typically don’t use the Engine directly once created; instead, it’s used behind the scenes by the ORM as we’ll see shortly.

See also

Database Urls - includes examples of create_engine() connecting to several kinds of databases with links to more information.

Declare a Mapping
When using the ORM, the configurational process starts by describing the database tables we’ll be dealing with, and then by defining our own classes which will be mapped to those tables. In modern SQLAlchemy, these two tasks are usually performed together, using a system known as Declarative, which allows us to create classes that include directives to describe the actual database table they will be mapped to.

Classes mapped using the Declarative system are defined in terms of a base class which maintains a catalog of classes and tables relative to that base - this is known as the declarative base class. Our application will usually have just one instance of this base in a commonly imported module. We create the base class using the declarative_base() function, as follows:

>>> from sqlalchemy.ext.declarative import declarative_base

>>> Base = declarative_base()
Now that we have a “base”, we can define any number of mapped classes in terms of it. We will start with just a single table called users, which will store records for the end-users using our application. A new class called User will be the class to which we map this table. Within the class, we define details about the table to which we’ll be mapping, primarily the table name, and names and datatypes of columns:

>>> from sqlalchemy import Column, Integer, String
>>> class User(Base):
...     __tablename__ = 'users'
...
...     id = Column(Integer, primary_key=True)
...     name = Column(String)
...     fullname = Column(String)
...     nickname = Column(String)
...
...     def __repr__(self):
...        return "<User(name='%s', fullname='%s', nickname='%s')>" % (
...                             self.name, self.fullname, self.nickname)
Tip

The User class defines a __repr__() method, but note that is optional; we only implement it in this tutorial so that our examples show nicely formatted User objects.

A class using Declarative at a minimum needs a __tablename__ attribute, and at least one Column which is part of a primary key [1]. SQLAlchemy never makes any assumptions by itself about the table to which a class refers, including that it has no built-in conventions for names, datatypes, or constraints. But this doesn’t mean boilerplate is required; instead, you’re encouraged to create your own automated conventions using helper functions and mixin classes, which is described in detail at Mixin and Custom Base Classes.

When our class is constructed, Declarative replaces all the Column objects with special Python accessors known as descriptors; this is a process known as instrumentation. The “instrumented” mapped class will provide us with the means to refer to our table in a SQL context as well as to persist and load the values of columns from the database.

Outside of what the mapping process does to our class, the class remains otherwise mostly a normal Python class, to which we can define any number of ordinary attributes and methods needed by our application.

[1]
For information on why a primary key is required, see How do I map a table that has no primary key?.

Create a Schema
With our User class constructed via the Declarative system, we have defined information about our table, known as table metadata. The object used by SQLAlchemy to represent this information for a specific table is called the Table object, and here Declarative has made one for us. We can see this object by inspecting the __table__ attribute:

>>> User.__table__ 
Table('users', MetaData(bind=None),
            Column('id', Integer(), table=<users>, primary_key=True, nullable=False),
            Column('name', String(), table=<users>),
            Column('fullname', String(), table=<users>),
            Column('nickname', String(), table=<users>), schema=None)
Classical Mappings

The Declarative system, though highly recommended, is not required in order to use SQLAlchemy’s ORM. Outside of Declarative, any plain Python class can be mapped to any Table using the mapper() function directly; this less common usage is described at Classical Mappings.

When we declared our class, Declarative used a Python metaclass in order to perform additional activities once the class declaration was complete; within this phase, it then created a Table object according to our specifications, and associated it with the class by constructing a Mapper object. This object is a behind-the-scenes object we normally don’t need to deal with directly (though it can provide plenty of information about our mapping when we need it).

The Table object is a member of a larger collection known as MetaData. When using Declarative, this object is available using the .metadata attribute of our declarative base class.

The MetaData is a registry which includes the ability to emit a limited set of schema generation commands to the database. As our SQLite database does not actually have a users table present, we can use MetaData to issue CREATE TABLE statements to the database for all tables that don’t yet exist. Below, we call the MetaData.create_all() method, passing in our Engine as a source of database connectivity. We will see that special commands are first emitted to check for the presence of the users table, and following that the actual CREATE TABLE statement:

>>> Base.metadata.create_all(engine)
SELECT ...
PRAGMA main.table_info("users")
()
PRAGMA temp.table_info("users")
()
CREATE TABLE users (
    id INTEGER NOT NULL, name VARCHAR,
    fullname VARCHAR,
    nickname VARCHAR,
    PRIMARY KEY (id)
)
()
COMMIT
Minimal Table Descriptions vs. Full Descriptions

Users familiar with the syntax of CREATE TABLE may notice that the VARCHAR columns were generated without a length; on SQLite and PostgreSQL, this is a valid datatype, but on others, it’s not allowed. So if running this tutorial on one of those databases, and you wish to use SQLAlchemy to issue CREATE TABLE, a “length” may be provided to the String type as below:

Column(String(50))
The length field on String, as well as similar precision/scale fields available on Integer, Numeric, etc. are not referenced by SQLAlchemy other than when creating tables.

Additionally, Firebird and Oracle require sequences to generate new primary key identifiers, and SQLAlchemy doesn’t generate or assume these without being instructed. For that, you use the Sequence construct:

from sqlalchemy import Sequence
Column(Integer, Sequence('user_id_seq'), primary_key=True)
A full, foolproof Table generated via our declarative mapping is therefore:

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, Sequence('user_id_seq'), primary_key=True)
    name = Column(String(50))
    fullname = Column(String(50))
    nickname = Column(String(50))

    def __repr__(self):
        return "<User(name='%s', fullname='%s', nickname='%s')>" % (
                                self.name, self.fullname, self.nickname)
We include this more verbose table definition separately to highlight the difference between a minimal construct geared primarily towards in-Python usage only, versus one that will be used to emit CREATE TABLE statements on a particular set of backends with more stringent requirements.

Create an Instance of the Mapped Class
With mappings complete, let’s now create and inspect a User object:

>>> ed_user = User(name='ed', fullname='Ed Jones', nickname='edsnickname')
>>> ed_user.name
'ed'
>>> ed_user.nickname
'edsnickname'
>>> str(ed_user.id)
'None'
the __init__() method

Our User class, as defined using the Declarative system, has been provided with a constructor (e.g. __init__() method) which automatically accepts keyword names that match the columns we’ve mapped. We are free to define any explicit __init__() method we prefer on our class, which will override the default method provided by Declarative.

Even though we didn’t specify it in the constructor, the id attribute still produces a value of None when we access it (as opposed to Python’s usual behavior of raising AttributeError for an undefined attribute). SQLAlchemy’s instrumentation normally produces this default value for column-mapped attributes when first accessed. For those attributes where we’ve actually assigned a value, the instrumentation system is tracking those assignments for use within an eventual INSERT statement to be emitted to the database.

Creating a Session
We’re now ready to start talking to the database. The ORM’s “handle” to the database is the Session. When we first set up the application, at the same level as our create_engine() statement, we define a Session class which will serve as a factory for new Session objects:

>>> from sqlalchemy.orm import sessionmaker
>>> Session = sessionmaker(bind=engine)
In the case where your application does not yet have an Engine when you define your module-level objects, just set it up like this:

>>> Session = sessionmaker()
Later, when you create your engine with create_engine(), connect it to the Session using sessionmaker.configure():

>>> Session.configure(bind=engine)  # once engine is available
Session Lifecycle Patterns

The question of when to make a Session depends a lot on what kind of application is being built. Keep in mind, the Session is just a workspace for your objects, local to a particular database connection - if you think of an application thread as a guest at a dinner party, the Session is the guest’s plate and the objects it holds are the food (and the database…the kitchen?)! More on this topic available at When do I construct a Session, when do I commit it, and when do I close it?.

This custom-made Session class will create new Session objects which are bound to our database. Other transactional characteristics may be defined when calling sessionmaker as well; these are described in a later chapter. Then, whenever you need to have a conversation with the database, you instantiate a Session:

>>> session = Session()
The above Session is associated with our SQLite-enabled Engine, but it hasn’t opened any connections yet. When it’s first used, it retrieves a connection from a pool of connections maintained by the Engine, and holds onto it until we commit all changes and/or close the session object.

Adding and Updating Objects
To persist our User object, we Session.add() it to our Session:

>>> ed_user = User(name='ed', fullname='Ed Jones', nickname='edsnickname')
>>> session.add(ed_user)
At this point, we say that the instance is pending; no SQL has yet been issued and the object is not yet represented by a row in the database. The Session will issue the SQL to persist Ed Jones as soon as is needed, using a process known as a flush. If we query the database for Ed Jones, all pending information will first be flushed, and the query is issued immediately thereafter.

For example, below we create a new Query object which loads instances of User. We “filter by” the name attribute of ed, and indicate that we’d like only the first result in the full list of rows. A User instance is returned which is equivalent to that which we’ve added:

>>> our_user = session.query(User).filter_by(name='ed').first() 
BEGIN (implicit)
INSERT INTO users (name, fullname, nickname) VALUES (?, ?, ?)
('ed', 'Ed Jones', 'edsnickname')
SELECT users.id AS users_id,
        users.name AS users_name,
        users.fullname AS users_fullname,
        users.nickname AS users_nickname
FROM users
WHERE users.name = ?
 LIMIT ? OFFSET ?
('ed', 1, 0)
>>> our_user
<User(name='ed', fullname='Ed Jones', nickname='edsnickname')>
In fact, the Session has identified that the row returned is the same row as one already represented within its internal map of objects, so we actually got back the identical instance as that which we just added:

>>> ed_user is our_user
True
The ORM concept at work here is known as an identity map and ensures that all operations upon a particular row within a Session operate upon the same set of data. Once an object with a particular primary key is present in the Session, all SQL queries on that Session will always return the same Python object for that particular primary key; it also will raise an error if an attempt is made to place a second, already-persisted object with the same primary key within the session.

We can add more User objects at once using add_all():

>>> session.add_all([
...     User(name='wendy', fullname='Wendy Williams', nickname='windy'),
...     User(name='mary', fullname='Mary Contrary', nickname='mary'),
...     User(name='fred', fullname='Fred Flintstone', nickname='freddy')])
Also, we’ve decided Ed’s nickname isn’t that great, so lets change it:

>>> ed_user.nickname = 'eddie'
The Session is paying attention. It knows, for example, that Ed Jones has been modified:

>>> session.dirty
IdentitySet([<User(name='ed', fullname='Ed Jones', nickname='eddie')>])
and that three new User objects are pending:

>>> session.new  
IdentitySet([<User(name='wendy', fullname='Wendy Williams', nickname='windy')>,
<User(name='mary', fullname='Mary Contrary', nickname='mary')>,
<User(name='fred', fullname='Fred Flintstone', nickname='freddy')>])
We tell the Session that we’d like to issue all remaining changes to the database and commit the transaction, which has been in progress throughout. We do this via Session.commit(). The Session emits the UPDATE statement for the nickname change on “ed”, as well as INSERT statements for the three new User objects we’ve added:

>>> session.commit()
UPDATE users SET nickname=? WHERE users.id = ?
('eddie', 1)
INSERT INTO users (name, fullname, nickname) VALUES (?, ?, ?)
('wendy', 'Wendy Williams', 'windy')
INSERT INTO users (name, fullname, nickname) VALUES (?, ?, ?)
('mary', 'Mary Contrary', 'mary')
INSERT INTO users (name, fullname, nickname) VALUES (?, ?, ?)
('fred', 'Fred Flintstone', 'freddy')
COMMIT
Session.commit() flushes the remaining changes to the database, and commits the transaction. The connection resources referenced by the session are now returned to the connection pool. Subsequent operations with this session will occur in a new transaction, which will again re-acquire connection resources when first needed.

If we look at Ed’s id attribute, which earlier was None, it now has a value:

>>> ed_user.id 
BEGIN (implicit)
SELECT users.id AS users_id,
        users.name AS users_name,
        users.fullname AS users_fullname,
        users.nickname AS users_nickname
FROM users
WHERE users.id = ?
(1,)
1
After the Session inserts new rows in the database, all newly generated identifiers and database-generated defaults become available on the instance, either immediately or via load-on-first-access. In this case, the entire row was re-loaded on access because a new transaction was begun after we issued Session.commit(). SQLAlchemy by default refreshes data from a previous transaction the first time it’s accessed within a new transaction, so that the most recent state is available. The level of reloading is configurable as is described in Using the Session.
