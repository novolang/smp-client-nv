# smp-client-nv

SMP, the Simple Management Protocol of MCUmgr, is the protocol that `mcumgr`,
nRF Connect Device Manager and `smpmgr` speak to a device that runs MCUboot,
Zephyr, the nRF Connect SDK or Apache Mynewt.  A client lists the images a
device holds, uploads a new one, marks it for a test and confirms it, and
reads and writes the files of the device's volume.  This package is the client
half of SMP for a host, written over the server half,
[mcumgr-nv](https://novo-lang.org/packages/mcumgr-nv).  It holds a `Transport`
trait with implementations for a serial port, a pipe and a TCP socket, the
requests and the readings of the OS, image and file groups, and two commands:
`smp-client-nv`, a client for a bench, and `novo-update`, which the toolchain
runs as `novo update` to update a board and write its data region.

## What it is

A packet is an 8-byte header and a CBOR body, and a client sends one request at
a time: it builds the packet, hands it to a `Transport`, reads packets until one
carries the request's sequence number, and reads the code in it.  A request that
is not answered within the timeout is sent again, up to a number of tries.
Every command answers a code: 0 for success, the server's code for a request it
refused, and -1 when no answer arrived.

What differs between the ways to reach a server is how a packet is carried.
Two ways are in the package:

- **The console framing** of the serial transport, in which a packet is split
  into base64 lines of at most 124 characters with a length and a CRC-16.  A
  serial port, the pty of one, and a TCP socket carry it.
- **The packet as it is**, one after the other, which a reader splits by the
  length in the header.  A pipe to a relay carries it.  The relay of
  noble-examples (`pc/ble_smp.nv`) is the one that puts the packets on the SMP
  service of a device over Bluetooth Low Energy; the package has no Bluetooth
  of its own.

## Install

```
novo pkg add smp-client-nv
```

The commands need no install: `novo update` is shipped with the toolchain
(`~/.novo/bin/novo-update` beside `novo`), and `smp-client-nv` is built with
`novo pkg build`.

## Example

A program that lists the images of a device on a serial port, uploads a new
one and marks it for a test:

```novo
use std.fs
use smpclient
use smptransport

fn main() [io, fs, mutate]
    // The port is put in raw mode at 115200 bits a second.
    match smptransport.open_serial("/dev/ttyACM0", 115200)
        None    => println("cannot open the port")
        Some(t) =>
            // A command waits 10 seconds for an answer and asks 3 times.
            let rc = smpclient.image_list(t, 10000, 3)
            println("image list answered ${rc}")
            match fs.read_bytes("app.img")
                None    => println("no app.img")
                Some(b) => println("upload answered ${smpclient.image_upload(t, b, 0, 10000, 3)}")
```

The commands do the same from a shell, and `novo update` adds what a board needs
(its serial port found, the checks against its flash table, the restart waited
for):

```
novo update --target=nrf52840-dk --serial app.img --test --confirm
```

## What the package contains

| Module | Contents |
| --- | --- |
| `smptransport` | The `Wire` trait (a byte at a time), the `Transport` trait (a packet at a time), `Framed` and `Raw` over a wire, `TcpWire`, `DevWire` and `PipeWire`, and `connect`, `open_dev`, `open_serial` and `spawn_pipe`. |
| `smpclient` | The commands over a `Transport`: echo, parameters, reset, the image list, upload, test, confirm and erase, and the files' upload, download, status, hash and close. |
| `smpcli` | The commands of the `smp-client-nv` program, and the exit status. |
| `update` | `novo update`: the command line, the board's flash table as the toolchain prints it, the listing of serial ports and of devices, and the upload with its trial, restart and confirmation. |

The package that holds the server, mcumgr-nv, also holds `smpreq`, which builds
the requests and reads the responses, and `smppack`, which writes the package
that nRF Connect Device Manager reads; the suites of the server use both, so
they stay there.

## How to choose an entry point

A program that talks to a device calls `smpclient` with a `Transport`.  A
program that carries SMP some other way implements `Transport` (`send` writes a
packet, `recv` answers the next one or `None` after a timeout) and passes it to
the same calls.  A person at a terminal runs `smp-client-nv` for the requests as
they are, or `novo update` for the update of a board.

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
(`NOVO_HCI_SOCKET`, `/tmp/novo_hci.sock` when unset).  The relay is `pc/ble_smp.nv`
of noble-examples, built as `novo-update-ble`; it takes `scan` and `stdio
ADDRESS` after the socket.

## The rules a user needs

1. **The server takes one request at a time on a serial line**, and the client
   asks again after the timeout.  A timeout must exceed the time the server
   takes for its slowest request, the erase of the second slot at the first
   request of an upload.
2. **An upload that was cut resumes.**  The server answers the offset it
   expects next, and the client goes on from there.
3. **A package and a bare image upload alike.**  The server stages the image
   and ignores the trailer.  The hash `--test` and `--confirm` name is the one in
   the image's header, which the package's trailer repeats.
4. **`--serial` sets the port with `stty`.**  A port that `stty` cannot set, or
   a path that is not a plain path, is not opened.  `--dev` does not run `stty`.
   A frame goes to the port in writes of at most 64 bytes, as the J-Link serial port of the nRF52840-DK drops a longer one.
5. **A board restarted by the update leaves the link.**  The program closes the
   link and opens it again until the board answers, so a serial port that
   disappears for a moment, or a Bluetooth connection, does not end the run.

## What is not included

- Bluetooth Low Energy.  The package has no Controller; the relay of
  noble-examples carries the packets.
- UDP, and the groups other than the OS, image and file groups: the server
  answers 8 (not supported) to them.
- A reader of the statistics, log or shell groups.

## Related packages

- [mcumgr-nv](https://novo-lang.org/packages/mcumgr-nv) is the server this
  client speaks to, and holds the header, the CBOR and the console framing the
  client is written over.
- [novoboot](https://novo-lang.org/packages/novoboot) is the bootloader the
  server writes the second slot for.
- [novofs](https://novo-lang.org/packages/novofs) makes the data image that
  `novo update data` sends.

## Development

The package depends on mcumgr-nv by a path (`../mcumgr-nv`) until mcumgr-nv 0.2.0
is published.

```bash
novo pkg build                                    # the smp-client-nv command
novo pkg build --bin novo-update -o novo-update   # the command novo update runs
bash tests/coverage.sh
```

## Tests

```bash
novo test tests/client_tests.nv     # 16: the commands against a server in the same process, a socket, a pipe, a descriptor, a serial port
novo test tests/transport_tests.nv  #  4: a packet across a pipe, the framing and the pieces of a write, the opening of a serial port
novo test tests/update_tests.nv     # 21: the command line, the board, the listing, the flow of an update
novo test tests/volume_tests.nv     #  1: a data image into the data region and the files of its volume
```

`tests/coverage.sh` merges the suites and reports the line coverage of `src/`.
The gate in the novo repository (`tests/orbit/test_smp_client_nv.sh`) runs the
commands against mcumgr-nv's host server, over TCP and over a pty.

## Licence

Apache-2.0.
