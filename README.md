# smp-client-nv

SMP, the Simple Management Protocol of MCUmgr, is the protocol that `mcumgr`,
nRF Connect Device Manager and `smpmgr` speak to a device that runs MCUboot,
Zephyr, the nRF Connect SDK or Apache Mynewt.  A client lists the images a
device holds, uploads a new one, marks it for a test and confirms it, and
reads and writes the files of the device's volume.  This package is the client
half of SMP, written over the server half,
[mcumgr-nv](https://novo-lang.org/packages/mcumgr-nv).  It holds a library
(`smpapi`) whose commands return values and take their transport as an
argument, a `Transport` trait with implementations for a serial port, a pipe, a
TCP socket and a script for a test, and two commands: `smp-client-nv`, a client
for a bench, and `novo-update`, which the toolchain runs as `novo update` to
update a board and write its data region.

The same calls serve a tool on a PC, a program on a board that is a Bluetooth
Central and an application on a phone: each hands the library the `Transport`
that reaches its device, and nothing else in the program changes.  Over Bluetooth
the transport is `smpgatt`, written over the `GattCentral` trait of ble-central-nv.

## What it is

A packet is an 8-byte header and a CBOR body, and a client sends one request at
a time: it builds the packet, hands it to a `Transport`, reads packets until one
carries the request's sequence number and is whole, and reads the code in it.  A
request that is not answered within the timeout is sent again, up to a number of
tries.  A command of the library answers `Ok(value)` or `Err(SmpError)`; the
error holds the code the server answered, or one of the codes of the client:

| Code | Name | Meaning |
| --- | --- | --- |
| 1 to 11 | the server's | the server refused the request (`ENOENT` is 5, `EBADSTATE` 6, `EACCESSDENIED` 11) |
| -1 | `E_NO_ANSWER` | no response arrived within the tries |
| -10 | `E_SEND` | the transport refused the request |
| -11 | `E_BAD_ANSWER` | a response had the request's number and was cut or malformed, or lacked what its command returns |
| -12 | `E_HASH` | after an upload the server holds no image with the file's hash |
| -13 | `E_CANCELLED` | the progress callback answered false |

`SmpError.message()` says it in a sentence.

## Install

```
novo pkg add smp-client-nv
```

The commands need no install: `novo update` is shipped with the toolchain
(`~/.novo/bin/novo-update` beside `novo`), and `smp-client-nv` is built with
`novo pkg build`.

## The library

`smpapi` has one function for each command.  `options(ms, tries)` says how long
a command waits for each of its attempts and how many it makes; `with_chunk`
caps the bytes of a file in one request, and the default is the most the buffer
and the transport allow.

| Call | Answers |
| --- | --- |
| `echo(t, text, o)` | `Str`, the text sent back |
| `params(t, o)` | `Params`: the buffer size and count of the server |
| `reset(t, o)`, `image_erase(t, o)`, `fs_close(t, o)` | `Unit` |
| `image_list(t, o)`, `image_test(t, hash, o)`, `image_confirm(t, hash, o)` | `[Slot]`: image, slot, version, hash and the flags bootable, active, confirmed, pending, permanent |
| `image_upload(t, file, o, p)` | `Upload`: the requests sent and the offset the first answer resumed at; then reads the list and checks that an image with the file's hash is there (`E_HASH` when none is) |
| `image_upload_to(t, image, file, o, p)` | `Upload`, with no check; image 1 is the data region |
| `fs_upload(t, name, file, o, p)`, `fs_download(t, name, o, p)` | `Unit`, `Bytes` |
| `fs_status(t, name, o)`, `fs_hash(t, name, o)` | `Int`, the length; `FileHash`, the CRC-32 and the length |

`p` is a `Progress`: a trait with one method, `step(done, total)`, called after
each response of a transfer; `Silent` ignores it, and a `step` that answers
false stops the transfer with `E_CANCELLED`.  An upload stopped that way, or cut
by a link that went, resumes when the call is made again on a new transport:
the first request is answered with the offset the server kept, and the bytes
before it are not sent.

An example over a serial port, with a progress line and a check:

```novo
use std.fs
use smpapi
use smptransport

struct Bar
    id: Int

impl Progress[io, mutate] for Bar
    fn step(self, done: Int, total: Int) -> Bool [io, mutate]
        println("${done} of ${total} bytes")
        true

fn main() [io, fs, mutate]
    match smptransport.open_serial("/dev/ttyACM0", 115200)
        None    => println("cannot open the port")
        Some(t) =>
            match fs.read_bytes("app.img")
                None    => println("no app.img")
                Some(b) =>
                    match smpapi.image_upload(t, b, smpapi.options(10000, 3), Bar { id: 0 })
                        Ok(u)  => println("sent in ${u.requests} requests, resumed at ${u.resumed_at}")
                        Err(e) => println(e.message())
```

## The transport

```novo ignore
pub trait Transport[e]
    fn send(self, packet: Bytes) -> Bool [e, mutate]
    fn recv(self, ms: Int) -> ?Bytes [e, mutate]
    fn mtu(self) -> Int        // 0 unless the transport limits a packet
```

`send` writes one packet, a header and its body, and says whether it was
written.  `recv` answers the next packet, or `None` when none arrives within
`ms` milliseconds or the link has closed.  `mtu` is the most bytes of a packet
the transport carries; the client sizes an upload's chunk by the smaller of that
and its own buffer, and the default body answers 0, no limit.  A transport that
splits a packet, as Bluetooth does, joins the pieces in `recv` and answers it
whole.  A struct is a reference, so a transport whose state changes with a
packet keeps it in a struct reached through `self` (`smpscript.nv` is an
example).

The package implements it as:

| Transport | Carries | Opened by |
| --- | --- | --- |
| `Framed` over `DevWire` | the console framing of the serial transport, over a serial port in raw mode, or a pty | `open_serial(path, baud)`, `open_dev(path)` |
| `Framed` over `TcpWire` | the same, over a TCP socket | `connect(host, port, ms)` |
| `Raw` over `PipeWire` | the packets as they are, over the standard input and output of a program | `spawn_pipe(argv)` |
| `Scripted` | canned packets, for a test | `smpscript.scripted()` |

A program with another way to carry bytes implements `Wire` (a byte at a time)
and gets `Framed` or `Raw` over it; a program with another way to carry packets
implements `Transport`.

### Serial and TCP

```novo
use smpapi
use smptransport

fn main() [io, net, mutate]
    match smptransport.connect("127.0.0.1", 5555, 3000)
        None    => println("cannot connect")
        Some(t) =>
            match smpapi.echo(t, "hello", smpapi.options(2000, 2))
                Ok(back) => println(back)
                Err(e)   => println(e.message())
```

`open_serial("/dev/ttyACM0", 115200)` is the same call for a port.  A frame
goes to a serial port in writes of at most 64 bytes, as the J-Link serial port
of the nRF52840-DK drops a longer one.

### A pipe

`spawn_pipe` starts a program and speaks to it in packets through its standard
input and output.  A relay that holds a Bluetooth Controller is such a program:

```novo
use std.bytes
use smpapi
use smptransport

fn main() [io, mutate]
    match smptransport.spawn_pipe(["ble_smp", "/tmp/novo_hci.sock", "stdio", "AA:BB:CC:DD:EE:FF"])
        None    => println("cannot start the relay")
        Some(t) =>
            match smpapi.image_confirm(t, bytes.zeros(0), smpapi.options(10000, 1))
                Ok(slots) => println("${slots.len()} images")
                Err(e)    => println(e.message())
```

### A script, for a test

`smpscript` is a transport with no device behind it.  A test queues the packets
the device would send, runs a command, and reads back what the command sent:

```novo
use smpapi
use smpscript

fn reset_is_answered(canned: Bytes) -> Bool [mutate]
    let dev = smpscript.scripted()
    dev.silence()                  // the first attempt times out
    dev.answer(canned)             // the second is answered
    match smpapi.reset(dev, smpapi.options(50, 2))
        Ok(_)  => dev.requests() == 2
        Err(_) => false
```

`answer` takes the sequence number of the request sent last, so a canned packet
need not know it; `answer_raw` delivers a packet as it is, for a late answer to
an earlier request; `silence` is a timeout; `refuse_sends(n)` makes the next `n`
writes fail; `set_limit(n)` is the `mtu`.  `smpscript.packet_of` makes a packet
from a buffer filled with the response builders of mcumgr-nv.

### Bluetooth Low Energy

The package has no radio of its own.  `smpgatt` carries SMP over a `GattCentral`, the trait of
[ble-central-nv](https://novo-lang.org/packages/ble-central-nv) that stands for the GATT operations of a Central: the noble Host on a PC, the
Bluetooth API of a phone, or the double of a test.  The dependency is on the trait package, which holds no stack.

```novo ignore
match smpgatt.open(central, peer, smpgatt.options())
    Err(e)   => println(e.message())
    Ok(link) =>
        match smpapi.image_list(link, smpapi.options(10000, 3))
            Ok(slots) => println("${slots.len()} images")
            Err(e)    => println(e.message())
```

`open` connects the peer, discovers the SMP service (`8D53DC1D-1DB7-4CD3-868B-8A527460AA84`), asks for an MTU of 517 and for the short
connection interval, and subscribes to the characteristic (`DA2E7828-FBCE-4E01-AE9E-261174997C48`: Write Without Response and Notify);
`attach` does the last two steps on a link the program connected itself.  `GattLink` is the `Transport`:

- `send` writes the packet to the characteristic without a response in pieces of the link's largest write (`max_write`), each waited for, after it
  has dropped what an earlier request left behind: the notifications that are waiting and the pieces of a packet that never ended.
- `recv` joins the notifications of one response by the length in the header of the first, whatever the pieces are: a header cut across pieces,
  a piece that ends one packet and starts the next.  It answers `None` when no piece arrives within the time, or the link has ended, and drops
  the pieces it held.  Bytes at the front that cannot be the header of a response (reserved bits set, a version the protocol does not define, an
  operation that is a request) are dropped one at a time, so a stray tail of an earlier response does not hold up the next.
- A piece that was lost, one that arrived twice and a response that came late cost one request: the packet that results is passed over by
  `smpapi`, which sends the request again.  The package cannot tell a duplicated piece from a continuation, so the repair is the retry, not the
  reassembly.

A link that ended is `lost()`, and `failure()` says why.  `smpapi` answers `E_SEND` or `E_NO_ANSWER` for a lost link, as it does for a device
that does not answer; a caller that must tell them apart asks the link.  `reopen(ms)` connects the same peer again on a new link of the same
value, which is the `Reopen` trait of `smptransport` that `smpflow` calls.  An upload that was cut resumes from the offset the device kept:

```novo ignore
if link.lost()
    if link.reopen(10000)
        let again = smpapi.image_upload(link, image, smpapi.options(10000, 3), smpapi.Silent { id: 0 })
```

The PC's adapter over the noble Host, `smp_gatt_transport` in noble-examples, and the one over a phone's Bluetooth API are the two
implementations of `GattCentral` that this module runs on; neither is in this package.

`smpdevice` is the device of a suite: the server of mcumgr-nv over RAM slots with novoboot's start-up rule (a reset swaps the slots, a trial image
that is not confirmed is reverted), reached as a `GattCentral` over the double of ble-central-nv (`periph()`), whose next response a test splits,
loses, doubles or holds back and whose link a test cuts.  A program that speaks SMP tests its flow against it with no radio and no board.

## What the package contains

| Module | Contents |
| --- | --- |
| `smpapi` | The commands as functions that return values: `CallOptions`, `Progress`, `Slot`, `SmpError` and the calls above. |
| `smpbody` | A response read as bytes: the check that a packet is whole (`valid`) and the readers of its keys, of any size and with arrays and nested maps. |
| `smptransport` | The `Wire` trait (a byte at a time), the `Transport` trait (a packet at a time), the `Reopen` trait, `Framed` and `Raw` over a wire, `TcpWire`, `DevWire` and `PipeWire`, and `connect`, `open_dev`, `open_serial` and `spawn_pipe`. |
| `smpscript` | `Scripted`, the transport of a test. |
| `smpgatt` | `GattLink`, the `Transport` over a `GattCentral`: `open`, `attach`, `reopen`, the UUIDs of the SMP service. |
| `smpdevice` | A device for a suite: the server of mcumgr-nv over RAM slots with novoboot's start-up rule, as a `GattCentral` (`periph`). |
| `smpclient` | The commands that print: the library's calls with the lines `smp-client-nv` writes, and a code in place of a `Result`. |
| `smpcli` | The commands of the `smp-client-nv` program, and the exit status. |
| `update` | `novo update`: the command line, the board's flash table as the toolchain prints it, the listing of serial ports and of devices, and the upload with its trial, restart and confirmation. |

The package that holds the server, mcumgr-nv, also holds `smpreq`, which builds
the requests, and `smppack`, which writes the package that nRF Connect Device
Manager reads; the suites of the server use both, so they stay there.  The
dependency runs from this package to mcumgr-nv: the client is written over the
server's header, CBOR and framing and its suites run against the server's
session, and the other direction would be a cycle.

## The commands

```
smp-client-nv (--tcp HOST:PORT | --serial PATH [--baud N] | --dev PATH | --pipe PROGRAM)
              [--timeout MS] [--tries N] COMMAND
```

`--tcp` and `--serial` carry the console framing over a socket and over a
serial port, which the program puts in raw mode with `stty`.  `--dev` uses a
descriptor as it is, for a pty already in raw mode.  `--pipe` starts PROGRAM and
talks to it in packets through its standard input and output.  The commands are
`echo TEXT`, `params`, `reset`, `image list`, `image upload FILE [--image N]`,
`image test HASH`, `image confirm [HASH]`, `image erase`, `fs upload FILE NAME`,
`fs download NAME FILE`, `fs status NAME`, `fs hash NAME`, `fs close`,
`pack IMAGE -o PACKAGE` and `unpack PACKAGE -o IMAGE`.  The exit status is 0
for success, 1 for a code the server answered or a file that could not be read,
2 for a usage error and 3 when no answer came.

```
novo update [--target BOARD] [--serial [PORT] | --ble [ADDRESS] | --tcp HOST:PORT] [OPTIONS] FILE
novo update [--target BOARD] [LINK] VERB [ARGUMENTS]
novo update scan [--target BOARD] [--serial | --ble]
```

A FILE alone is an upload to the second slot: a NOVB image that `novo build
--role=application` wrote (`<name>.img`), or the package that `novo image pack`
writes.  `--test` marks the new image for a trial and restarts the board;
`--confirm` then waits for the board to answer again and confirms the image that
runs; `--confirm` alone makes the staged image permanent.  The verbs are
`upload`, `data FILE` (the file `novofs-tools mk --target` wrote, into the data
region, which is image 1), `list`, `test HASH`, `confirm [HASH]`, `reset`,
`erase`, `params`, `echo TEXT` and `file up|down|status|hash|close`.  The
options are `--baud` (115200), `--timeout` (10000 milliseconds), `--tries` (3)
and `--wait` (120 seconds for the board after a restart).

With `--target` the program asks the toolchain for the board's flash table
(`novo bsp flash --target BOARD --json`) and refuses before it opens the link an
image larger than slot B and a data image of another size than the data region.
`--serial` without a port lists the serial ports of attached USB boards, those
of the board's probe first (the vendor id of its `[bsp] probe`, or the serial
number in `NOVO_PROBE`), and uses the one there is.  `--ble` without an address
scans for devices that advertise the SMP service and does the same.  `novo
update scan` is the listing alone.  The exit status is that of `smp-client-nv`,
and 3 also for a board that does not answer within `--wait` of its restart.

Bluetooth needs a program that has a Controller.  `novo update --ble` starts the
relay named by `NOVO_UPDATE_BLE`, else `novo-update-ble` beside `novo`, else the
one on PATH, with the socket of the bridge in front of the Controller
(`NOVO_HCI_SOCKET`, `/tmp/novo_hci.sock` when unset).  The relay is the program
`ble_smp` of noble-examples, built as `novo-update-ble`; it takes `scan` and
`stdio ADDRESS` after the socket.

## The rules a user needs

1. **The server takes one request at a time on a serial line**, and the client
   asks again after the timeout.  A timeout must exceed the time the server
   takes for its slowest request, the erase of the second slot at the first
   request of an upload.
2. **An upload that was cut resumes.**  The server answers the offset it
   expects next, and the client goes on from there.  mcumgr-nv writes its pages
   of 512 bytes after the header and asks again for the last page it may have
   torn, so the offset of a resume lies behind the bytes sent by up to two pages.
3. **A package and a bare image upload alike.**  The server stages the image
   and ignores the trailer.  The hash `--test` and `--confirm` name is the one in
   the image's header, which the package's trailer repeats.
4. **`--serial` sets the port with `stty`.**  A port that `stty` cannot set, or
   a path that is not a plain path, is not opened.  `--dev` does not run `stty`.
   A frame goes to the port in writes of at most 64 bytes, as the J-Link serial port of the nRF52840-DK drops a longer one.
5. **A board restarted by the update leaves the link.**  The program closes the
   link and opens it again until the board answers, so a serial port that
   disappears for a moment, or a Bluetooth connection, does not end the run.

## Limits

- One request is in flight in a process.  The buffer a request is built in and
  its sequence numbers belong to the package, so two threads must not call it at
  the same time.
- A request is at most `smp_buf_bytes` of mcumgr-nv's configuration (512 by
  default) and at most the transport's `mtu`; a response may be of any size the
  transport delivers.
- A file upload goes whole or not at all: the server gathers the file in memory
  and writes it when the last chunk has arrived, so an upload of a file that
  stopped starts again from its first byte.  An image upload resumes.
- The check of `image_upload` reads the hash of a NOVB image, or of the NOVB
  image inside a package.  A file of another format is uploaded and not
  checked; the commands of the programs do not check after an upload, and
  `test HASH` is refused by a server that holds no such image.
- There is no call that lists the files of a volume: the file group of SMP has
  no such command, and the server of mcumgr-nv offers none.  A file is named by
  its absolute path.
- The package has no radio and no UDP, and no reader of the
  statistics, log, shell or settings groups: the server answers 8 (not
  supported) to them.

## Related packages

- [mcumgr-nv](https://novo-lang.org/packages/mcumgr-nv) is the server this
  client speaks to, and holds the header, the CBOR and the console framing the
  client is written over.
- [novoboot](https://novo-lang.org/packages/novoboot) is the bootloader the
  server writes the second slot for.
- [novofs](https://novo-lang.org/packages/novofs) makes the data image that
  `novo update data` sends.

## Development

The package depends on mcumgr-nv by the registry range `^0.2.0` and on ble-central-nv, the trait of `smpgatt`, by a path to the novo-lang
tree (`../../novo/orbit/ble-central-nv`) until ble-central-nv is published.  `novo pkg publish` refuses a path, so the manifest line becomes
`ble-central-nv = "^0.1.0"` at publish, after ble-central-nv 0.1.0.

```bash
novo pkg build                                    # the smp-client-nv command
novo pkg build --bin novo-update -o novo-update   # the command novo update runs
bash tests/coverage.sh
```

## Tests

```bash
novo test tests/api_tests.nv        # 16: the library against a script (codes, a lost answer, an answer of another request, a cut packet, a hash that differs, resume, progress) and against the server in the same process
novo test tests/client_tests.nv     # 16: the commands against a server in the same process, a socket, a pipe, a descriptor, a serial port
novo test tests/gatt_tests.nv       # 18: the transport over a Central against the server in the same process (every command, writes of 20, 185 and 244 bytes, a response cut at every boundary, a lost, a doubled and a late piece, a link dropped in the middle of an upload and resumed)
novo test tests/transport_tests.nv  #  4: a packet across a pipe, the framing and the pieces of a write, the opening of a serial port
novo test tests/update_tests.nv     # 21: the command line, the board, the listing, the flow of an update
novo test tests/volume_tests.nv     #  1: a data image into the data region and the files of its volume
```

`tests/coverage.sh` merges the suites and reports the line coverage of `src/`:
100 %.  The toolchain's repository runs the commands against mcumgr-nv's host
server, over TCP and over a pty, as a gate.

## Licence

Apache-2.0.
