# What exactly is Sans IO?
==========================

This might seem like an answered question in 2026.
After all, we have a site dedicated to answering this question:
[sans-io.readthedocs.io](https://sans-io.readthedocs.io/).
But one phrase there is bugging me.

> this excludes libraries that just abstract out I/O

This parenthetical gives me pause,
because it seems to include more than is needed
to get the benefit that Sans IO seems intended to provide.
It seems that I'm missing something important
about what Sans IO is defining,
and I'm having trouble identifying
what counts as Sans IO and what does not.

I'm trying to figure out where,
between Sans IO and an IO abstraction,
Sans IO is finally lost,
to help clarify what exactly Sans IO really is.

The project
-----------

I'd like to focus on a very simple example server.
This server operates over some IO that sends a stream of bytes.
When it receives a line that contains only the word `Ping`,
it responds with a line that contains `Pong`.
It ignores all other lines.

I separate the protocol from the application.
The protocol is responsible
for taking bytes that are received
and converting them into lines for the application to consume,
and for taking lines from the application
and converting them to bytes to send back to the client.

The application is responsible for
receiving lines that may or may not be the word `Ping`
and sending lines that have only the word `Pong` in response
in the right quantity for the number of pings that were sent.

The clear parts of Sans IO are
that bytes coming from the wire should be pushed into the protocol
and bytes that need to be sent over the wire should come from the protocol.
What is unclear are the mechanisms that are acceptable
for sending and receiving bytes
and for interfacing with the application.

To simplify the examples, I only consider synchronous functions,
but these apply to asynchronous functions and generators as well.
To represent the IO, I use these functions:

```python
def recv() -> bytes: ...
def send(data: bytes) -> None: ...
```

Example 1
---------

Let's start with an example that I'm confident meets definition of Sans IO.

```python
class LineParser:
    def __init__(self):
        self.buffer = bytearray()

    def decode(self, data: bytes) -> list[str]:
        self.buffer.extend(data)

        lines = []
        while b"\n" in self.buffer:
            line, self.buffer = self.buffer.split(b"\n", 1)
            self.lines.append(line.decode("utf-8"))
        return lines

    def encode(self, lines: list[str]) -> bytes:
        return b"".join(line.encode("utf-8") + b"\n" for line in lines)

class PingResponder:
    def send(self, lines: list[str]) -> list[str]:
        return ["Pong" for line in lines if line == "Ping"]

def drive():
    parser, app = LineParser(), PingResponder()
    while True:
        input_bytes = recv()
        input_lines = parser.decode(input_bytes)  
        output_lines = app.send(input_lines)
        output_bytes = parser.encode(output_lines)
        send(output_bytes)
```

I'm going to explore other possible rules
as I look at other examples,
but I am confident of the first rule of Sans IO:

> The parser must be driven by an external IO driver.

Example 2
---------

I might like for this drive function to only be responsible
for sending and receiving bytes,
and not to be responsible for interfacing with the application.

I can do that by having a "line" application protocol
that the parser calls directly.
It won't know about the specifics of what the application does,
it only knows how it interacts with the LineParser.

```python
class LineApp(Protocol):
    def send(self, lines: list[str]) -> list[str]: ...

class LineParser[A: LineApp]:
    def __init__(self, app: A):
        self.app = app
        self.buffer = bytearray()

    def decode(self, data: bytes) -> list[str]:
        self.buffer.extend(data)

        lines = []
        while b"\n" in self.buffer:
            line, self.buffer = self.buffer.split(b"\n", 1)
            lines.append(line.decode("utf-8"))
        return lines

    def encode(self, lines: list[str]) -> bytes:
        return b"".join(line.encode("utf-8") + b"\n" for line in lines)

    def send(self, input_bytes: bytes) -> list[str]:
        input_lines = self.decode(input_bytes)
        output_lines = self.app.send(input_lines)
        return self.encode(output_lines)

class PingResponder(LineApp):
    def send(self, lines: list[str]) -> list[str]:
        return ["Pong" for line in lines if line == "Ping"]

def drive():
    parser = LineSplitParser(PingResponder())
    while True:
        input_bytes = recv()
        output_bytes = parser.send(input_bytes)
        send(output_bytes)
```

My feeling is that this approach also meets the definition of Sans IO.

Example 3
---------

I might also like to invert the authoring style of the protocol.
[ohneio](https://github.com/acatton/ohneio) has done this in its own way,
but we can do something similar by using a regular generator.
This allows us to write the parser in a pull-based fashion,
while preserving the driver's responsibility to connect to IO.

```python
LineApp = Generator[list[str], list[str], None]

def LineParser[A: LineApp](app: A) -> Generator[bytes, bytes, None]:
    buffer = bytearray()
    output = b""
    
    buffer.extend(yield bytes(output))
    while True:
        input_lines = []
        while b"\n" in input_buffer:
            line, self.buffer = self.buffer.split(b"\n", 1)
            input_lines.append(line.decode("utf-8"))

        output_lines = app.send(input_lines)
        output_buffer = b"".join(line.encode("utf-8") + b"\n" for line in output_lines)
    
        yield bytes(output_buffer)

def PingResponder() -> LineApp:
    output = []
    
    input = yield output
    while True:
        output = ["Pong" for line in input if line == "Ping"]
        input = yield output

def drive():
    app = PingResponder()
    app.send(None)  # Prime the generator
    parser = LineParser(app)
    parser.send(None)  # Prime the generator
    
    while True:
        input_bytes = recv()
        output_bytes = parser.send(input_bytes)
        send(output_bytes)
```

This has some extra fluff because we're using generators,
causing us to need to prime the generators
and send empty output before we get input,
but it allows us to think about the work more sequentially.

Like the last, I think this is Sans IO,
especially since [ohneio](https://github.com/acatton/ohneio)
is using a similar style
and is linked from the [Sans IO](https://sans-io.readthedocs.io/) site.