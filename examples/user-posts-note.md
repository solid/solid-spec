# Example: User Posts a Note

**Note:** This example is a component of the parent
[Solid specification](README.md); the parent spec and all its components are
versioned as a whole.

This example is taken from [W3C Social Web
WG](http://www.w3.org/wiki/Socialwg/)'s [user
stories](https://github.com/solid/solid-spec/tree/master/UserStories), where it
is called ["user posts a
note"](http://www.w3.org/wiki/Socialwg/Social_API/User_stories#User_posts_a_note):

 1. Eric writes a short note to be shared with his followers.
 2. After posting the note, he notices a spelling error. He edits the note and
  re-posts it.
 3. Later, Eric decides that the information in the note is incorrect. He
  deletes the note.

Here is how Solid would handle the three steps, using
[curl](http://curl.haxx.se/) as the client application:

1) Eric writes a short note to be shared with his followers. The `Slug` header
is optional but useful for controlling the resulting URL.

```
curl -H"Content-Type: text/turtle" \
     -H"Slug: social-web-2015" \
     -X POST \
     --data ' @prefix as: <http://www.w3.org/ns/activitystreams#>. <> a as:Note; as:content "Going to Social Web WG".' \
     https://eric.example.org/notes/
```

The URL of the new note can be found in the `Location` header returned by the
server. In this example it is likely to be
`https://eric.example.org/notes/social-web-2015` (since the `social-web-2015`
part was politely requested by the `Slug` header).

2) After posting the note, he notices a spelling error. He edits the note and
re-posts it. Solid servers can handle updates in two different ways: a PUT
(overwrite) or a PATCH with `application/sparql-update` content type.

Use HTTP PUT, when you just want to replace the data:

```
curl -H"Content-Type: text/turtle" \
     -X PUT \
     --data ' @prefix as: <http://www.w3.org/ns/activitystreams#>. <> a as:Note; as:content "Going to Social Web WG in Paris".' \
     https://eric.example.org/notes/social-web-2015
```

Or you can use HTTP PATCH with SPARQL if you only want to change certain parts
of the resource, leaving the others unchanged (perhaps because other
applications are modifying them):

```
curl -H"Content-Type: application/sparql-update" \
     -X PATCH \
     --data 'DELETE DATA {<> <http://www.w3.org/ns/activitystreams#content> "Going to Social Web WG" .}; INSERT DATA {<> <http://www.w3.org/ns/activitystreams#content> "Going to Social Web WG in Paris" .} ' \
     https://eric.example.org/notes/social-web-2015
```

If no match is found for the triple to DELETE, the request safely aborts without
changing any data.

3) Later, Eric decides that the information in the note is incorrect. He deletes
  the note.

```
curl -X DELETE https://eric.example.org/notes/social-web-2015
```

Note that all three actions have been performed through RESTful HTTP requests.

In these example, data was sent to the server using `text/turtle` (which is
mandated by LDP), but other content types (such as JSON-LD) could be used if
implemented by servers.
