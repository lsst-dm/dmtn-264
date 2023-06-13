:tocdepth: 1


Abstract
========

A lot of astronomy data is stored in SQL databases.  But although
many people think SQL stands for "Standard Query Language," it
actually stands for "Structured Query Language."  This is
unfortunate for the users of the data, since depending on the
brand of SQL database used (Microsoft SQL Server, Postgres, MySQL,
etc.) many things can change.

The IVOA (International Virtual Observatory Alliance) therefore
created a standard named TAP (Table Access Protocol).  The TAP
standard tries to standardize both the SQL queries, as well
as the results (specifically data types), to make it possible
that the same query string can be run against data stored
in any brand of SQL server, and return the same data.

Reading List and References
===========================

1. TAP standard https://www.ivoa.net/documents/TAP/20190927/REC-TAP-1.1.pdf
2. TOPCAT client https://www.star.bris.ac.uk/~mbt/topcat/
3. CADC TAP codebase https://github.com/opencadc/tap
4. QServ https://qserv.lsst.io/index.html
5. QServ compatible CADC TAP Service https://github.com/lsst-sqre/lsst-tap-service

What does TAP solve?
====================

TAP tries to standardize access to Tables stored
in a database.

First, the user (and TAP service) needs to be
able to discover what data is in the backend,
including things like what tables exist,
the columns in those tables, and the datatypes
that those columns are.  This is solved by the
TAP_SCHEMA database.

The TAP service also needs to standardize the
SQL queries that users send to the backend
database, as well as standardizing the results
in terms of things like datatypes.

TAP_SCHEMA
----------

In order for the TAP server to know what
tables (and columns) exist in the backend
database, there has to be a database
named TAP_SCHEMA.  This database has a
standard set of tables and columns that
must exist, and are outlined in the TAP
standard.

The TAP_SCHEMA database can be queried
by both the TAP server, as well as the
users executing queries against the
TAP service.

The TAP_SCHEMA database can be used
to discover what tables exist, what
columns are in these tables, and what
datatypes these columns are.  If the
TAP_SCHEMA database doesn't know about
a table or column, the user should not
be allowed to query it.

The Query Side
--------------

Different SQL servers sometimes use different
grammar to mean basically the same thing, or
even different things.

One easy example is that some databases use
the word LIMIT, while others use the word TOP.
In this case the word means the same thing,
but it is just a different word.  This means
the person executing the query needs to know
what type of database is behind the TAP server,
meaning it isn't standard.  The TAP specification
usually picks one of these words and says that
is the standard.  For example, in this case,
the word is LIMIT.  But when hooked up to 
SQL servers that use TOP, the query must be
changed or "rewritten" to change TOP to LIMIT.

While TOP/LIMIT may be trivial, one place
where it is less trivial is queries relating
to geometry.  Many times each SQL server has
its own set of functions to use to do
spatial geometry, and these functions are not
only different, but many times have different
numbers of functions and parameters.

The Results Side
----------------

Once the query is executed, a SQL results set
is returned.  While many times the results sets
seem similar, different SQL servers can provide
different datatypes in the table.

One example is booleans.  Some SQL servers
just assume you will use an integer with 0 or 1
to represent true and false.  Other SQL servers
have a specific boolean datatype that handles
this for you.  Different SQL servers also provide
different numerical types to also watch out for.
Strings also aren't necessarily standard.

When the TAP service goes through each row of
the results, the TAP service needs to change the
data returned to a standard set of TAP datatypes
outlined in the standard.  Sometimes this can
be complicated.

Many times the results are returned immediately.
This is considered a "synchronous" request.

But in the TAP standard, since some queries
can take a long time to execute, you can execute
an "asynchronous" request.  In this mode, the
TAP service returns a document that contains
a URL to poll for the result being finished
and then to receive the results when the request
is finished.

Note: This is completely different than async
mode that QServ allows for.  We do not currently
use that mode.

UWS Database
------------

In order to keep track of which queries
are running, including who is running, and what
the URLs are for the results, the Universal
Worker Service Pattern is used.  This is a
standard from the IVOA that outlines what
the URLs are.  The data for this service
is stored in a postgres database that the
TAP service (and only the TAP service)
connects to.

The standard can be found here:

https://www.ivoa.net/documents/UWS/

If you want to be able to see what queries
are running, you can use kubectl to create
a shell in the UWS pod, and locally connect
to the postgres database.  This stores what
the status of each query is, including what
user has created the query.

TAP at Rubin
============

Right now we use the CADC TAP service, which has been
developed by the Canadian Astronomy Data Centre.  This
codebase is both well coded and mature.  The CADC TAP
service can run against a few different backends,
including Postgres, MySQL, and Oracle.  The CADC TAP service
is the best codebase that can be easily used
from GitHub.

But there will always be needs to modify or extend
code.  The CADC TAP service provides many hooks and
is well architected to allow for new databases to
be connected to it.

TAP + Databases
===============

Now that we've outlined the general tricky
spots of the TAP service, let's get specific.

At Rubin we use a few different SQL databases.
Mainly QServ and Postgres.  Others may be
added in the future though.

QServ is based on MySQL and Postgres is
Postgres.  Both of these have completely
different geometry libraries.  We will be
using the geometry libraries heavily to be
able to write queries such as "the Circle
around this point in the sky" or "the
distance between these two points is
greater than or less than X."

TAP + QServ
-----------

The biggest thing to know about QServ is
that it is a distributed database, built
on top of MySQL and
uses some of the functions in the query
to determine what databases the query
should be run on.  But that should be
a hidden feature of QServ.

The way to use spatial constraints
are outlined here: 

https://qserv.lsst.io/user/index.html#spatial-constraints

The datatypes that are returned
are based on the MySQL datatypes.

In order to get TAP to run on QServ,
there are a number of changes and
plugins that needed to be used.  The
QServ compatible TAP server are
located at: 

https://github.com/lsst-sqre/lsst-tap-service

You can see where the query is rewritten
for QServ here:

https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/QServRegionConverter.java

TAP + Postgres
--------------

CADC uses TAP with Postgres, so this is
very well supported.  Here's the code
on GitHub:

https://github.com/lsst-sqre/tap-postgres

The spatial library that Postgres
uses is called pgsphere:

https://pgsphere.github.io/

TAP Instances
=============

There are a couple of different TAP instances,
where a TAP server connects to a different
backend database.

You can check in the Phalanx repository if
you think this list is incomplete.  You can
find that repository here:

https://github.com/lsst-sqre/phalanx

There are currently 3 different TAP services running:

1. TAP for QServ: https://github.com/lsst-sqre/phalanx/tree/main/applications/tap
2. SSO TAP (Postgres): https://github.com/lsst-sqre/phalanx/tree/main/applications/ssotap
3. LiveTAP (Postgres): https://github.com/lsst-sqre/phalanx/tree/main/applications/livetap

TAP Configuration
-----------------

Each of these TAP instances is controlled by
configuration located in the phalanx repository.
This configuration can be found in the directories
referenced above, and in those directories, you can
find files named values then followed by an environment.

Each of these values files control the configuration
for that particular environment.

For example:

https://github.com/lsst-sqre/phalanx/blob/main/applications/livetap/values-usdfdev.yaml controls the LiveTAP service for the
environment usdfdev.  This controls things like
resource limits for different databases as well
as configure what backend it points at.

TAP_SCHEMA instances
--------------------

You can see what tables and columns are presented
to the user by looking at what TAP_SCHEMA database
is used for each TAP instance.

This can be found in the values files
and looking for the configuration marked
tap_schema.  You will see a container such as:
"lsstsqre/tap-schema-usdf-dev-livetap"

You can then look at the sdm_schemas repository
located at: https://github.com/lsst/sdm_schemas

In the tap-schema directory, you will find a
script named build-all.  In that script you
can see what yaml files are used to create
that container.  For livetap you can see
the line here:

./build usdf-dev-livetap ../yml/oga_live_obscore.yaml

This means that the TAP_SCHEMA container
tap-schema-usdf-dev-livetap contains the tables and
columns referenced in the oga_live_obscore.yaml file
located in the same repository.

Some of these TAP_SCHEMA containers reference multiple
yaml files, which combines all of those yaml files
together.

TAP Vault Configuration
-----------------------
There are also some configuration like passwords
located in the vault instance in each environment.
These vault instances have different keys to point
to the instance referencing passwords and GCS files.

The pattern for the vault key to use is:

secrets/k8s_operator/environment/tap_instance

So for data-dev.lsst.codes, the TAP instance
would be:

secrets/k8s_operator/data-dev.lsst.codes/tap

For environments located at SLAC, you have to use
a different pattern, which is

secrets/secret/rubin/environment/tap_instance

This can be accessed through the webpage:

https://vault.slac.stanford.edu/ui/vault/

The keys for a TAP + QServ instances are simply
google_creds.json, which must contain a JSON
document that contains the credentials to
access a GCS bucket.

For a TAP + Postgres instance, this requires
both the google_creds.json key, which contains
a JSON document, and a key named pgpassword
which contains the password for the Postgres
database to connect to.
