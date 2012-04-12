# Editable webservice specification

This documents the changes which will be made to the /ws/2/ webservice
to make entitities editable through the webservice.

## Overview 

In general, editing an entity should be as simple as performing a GET
request on a resource, making changes to the document you recieved,
and pushing the changed document back to the server in the body of a
PUT request.

The body of a PUT request should contain a full document, any elements
missing from the document will be deleted from the entity (except for
those elements which inherit from other elements in the document if
they're omitted, e.g. track titles are inherited from recording titles
if track and recording titles are identical, and artist credits
inherit from artist names).

However, for this to work without unexpected changes the server should
be aware of any changes made to the document by someone else, between
the time of the GET request and the subsequent PUT request from the
client.  I believe this is usually solved by either versioning or
hashing documents, or using some kind of transaction ID. For us,
versioning is troublesome because a document returned by the
webservice is composed from many distinct database tables, and
transaction IDs would be difficult to manage unless every change to
the database goes through the webservice.  Thus hashing seems most
suitable.  This means that when a client submits a PUT request along
with the hash of the original document, the server then needs to hash
the current data itself, compare the hashes, and reject the PUT if
anything changed.  (All of this to prevent an edit from inadvertantly
reverting changes from other edits).

Generally, to get a full document you currently have to include a
number of ?inc= arguments, the XML property on the "details" tab on
the musicbrainz web page for an entity lists the url with all inc
arguments required to get a full document.
(NOTE: need to check if this is true for all entities. --warp)

To avoid having to know which ?inc= arguments to supply to get a full
document I suggest we need a simple way to request the full document
for a resource, as an alternative to using ?inc=.
(e.g. a ?for_edit=true, or make the default response without any
 ?inc= the full document, or some http header in the request).


### /ws/2/{entity}/

#### Summary

Create an entity.

NOTE: for backward compatibility, /ws/2/recording/ and /ws/2/release/
will also accept ISRC and barcode submits to this url using a POST
request.  The body of these requests contain a *recording-list* or
*release-list*, and are thus easily distinguished from proper requests
to this resource which would just have a single *recording* or
*release* element as child of *metadata*.

ref: http://wiki.musicbrainz.org/XML_Web_Service/Version_2#Submitting_data

#### Accepted Methods

- POST
- HEAD

#### Responses

- 201 Created, if the edit to add this entity to the database has been
  created.  The Location header of the response contains the full url to
  the webservice resource which describes the edit.

- 409 Conflict, if this entity already exists.  E.g. the submitted
  document describes an artist with an artist name which already
  exists in the database, and no disambiguation comment was provided.

- 412 Precondition Failed, if the document has had changes between the
  time the client requested a copy and submitted its changes.

### /ws/2/{entity}/{mbid}/

#### Summary

Read, Update or Delete an entity.

#### Accepted Methods

- PUT
- DELETE
- HEAD

#### Responses

- 201 Created, if the edit to make the requested changes has been
  created.  The Location header of the response contains the full url
  to the webservice resource which describes the edit.

[ if multiple edits are created, should we just link to all of them in
  the 201 body, and pick one for the location header -- or should we
  use "207 Multi-Status" from WEBDAV ? ]

- 409 Conflict, if a change to the entity creates a conflict with some
  other entity in the database. E.g. the submitted document now
  describes an artist with an artist name which already exists in the
  database, and no disambiguation comment was provided.

- 412 Precondition Failed, if the document has had changes between the
  time the client requested a copy and submitted its changes.


### /ws/2/{edit}/{id}/

#### Summary

Post edit notes or vote on the edit.

(It should be possible to submit an edit note and a vote in a single
 request, a POST to this resource allows this).

#### Accepted Methods

- GET (not needed immediatly, but would be nice to provide eventually)
- POST
- HEAD

#### Responses

- 200 OK, if the edit note and/or vote submitted in a POST request were
  accepted.

- 403 Forbidden, if the user submitted a POST request but is not allowed
  to vote or add notes to this edit.
