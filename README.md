# The Long Road Towards a Universal REST API

Part 1: Design Problems for a Universal REST API
Part 2: Permissions (Authentication and Authorization)
Part 3: Querying, Filtering and Sorting

## The Problem

Even before Roy Fielding nailed his REST thesis to the church doors, app
designers and API architects have been dreaming of a (hopefully simple) general
purpose REST API for reading and writing data over HTTP.

The Web was supposed to be a Read-Write Web. We have the *read* part of that
down, we've gotten pretty good over the years, but what about writing or
updating data? Sure, we have the `POST` verb, and it's easy enough to wire
together an HTML form with a Submit button. But beyond a trivial app, a
developer is faced with an endless string of design decisions (and bikeshedding
arguments) about data modeling, API semantics and error handling.

Let's get a few points out of the way, first. We _don't_ think that REST (or
even HTTP) is the One True way of doing inter-application and inter-service
APIs. The arguments made in [REST is the new
SOAP](https://medium.freecodecamp.org/rest-is-the-new-soap-97ff6c09896d) and
[its follow-up](https://medium.com/@pakaldebonchamp/follow-up-to-rest-is-the-new-soap-the-origins-of-rest-21c59d243438)
are valid ones. For complex action- and process-oriented API calls, it very well
may be that the RPC methodology (especially in its modern JSON-RPC and gRPC
incarnations) is the saner approach. But even the author of those arguments
admits that for certain use cases (such as an administrative interface to data
stores), a simple mapping of HTTP verbs to CRUD actions is a good match. And
that's what we want to talk about here.

However, even in simple CRUD REST API land, not all is well. Why does every
database (SQL and NoSQL) and every general purpose data store service have a
very similar but subtly incompatible REST API?

The design space is so similar! We've got collections/containers, we've got
items in those collections -- pretty much what REST was made for, right? Almost
everyone can agree that the GET, PUT, POST and DELETE HTTP verbs roughly map to
CRUD operations for resources. But beyond that, the agreement stops.

Every time a team sits down to design a general purpose HTTP-based CRUD API,
they are faced with a series of design decisions. And their choices in these
decisions is what accounts for all the similar but incompatible APIs out there.

Some of the important decisions are:

1. How to access Contents vs Metadata
2. Whether to allow partial gets and partial updates
3. Flat container structure or allow nested containers
4. Single-type containers/collections, or allow mixed type.
  - Also, does the data store make provisions for schema discovery or schema enforcement?
5. How to handle blobs and file attachments (assuming that much of the focus is on structured data of some sort, such as tables or documents)
6. Error and Exception formats

We'll examine the current state of the art and solutions to each of these
questions, and then make a proposal for **JSON Web Collections**, a general
purpose REST storage API.

There are of course other key considerations. One is about the fundamental data
model used to store data, but that's often dictated by the team's choice in the
underlying data store, whether it's a relational database, a JSON document
store, a graph store, plain key/value, or other exotic types like columnar
stores.

Another set of considerations is - what sort of permission structure will this
API have? What methods to use for authentication? More importantly, how
fine-grained will the access control be? Do you authenticate to the whole API
and then have full access? Access control per collection/table? Per item? What
about even finer-grained, maybe access control filter on individual attributes?
We'll address permissions in Part 2 of this series.

And lastly, if you've set up your general-purpose CRUD API (and have thought
through the auth questions), do you leave it at the simple four key/value CRUD
verbs, or do you allow for some sort of filtering, sorting and even outright
queries? We'll take a look at the various attempts to achieve general purpose
querying languages (specifically for data APIs), from OpenAPI to JSON-API to
GraphQL, and others, in Part 3.

## Terminology

(not sure if this section is needed)
- discuss collections / containers vs individual resources

## 1. Accessing Contents vs Metadata

Though we typically think of a resource (residing at a URL) as a single thing,
it's actually two or three things under the hood -- the *contents* of the
resource, its *metadata* (who created it and when, what the permissions are to
access it, its content type or schema, misc settings, etc), and sometimes even
(in the case of documents in a document db like Mongo or Couch) the list of
*binary attachments* to that resource. So, how do you access those different
parts of the object? Several solutions have evolved over the years, including
using well-known structured URLs (how most database HTTP APIs do it), using
different HTTP verbs to access the contents of the object vs its metadata (see
`GET` vs `PROPFIND` in WebDAV), linking to the metadata in HTTP headers
(Solid/LDP), linking to the metadata in the contents itself
([JSON-API](http://jsonapi.org/), Collections+JSON, Hydra), or putting the
metadata directly into the headers instead of linking (Riak HTTP API). This may
seem like a minor problem, but it's actually the source of most of the API
incompatibility out there.

A collection (or folder, or container) is basically two things: 1) a list of
contents, and 2) a bunch of metadata (including an optional schema for the
contents, replication settings, backend storage configs, permissions and access
control, *indexes*, and so on).

Similarly, any resource or document (at a given URL) is also at least two
different things: 1) body or contents, and 2) metadata (creator id, created_at
and updated_at timestamps, ACL/permissions, schema (if not inherited from
collection), etc etc).

This may seem obvious, but the difficulty (and lack of interop) starts when you
ask a simple question - how do you access a given collection's or item's
metadata (vs accessing its contents)?

Let's look at existing systems and their solutions to this question.

### 1.1 Well-Known URL Patterns

The most common solution to separating contents vs metadata is to put them at
slightly different URLs. For example:

#### Riak

* `/buckets/<name>/props` - collection metadata
* `/buckets/<name>/keys` - collection contents
* `/buckets/<name>/keys/<key>` - a single item

#### MongoDB
Has many REST API interfaces, here's a typical one:

[RESTHeart MongoDB API Reference](https://softinstigate.atlassian.net/wiki/spaces/RH/pages/9207882/Reference+sheet)

* `/<dbname>/<collname>` - collection contents
* `/<dbname>/<collname>/<docid>` - a single item
* `/<dbname>/<collname>/_indexes/` - collection (index) metadata. Btw, what if your doc id just *happens* to be `_indexes`? How do you tell that apart from the previous pattern? Hopefully the API checks for that...
* `/<dbname>/<collname>/_indexes/<indexid>`
* `/<dbname>/_schemas` - note that schemas live under a different uri space

#### CouchDB

CouchDB has no collections (that is, a database is a single mixed collection).

* `	/{db}/{docid}` - single item
* `/{db}/{docid}/{attname}` (doc attachments for that item)
* Get a view: `GET /{db}/_design/{ddoc}/_view/{view}`
* Deleting an index: `DELETE /{db}/_index/{designdoc}/json/{name}`

#### PostgresQL

[Postgres HTTP API](https://wiki.postgresql.org/wiki/HTTP_API)

* `/databases/<database_name>`
* `/databases/<database_name>/schemas/<schema_name>`
* `/databases/<database_name>/schemas/<schema_name>/table/<table_name>/indexes/<index_name>`

#### Mozilla Kinto

[Kinto](http://docs.kinto-storage.org/) is Mozilla's generic JSON store with
sharing and sync capabilities. State of the art, as far as this REST style goes.

* `/buckets/(bucket_id)/groups`
* `/buckets/(bucket_id)/collections/(collection_id)`
* `/buckets/(bucket_id)/collections/(collection_id)?field.subfield=value` - Querying / filtering
* `/buckets/(bucket_id)/collections/(collection_id)/records`
* `/buckets/(bucket_id)/collections/(collection_id)/records/(record_id)`

##### OData

Microsoft's (and later, OASIS's) [Open Data
Protocol](https://en.wikipedia.org/wiki/Open_Data_Protocol) also aimed to be a
general purpose read-write API.

```
http://host:port/path/SampleService.svc/Categories(1)/Products?$top=2&$orderby=Name
\______________________________________/\____________________/ \__________________/
                  |                               |                       |
          service root URL                  resource path           query options
```

---
All of these designs follow a similar pattern.

Some designer said, "OK, when you access a collection URL, you get its
*contents*!" (ie `GET /users/  -> contents`). Ok, then how do you access
metadata for that collection? "How about, `GET /<collection name>/props` or `GET
/<collection name>/schemas`?"

Some other designer started with "When you access a collection URL, you get its
*metadata*!". So, `GET /users/ -> properties object (schemas, indexes, etc)`.
How about the contents? Why not do `GET /<collection name>/records` to get the
contents. (Or `/keys` or `/items`).

It gets even worse, because if you go this "URL pattern" route, you can't just
have a simple `/<collection name>/<item id>` scheme. Because if your metadata
lives in `/<collection name>/props`, what if you have a collection item with the
id of `props`? Do you filter it out? So then you have a separator keyword to
prevent having to do that, like `/<collection name>/records/<id>` so that even
if somebody is named `props`, you can tell `/users/props` apart from
`/users/records/props`.

And of course, every database or generic storage API uses slightly different
keywords to separate their URLs.

### 1.2 Using different HTTP verbs

WebDAV, a general "let's map the file system to the web" REST storage protocol
from the dawn of time (spec was written 1997-2007) solved this problem (of
contents vs metadata) by creating *several new HTTP verbs*. So, if you did a
`GET /<folder>/<resource>` you got the contents of that resource, but if you did
`PROPFIND /<folder>/<resource>`, then you got its metadata. So, same URL, but
different HTTP verbs.

WebDAV also registered another half dozen new HTTP verbs for various other
operations, some useful (like the `COPY` verb) and some less so (creating
folders uses a separate `MKCOL` verb, instead of a `POST` with different
parameters).

### 1.3 Metadata Embedded in HTTP Headers

A rarely-taken approach (I can only think of Riak's HTTP API that did this, and
even then it was also supplemented with URL patterns from 1.1) is to embed
various metadata attributes completely in HTTP headers. That way, if you `GET`
an object, you get its contents + metadata, and if you just want metadata, you
can issue a `HEAD` request.

I suspect this approach didn't catch on because a) There's a practical limit on
the length of an individual HTTP header, most likely, and b) Object size (you
always get the metadata even if you only wanted the contents), although note
that method 1.5, below, also has this limitation.

### 1.4 Linking to Metadata in HTTP Headers

A slightly modified version of the previous strategy just uses headers to *link*
to one or more documents that contain a resource's metadata, instead of
embedding the metadata directly.

For example, here is how the [Solid](https://github.com/solid/solid-spec) REST
protocol (a profile of the RDF-based [Linked Data
Platform](https://www.w3.org/TR/ldp/) spec) links to an object's metadata and
permissions (ACL):

`GET https://example.com/index.json`

```
GET /index.json HTTP/1.1

HTTP/1.1 200 OK
Content-Type: application/json
Link: <https://example.com/index.json.acl>; rel="acl",
Link: <https://example.com/index.json.meta>; rel="meta"

<object contents...>
```

#### 1.5 Metadata in Object Contents

The other really common approach (aside from 1.1/using URL patterns), is to
embed (or link to) metadata in the *object contents itself*.

Atom and AtomPub does this. So do many other general-purpose "Hypermedia" REST
APIs,.

For example, here's a [JSON-API](http://jsonapi.org/format/) object:

```
{
  "meta": {
    "copyright": "Copyright 2015 Example Corp.",
    "authors": [
      "Yehuda Katz",
      "Dan Gebhardt"
    ]
  },
  "data": {
    // object contents
  },
  "links": [ // ... ]
}
```

Incidentally, this is an approach similar to that of JWT / JWD (with separates
the object into `header` and `payload`).

Similarly, here's how [Mike Amundsen's
Collection+JSON](https://github.com/collection-json/spec) represents a
collection's contents (`items`) and its metadata (`links`, `version`, `template`
etc):

```
GET /friends/

{ "collection" :
  {
    "version" : "1.0",
    "href" : "http://example.org/friends/",

    "links" : [
      {"rel" : "feed", "href" : "http://example.org/friends/rss"}
    ],

    "items" : [
      {
        "href" : "http://example.org/friends/jdoe",
        "data" : [
          {"name" : "full-name", "value" : "J. Doe", "prompt" : "Full Name"},
          {"name" : "email", "value" : "jdoe@example.org", "prompt" : "Email"}
        ],
        "links" : [
          {"rel" : "blog", "href" : "http://examples.org/blogs/jdoe", "prompt" : "Blog"},
          {"rel" : "avatar", "href" : "http://examples.org/images/jdoe", "prompt" : "Avatar", "render" : "image"}
        ]
      },
    ],

    "queries" : [
      {"rel" : "search", "href" : "http://example.org/friends/search", "prompt" : "Search",
        "data" : [
          {"name" : "search", "value" : ""}
        ]
      }
    ],

    "template" : {
      "data" : [
        {"name" : "full-name", "value" : "", "prompt" : "Full Name"},
        {"name" : "email", "value" : "", "prompt" : "Email"},
        {"name" : "blog", "value" : "", "prompt" : "Blog"},
        {"name" : "avatar", "value" : "", "prompt" : "Avatar"}

      ]
    }
  }
}
```

Here is how [Hydra Core](https://www.hydra-cg.com/spec/latest/core/) represents a collection (`member` is its contents, the rest is metadata):

```
{
  "@context": "http://www.w3.org/ns/hydra/context.jsonld",
  "@id": "http://api.example.com/an-issue/comments",
  "@type": "Collection",
  "totalItems": "4980",
  "member": [
    {
      "@id": "/comments/429"
    },
    {
      "@id": "/comments/781",
      "title": "Properties may be embedded directly in the collection"
    },
    ...
  ]
}
```

#### 1.6 Recommended Solution

TBD (method 1.4, linking to metadata in HTTP headers, seems to make the most sense)

### 2. Accessing Document Attributes/Sub-Trees/Sub-Graphs

Summary - Databases and storage systems that let you fetch and update parts of a document, or "walk the document tree" are way better than full fetches and updates only.

Is it possible to fetch just a part of the document? (An attribute or a sub-tree, for document stores). Some document stores allow it (and some, like Firebase and IPLD go so far as to model the whole database as a single traversable document tree), and some do not. Similarly, are partial updates (PATCHes) allowed?

[...]

Examples:

* Firebase API: `<project>.firebaseio/<"collection"/list>/<id>/<attribute>.json` - walk the json doc tree
* [IPLD - InterPlanetary Linked Data](https://github.com/ipld/specs/tree/master/ipld) spec

### 3. Nested Folders or Collections

Organizing records or documents into collections (or folders or buckets) makes intuitive sense. Should you allow nested collections, or just a flat namespace (collection/item)? That is, should you be able to have `//departments/Accounting/users/Alice`? Or do you only have `//departments/Accounting`, which has a list of contents that links to `//users/Alice` and so on?

WebDAV, Solid/LDP (as well as Firebase, in a sense) allow arbitrary nesting of collections within collections. Nested collections, and items within them. For example:

`GET /departments/accounting/users/admins/user123`

Most of the other databases or storage APIs (Kinto, Mongo, Couch, Facebook's Parse, etc) do not -- a "flat" hierarchy of collections only. (Partly due to their use of URL templates, see section 1.1).

[...]

## 4. Schemas, Single Type vs Mixed Collections

Some systems only allow documents with the same structure or schema to reside in a collection, and some are schema-agnostic (for example, you can throw any sort of document into a CouchDB database). What about completely different content types? Should you be able to mix images, Word docs, and JSON documents in a single collection or folder?
  - schema discovery, schema enforcement

WebDAV and LDP allow mixed resources (JSON docs, RDF, binary files) in the same collection (as well as nested collections). CouchDB databases are schema-less, as are MongoDB collections (though the notion of schemas comes in immediately as you try and set up indexes).

## 5. Files and Binary Attachments to Documents

Similar to the previous item, if you're a document store, how do you model reading and writing binary attachments to a document?

Couch: `/{db}/{docid}/{attname}` (doc attachments for that item)

## 6. Error handling

TBD

## JSON Web Collections Proposal

TBD
