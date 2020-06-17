# Solid WebSockets API Spec

**Note:** This spec is a component of the parent
[Solid specification](README.md); the parent spec and all its components are
versioned as a whole.

## Status
This is a _draft_ protocol.
This specific version is identified by the string `solid-0.1-alpha`.

## Protocol description

Live updates are currently only supported through WebSockets. This describes a
subscription mechanism through which clients can be notified in real time of
changes affecting a given resource.

### Discovery
The PubSub system is very basic.
First, a client needs to obtain the URI of the WebSocket.
The WebSocket server URI is the same for any resource
located on a given data space (same hostname).
To discover the URI of the WebSocket server,
clients can send an HTTP `OPTIONS` request.
The server will then include an `Updates-Via` header in the response:

REQUEST:

```http
OPTIONS /data/test HTTP/1.1
Host: example.org
```

RESPONSE:

```http
HTTP/1.1 200 OK
...
Updates-Via: wss://example.org
```

### Connection
Then, the client needs to open a WebSocket connection
to that URI.
The client _SHOULD_ include the protocol version `solid-0.1-alpha`
in the `Sec-WebSocket-Protocol` header.

For example, in JavaScript, this could be done as follows:

```
const socket = new WebSocket('wss://example.org', ['solid-0.1-alpha']);
```

Upon connection,
the server SHOULD indicate the protocol version as follows:

```
protocol solid-0.1-alpha
warning Unstandardized protocol version, proceed with care
```

If the client did not specify a `Sec-WebSocket-Protocol` header,
the server SHOULD warn the client as follows:

```
warning Missing Sec-WebSocket-Protocol header, expected value 'solid-0.1-alpha'
```

Otherwise, if the set of values obtained
from parsing the `Sec-WebSocket-Protocol` header
does not contain `solid-0.1-alpha`,
then the server SHOULD emit a warning
and SHOULD close the connection:

```
error Client does not support protocol solid-0.1-alpha
```

### Subscription
Then, the client needs to *sub*(scribe) to a given resource URI.
If any change occurs in that resource,
a *pub*(lish) event will be sent to all the subscribed clients.

To subscribe to a resource, clients will need to send the keyword `sub` followed
by an empty space and then the URI of the resource:

```
sub https://example.org/data/test
```

If a change occurs and the client is subscribed to that resource, it will
receive a WebSocket message composed of the keyword `pub`, followed by an empty
space and the URI of the resource that has changed:

```
pub https://example.org/data/test
```

Only absolute URIs should be used in both the `sub` and the `pub` message.

Subscribing to a container can also be really useful, since all CRUD operations
(POST, PUT, PATCH, DELETE) performed on resources of that container will trigger
a notification for the container URI. This makes synchronization between
multiple apps really easy.

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

### Example
Here is a Javascript example on how to subscribe to live updates for a `test`
resource at `https://example.org/data/test`:

```js
var socket = new WebSocket('wss://example.org/', ['solid-0.1-alpha']);
socket.onopen = function() {
	this.send('sub https://example.org/data/test');
};
socket.onmessage = function(msg) {
	if (msg.data && msg.data.slice(0, 3) === 'pub') {
        // resource updated, refetch resource
	}
};
```
