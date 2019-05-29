# Solid WebSockets API Spec

**Note:** This spec is a component of the parent
[Solid specification](README.md); the parent spec and all its components are
versioned as a whole.

## Subscribing

Live updates are currently only supported through WebSockets. This describes a
subscription mechanism through which clients can be notified in real time of
changes affecting a given resource.

The PubSub system is very basic. Clients only need to open a WebSocket
connection and *sub*(scribe) to a given resource URI. If any change occurs in
that resource, a *pub*(lish) event will be sent to all the subscribed clients.

The WebSocket server URI is the same for any resource located on a given data
space (same hostname). To discover the URI of the WebSocket server, clients can
send an HTTP OPTIONS request. The server will then include an `Updates-Via` header in
the response:

REQUEST:

```http
OPTIONS /data/test HTTP/1.1
Host: example.org
```

RESPONSE:

```http
HTTP/1.1 200 OK
...
Updates-Via: wss://example.org/?access_token=Ne3jiZ1Mei6Air6iefoh
```

The URL mentioned in the `Updates-Via` header should act as a [capability](https://en.wikipedia.org/wiki/Capability-based_security).
To subscribe to a resource, clients will need to send the keyword `sub` followed
by an empty space and then the URI of the resource:

```
sub https://example.org/data/test
```

If a change occurs and the client is subscribed to that resource and has Read access to it, it will
receive a WebSocket message composed of the keyword `pub`, followed by an empty
space and the URI of the resource that has changed:

```
pub https://example.org/data/test
```

Subscribing to a container can also be really useful, since all CRUD operations
(POST, PUT, PATCH, DELETE) performed on member resources of that container will trigger
a notification for the container URI. This makes synchronization between
multiple apps really easy. It only affects the parent container, of which the resource is a member,
not further ancestor containers.

For example, a client subscribes to the `data/` container:

```
sub https://example.org/data/
```

If another client deletes the resource `foo` inside `data/`:

REQUEST:

```http
DELETE /data/foo HTTP/1.1
Host: example.org
```

Then the following notification message will be sent:

```
pub https://example.org/data/
```

The server may send additional informational messages for e.g. error reporting,
as long as they don't start with `pub`.
If a client subscribes to too many updates, the server may close the socket.

Here is a Javascript example on how to subscribe to live updates for a `test`
resource at `https://example.org/data/test`:

```js
var socket = new WebSocket('wss://example.org/');
socket.onopen = function() {
	this.send('sub https://example.org/data/test');
};
socket.onmessage = function(msg) {
	if (msg.data && msg.data.slice(0, 3) === 'pub') {
        // resource updated, refetch resource
	}
};
```
