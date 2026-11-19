# lsp

The Language Server Protocol for Avin: JSON-RPC 2.0 over `Content-Length` framing, the lifecycle,
text documents kept in step with the editor, positions in UTF-16 or UTF-8, and typed handlers for
diagnostics, hover, completion, definitions, references, document symbols, formatting, rename and
semantic tokens, each request answered on a lightweight thread of its own.

The package's module is **`dev.avinlang.lsp`**, written in Avin. A program depends on it in its
`avin.hocon`:

```hocon
dependencies {
  lsp { git = "github.com/avin-lang/lsp", version = "0.0.1" }
}
```

It depends on [`json`](https://github.com/avin-lang/json), whose derived encoders and decoders read
and write every protocol type.

## A server

A server for a toy settings language, whose lines are `name = value`: a line with no `=` is an
error, published as a diagnostic, and a hover shows the line under the cursor.

```avin
import dev.avinlang.lsp.{
  Diagnostic, DiagnosticSeverity, Hover, MarkupContent, Position, PositionEncoding, Range, Server, TextDocument
}

// Every line that is not blank and has no `=`, as an error.
check(doc: TextDocument, encoding: PositionEncoding) -> Vector[Diagnostic] = (0 ..< doc.lineCount).toVector.flatMap(line ->
  val text = doc.lineText(line)
  if text.isBlank || text.contains("=") then None
  else
    val start = doc.offsetAt(Position(line, 0), encoding)
    val whole = Range(Position(line, 0), doc.positionAt(start + text.byteLength, encoding))
    Some(Diagnostic(whole, "expected `name = value`", Some(DiagnosticSeverity.Error))))

server() -> Server = Server("settings", "0.1")
  .onDiagnostics((ctx, doc) -> check(doc, ctx.encoding))
  .onHover((ctx, p) -> Ok(ctx.document(p.textDocument.uri).map(doc ->
    Hover(MarkupContent.plain(s"line ${p.position.line + 1}: ${doc.lineText(p.position.line)}")))))

main(args: Vector[string]) -> int = server().serveArgs(args)
```

`examples/words` is a server answering every kind of request, built and driven by the tests.

## Handlers

Each handler takes the `Context` and the request's params, typed, and answers a `Result` whose `Err`
is the `ResponseError` the request is answered with. Registering a handler announces its capability
at `initialize`; a method with no handler is answered `MethodNotFound`, and params that do not read as
the handler's type `InvalidParams`, naming what is wrong.

| Register | Params | Result |
| --- | --- | --- |
| `onDiagnostics(find)` | the `TextDocument`, on each open and change | `Vector[Diagnostic]`, published |
| `onHover` | `TextDocumentPositionParams` | `Option[Hover]` |
| `onCompletion(h, triggerCharacters)` | `CompletionParams` | `CompletionList` |
| `onDefinition` | `TextDocumentPositionParams` | `Vector[Location]` |
| `onReferences` | `ReferenceParams` | `Vector[Location]` |
| `onDocumentSymbols` | `DocumentSymbolParams` | `Vector[DocumentSymbol]` |
| `onFormatting` | `DocumentFormattingParams` | `Vector[TextEdit]` |
| `onRename` | `RenameParams` | `Option[WorkspaceEdit]` |
| `onSemanticTokens(legend, h)` | `SemanticTokensParams` | `SemanticTokens` |
| `onRequest[P, R](method, h)` | any `P` with a `JsonDecoder` | any `R` with a `JsonEncoder` |
| `onNotification[P](method, h)` | any `P` with a `JsonDecoder` | nothing |

A `Context` gives the documents open when the message arrived (`document(uri)`), the `encoding`
agreed, what the client said at `initialize` (`client`), whether the request has been cancelled
(`isCancelled`), and `notify`, `log`, `publishDiagnostics`, and `beginProgress`, `reportProgress`
and `endProgress`, which report on the request's `workDoneToken`. `semanticTokens(tokens)` encodes
tokens placed absolutely as the protocol sends them, each relative to the one before.

## What the server does

- **Framing.** A message is a header block, each line ended by `\r\n`, a blank line and a body of
  `Content-Length` bytes. Header names are matched without regard to case and every other header,
  `Content-Type` among them, is passed over. A missing or bad `Content-Length`, a header block past
  8 KiB, a body past `maxMessageBytes` (64 MiB unless given) and input ending inside a message end
  the server with status 1, since past them there is no telling where the next message starts. A
  body that is not UTF-8 is answered `ParseError`, and the next message is read.
- **JSON-RPC.** Text that is not JSON is `ParseError` (-32700); JSON that is not a request,
  `"jsonrpc"` other than `"2.0"`, or an id that is neither a whole number nor a string is
  `InvalidRequest` (-32600); a batch is refused with one `InvalidRequest`. An id is written back as it
  came, a number as a number and a string as a string. A notification is never answered, an unknown
  one included.
- **Concurrency.** Each request runs on a lightweight thread of its own, so a quick request is
  answered while a slow one works, and responses go out in the order they are ready. One writer
  thread writes every message, whole. Notifications are handled in order, before anything read after
  them: a document's change is in place before the request after it is handled.
- **Cancellation.** `$/cancelRequest` answers the request `RequestCancelled` (-32800) at once, unless
  its handler answered first; the handler's `isCancelled` turns true, and what it answers after is
  dropped. A cancellation of an id not being answered is passed over.
- **Lifecycle.** A request before `initialize` is `ServerNotInitialized` (-32002), and a notification
  then is dropped; `initialize` a second time, and any request after `shutdown`, is `InvalidRequest`.
  `exit` ends the server with status 0 after `shutdown` and 1 without.
- **Documents.** `didOpen`, `didChange`, full or incremental (the server announces incremental), and
  `didClose`. A change's range is placed as positions are: a line past the last is the end of the
  text, a character past a line's end is its end, before `\r\n`, and a range whose end comes first
  replaces nothing. Diagnostics found for a version that has since changed are dropped, so they never
  arrive out of order; a closed document's are cleared.
- **Positions.** A character counts UTF-16 code units, the protocol's default, unless the client
  offers `utf-8` in `general.positionEncodings`, which is then used. `TextDocument.offsetAt` and
  `positionAt` convert between positions and the byte offsets of the text; a position between the
  two halves of a surrogate pair is the character's start.

## Connections

`serve(input, output)` serves any `io.Reader` and `io.ByteWriter`. `serveArgs(args)` connects the way
an editor starts a server, then serves: `--pipe=<path>` connects to the Unix domain socket the client
listens on, and `--socket=<port>` to the TCP port on the local host. Standard input and output are not
a connection it serves: the standard library reads standard input only as text, and a body's length
is counted in bytes.

## Testing a server

`Client` is the editor's side, for tests and tools: `initialize`, `request`, `notify`, `call` (a
request and its response), `notification(method)` and `shutdown`, with each `Message`'s `result[R]()`
and `params[P]()` read through the derived decoders.

## Tests

`avin test` in this directory runs the package's tests, in `tests/`; `tests/readme.avin` holds this
page's example, and `tests/e2e.avin` builds `examples/words` and drives it as a child process.
