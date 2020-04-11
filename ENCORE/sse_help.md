
# HTML5 Server Sent Event's

![](https://spacloud.eu/public/9786582aaf93e/assets/sse.png)

## JavaScript API

To subscribe to an event stream, create an `EventSource` object and pass it the URL of your stream:

```json
if (!!window.EventSource) {
  var source = new EventSource('stream.php');
} else {
  // Result to xhr polling :(
}
```

Note: If the URL passed to the `EventSource` constructor is an absolute URL, its origin (scheme, domain, port) must match that of the calling page.

Next, set up a handler for the `message` event. You can optionally listen for `open` and `error`:

```json
source.addEventListener('message', function(e) {
  console.log(e.data);
}, false);

source.addEventListener('open', function(e) {
  // Connection was opened.
}, false);

source.addEventListener('error', function(e) {
  if (e.readyState == EventSource.CLOSED) {
    // Connection was closed.
  }
}, false);
```

When updates are pushed from the server, the `onmessage` handler fires and new data is be available in its `e.data` property. The magical part is that whenever the connection is closed, the browser will automatically reconnect to the source after ~3 seconds. Your server implementation can even have control over this reconnection timeout.  **See Controlling the reconnection-timeout in the next section.

That's it. Your client is now ready to process events from stream.php.

## Event Stream Format

Sending an event stream from the source is a matter of constructing a plaintext response, served with a `text/event-stream` Content-Type, that follows the SSE format. In its basic form, the response should contain a "`data:`" line, followed by your message, followed by two "\\n" characters to end the stream:

```json
data: My message\\n\\n
```
### Multiline Data

If your message is longer, you can break it up by using multiple "`data:`" lines. Two or more consecutive lines beginning with "`data:`" will be treated as a single piece of data, meaning only one `message` event will be fired. Each line should end in a single "\\n" (except for the last, which should end with two). The result passed to your `message` handler is a single string concatenated by newline characters. For example:

```json
data: first line\\n
data: second line\\n\\n
```
will produce "first line\\nsecond line" in `e.data`. One could then use e`.data.split('\\n').join('')` to reconstruct the message sans "\\n" characters.

### Send JSON Data

Using multiple lines makes it easy to send JSON without breaking syntax:

```json
data: {\\n
data: "msg": "hello world",\\n
data: "id": 12345\\n
data: }\\n\\n
```
and possible client-side code to handle that stream:

```json
source.addEventListener('message', function(e) {
  var data = JSON.parse(e.data);
  console.log(data.id, data.msg);
}, false);
```
### Associating an ID with an Event

You can send a unique id with an stream event by including a line starting with "`id:`":

```json
id: 12345\\n
data: GOOG\\n
data: 556\\n\\n
```
Setting an ID lets the browser keep track of the last event fired so that if, the connection to the server is dropped, a special HTTP header (`Last-Event-ID`) is set with the new request. This lets the browser determine which event is appropriate to fire. The `message` event contains a `e.lastEventId` property.

### Controlling the Reconnection-timeout

The browser attempts to reconnect to the source roughly 3 seconds after each connection is closed. You can change that timeout by including a line beginning with "`retry:`", followed by the number of milliseconds to wait before trying to reconnect.

The following example attempts a reconnect after 10 seconds:

```json
retry: 10000\\n
data: hello world\\n\\n
```

### Specifying an event name

A single event source can generate different types events by including an event name. If a line beginning with "`event:`" is present, followed by a unique name for the event, the event is associated with that name. On the client, an event listener can be setup to listen to that particular event.

For example, the following server output sends three types of events, a generic 'message' event, 'userlogon', and 'update' event:

```json
data: {"msg": "First message"}\\n\\n
event: userlogon\\n
data: {"username": "John123"}\\n\\n
event: update\\n
data: {"username": "John123", "emotion": "happy"}\\n\\n
```

With event listeners setup on the client:

```json
source.addEventListener('message', function(e) {
  var data = JSON.parse(e.data);
  console.log(data.msg);
}, false);

source.addEventListener('userlogon', function(e) {
  var data = JSON.parse(e.data);
  console.log('User login:' + data.username);
}, false);

source.addEventListener('update', function(e) {
  var data = JSON.parse(e.data);
  console.log(data.username + ' is now ' + data.emotion);
}, false);
```

## Cancel an Event Stream

Normally, the browser auto-reconnects to the event source when the connection is closed, but that behavior can be canceled from either the client or server.

To cancel a stream from the client, simply call:

```json
source.close();
```
To cancel a stream from the server, respond with a non "`text/event-stream`" `Content-Type` or return an HTTP status other than `200 OK` (e.g. `404 Not Found`).

Both methods will prevent the browser from re-establishing the connection.

## A Word on Security

From the WHATWG's section on `Cross-document messaging security`:

> Authors should check the origin attribute to ensure that messages are only accepted from domains that they expect to receive messages from. Otherwise, bugs in the author's message handling code could be exploited by hostile sites.
> So, as an extra level of precaution, be sure to verify e.origin in your message handler matches your app's origin:

```json
source.addEventListener('message', function(e) {
  if (e.origin != '<http://example.com'>) {
    alert('Origin was not <http://example.com'>);
    return;
  }
  ...
}, false);
```
Another good idea is to check the integrity of the data you receive:

> Furthermore, even after checking the origin attribute, authors should also check that the data in question is of the expected format....
