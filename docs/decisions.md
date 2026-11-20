# Design decisions

Each entry was settled when the package was written, 2026-11-19. Entries marked **Review with the
user** are the debatable ones.

## A server is registered handlers, typed by their params and result

`Server(name, version)` takes one handler per method: `onHover`, `onCompletion` and the rest for the
protocol's methods, and `onRequest[P, R]` and `onNotification[P]` for any other. A handler is a
`@sendable (Context, P) -> Result[R, ResponseError]`, its params read by `P`'s derived decoder and its
result written by `R`'s encoder; inside, the types are erased into a function of the message's text,
so the table is one map. The alternative, a trait with a method per request, makes a server
implement every method, and makes the capabilities a second list to keep in step; here the
capabilities announced are those of the handlers registered.

`Server` is a builder: each `on…` registers and answers the same server, so registrations chain.
**Review with the user**: an immutable `Server` copied by each registration would be sendable, so
one value could serve several connections; the builder is the smaller API for a server that is made
once in `main`.

## Results are typed to one shape per method

Where the protocol allows several shapes, the handler answers one: a hover's contents are
`MarkupContent`, a completion is a `CompletionList`, a definition a `Vector[Location]`, and a rename
an `Option[WorkspaceEdit]` of `changes` by document. Every client reads these, and one shape keeps
the handler types plain. A program needing another shape registers the method with `onRequest`.

## Optional fields are left out, not written `null`

Every result and notification is encoded with `omitNone`, since the specification's optional
properties are absent rather than null. A result that is `None` as a whole, a hover with nothing to
show, is still `null`, as the specification says.

## A message is parsed once, and its params decoded from the tree

A message's text is parsed once into a `Json` value; text that does not parse is `ParseError`. Its
top (`jsonrpc`, `id`, `method`) is decoded from that value, and the handler's params are decoded from
the same value by the handler's type (`json.decode[A](Json)`, json 0.0.3), so the text is read once
whatever the method. The params are decoded as the message's `params` member rather than from the
member alone, so an error names the path from the message, `params.textDocument.uri`, and a message
with no `params` is `InvalidParams` from the same decoder, as when they were read from the text.

## Each request runs on a thread of its own, and notifications run in order

A request is handed to a new lightweight thread, so a slow handler holds nothing up and responses go
out as they are ready; one writer thread frames and writes every message, so messages never
interleave. Notifications run on the reading thread, in order, before anything after them is read: a
document change is in place before a request after it is handled, and each request is handed the
documents as they were when it arrived. There is no limit on requests in flight. **Review with the
user**: a bound (a pool) would cap a flood of requests, at the cost of a slow handler holding up the
others.

## Cancellation answers at once

`$/cancelRequest` answers the request `RequestCancelled` as soon as it arrives, unless the handler
answered first; whichever comes first is the only answer, decided by one compare-and-set. The handler
is not stopped: it sees `isCancelled` and stops itself, and what it answers after is dropped. A
lightweight thread can be stopped only through a scope, and stopping one mid-handler would leave what
it was building half made. A cancellation of an id not being answered is passed over, as the
specification asks of a notification.

## A request reusing an id being answered is refused

A second request with the id of one still being answered is `InvalidRequest`, since its answer could
not be told from the first's. Once the first is answered the id may be used again.

## Batches are refused

LSP never sends a JSON-RPC batch, so a body that is an array is answered with one `InvalidRequest`
(id `null`) and nothing in it is run.

## An id of null is no id

`"id": null` reads as no id, so the message is a notification. The protocol's ids are numbers or
strings, and a number with a fraction is `InvalidRequest`.

## Framing faults end the server

A missing or unreadable `Content-Length`, a header line with no colon, a header block past 8 KiB, a
body past the reader's limit (64 MiB unless given) and input ending inside a message end `serve` with
status 1, said on standard error: past them there is no telling where the next message starts. A
body that is not UTF-8 is answered `ParseError`, since its length was read and the next message's
start is known.

## UTF-8 positions where offered

At `initialize` the server takes `utf-8` where the client lists it in `general.positionEncodings`,
since an Avin string is UTF-8 and its offsets then need no counting, and UTF-16, the protocol's
default, otherwise. UTF-32 is not offered.

## Positions out of range are clamped

A line past the last is the end of the text, a character past its line's end is the line's end
before its terminator, and a character between the halves of a surrogate pair is the character's
start. A change's range is placed the same way, so an edit past the end appends, and a range whose
end is before its start replaces nothing. VS Code clamps the same way; refusing such an edit would
leave the server's copy out of step with the editor's for good.

## Diagnostics of an old version are dropped

`onDiagnostics` runs on a thread of its own after each open and change, and publishes only if the
document is still at the version it read; the check and the send happen holding the documents' lock,
so a later version's diagnostics are never followed by an earlier one's. Closing a document publishes
an empty list for it.

## Connections: standard input and output, a pipe or a socket

`serve(input, output)` serves any `Reader` and any sendable `ByteWriter`. `serveArgs` connects as
editors start servers: `--stdio`, the process's own standard input and output through `avin.io`'s
byte streams `stdin()` and `stdout()` (Avin 0.0.35), `--pipe=<path>`, a Unix domain socket the
client listens on, and `--socket=<port>` (or `--port`), a TCP port on the local host. With no
connection named it serves standard input and output, since that is how most editors start a server
and a server started by hand from a terminal is then usable at once. The last connection named
counts, as a later option overrides an earlier one. Over standard input the server ends at the
input's end with status 1, as for `exit` without `shutdown`: the editor has gone.

The tests drive a child server over standard input and output with `cat` feeding its input from a
FIFO the test writes, so the input is a pipe as an editor's is, and with `bash` connecting its
output to a TCP port the test listens on, since `avin.process` gives a started command's streams
only as files or captured text, and a `File` read fills its whole request before answering.

## The progress a request reports goes on its own token

`beginProgress`, `reportProgress` and `endProgress` send `$/progress` on the `workDoneToken` the
request's params carry, and nothing where they carry none. Progress the server starts itself, through
`window/workDoneProgress/create`, needs the server to send requests, which it does not.

## The server sends no requests

The server only answers and notifies; a response from the client, a message with an id and a result
or an error but no method, is passed over. Server-to-client requests (`workspace/configuration`,
`window/showMessageRequest`, dynamic registration) would need a table of the server's own pending
requests, and nothing here needs them.

## Kinds are numbers with names

`SymbolKind`, `CompletionItemKind` and `DiagnosticSeverity` are data types of one number with the
specification's values as named constants (`SymbolKind.Function`), written as the number. The
specification's kinds are open to new values, which a closed set of variants could not read.
