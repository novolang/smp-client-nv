# Changelog

All notable changes to smp-client-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-11-22

- `smpapi`: the commands of the OS, image and file groups as functions that return values.  Each takes a `Transport` and `CallOptions` (the time to wait
  for an attempt, the number of attempts, the cap on the bytes of a file in a request) and answers `Result<T, SmpError>`: `echo`, `params`,
  `reset`, `image_list`, `image_test`, `image_confirm`, `image_erase`, `image_upload` and `image_upload_to`, `fs_upload`, `fs_download`,
  `fs_status`, `fs_hash` and `fs_close`.  `SmpError` holds the server's code or one of `E_NO_ANSWER` (-1), `E_SEND` (-10), `E_BAD_ANSWER` (-11),
  `E_HASH` (-12) and `E_CANCELLED` (-13), and says it in a sentence.
- `Progress`, a trait with one method, `step(done, total)`, which `image_upload`, `image_upload_to`, `fs_upload` and `fs_download` call after each
  response; an answer of false stops the transfer with `E_CANCELLED`.  `Silent` is the progress that ignores it.  An upload that was stopped, or cut
  by a link that went, resumes when it is called again on a new transport: the first answer is the offset the server kept.  `Upload` says how
  many requests were sent and the offset the first answer resumed at.
- `image_upload` reads the image list after the last request and answers `E_HASH` when no image has the hash the file carries (the 32 bytes at 0x20
  of a NOVB image, or of the NOVB image in a package; a file of another format is not checked).  The first request of an upload of the firmware
  carries that hash under `sha`.
- `Transport.mtu`, with a default body that answers 0 for no limit: the most bytes of a packet the transport carries.  The chunk of an upload is the
  smaller of what the buffer of the configuration and the limit allow.  A program that implements `Transport` needs no change.
- `smpscript`: `Scripted`, a `Transport` with no device behind it, for the tests of a program that speaks SMP.  A test queues answers (`answer`,
  which takes the sequence number of the request sent last, `answer_raw`, `silence`), makes writes fail (`refuse_sends`), sets the limit
  (`set_limit`) and reads back what was sent (`requests`, `request`).
- `smpbody`: a response read as bytes.  `valid` accepts a packet whose header the protocol defines, whose body is the length the header states and
  whose body is one definite map whose every item ends inside the packet; the readers take keys from a map of any size, with arrays, nested maps
  and tags before the key.  A response is no longer bound by the buffer of the configuration, and a packet that is cut is never read past its
  end.
- A response with the request's number that is not whole is passed over, the request is sent again, and a request that got only such responses is
  `E_BAD_ANSWER`; a request that got nothing is `E_NO_ANSWER`, and one the transport refused is `E_SEND`.
- `smpclient` and the `smp-client-nv` and `novo-update` programs call `smpapi` and print what it returns, with the signatures they had.  Three
  results differ: a refused write is -10 where it was -1, a response that lacks what the command prints is -11 where the command printed nothing
  and answered 0 (an image list with no `images` key is still an empty list), and an upload or a file upload whose server asks for an offset past
  the end of the file is -11 where the loop ended with 0.  The exit status of a negative code is still 3, and the message says which.  The line
  "server asks for offset" is no longer printed.
- The sources and the README name no path of the novo repository.
- Five suites, 58 tests, at 100 % of the lines of `src/` (tests/coverage.sh).

## 0.1.0 — 2026-11-15

- The client half of mcumgr-nv 0.1.3, as a package of its own: mcumgr-nv 0.2.0 is the server and this package depends on it.  `smpclient` and
  `smpcli` came across with their suites; `smpreq` and `smppack` stayed in mcumgr-nv, which builds its own requests and packages with them.
- `smptransport`: the `Transport` trait, a packet at a time (`send`, and `recv` with a timeout that answers `None`), beside the `Wire` trait of
  a byte at a time.  `Framed` carries the console framing of the serial transport over a wire, and `Raw` carries the packets as they are,
  which a reader splits by the length in the header.  `TcpWire`, `DevWire` and `PipeWire` are the wires of a socket, a descriptor and the two
  ends of a child process; `connect`, `open_dev`, `open_serial` (the port in raw mode at a baud rate, set by `stty`) and `spawn_pipe` open them.
- `smpclient`'s commands take a `Transport` in place of a `Wire`.  An answer larger than the buffer is passed over, not cut.
- The `smp-client-nv` command is the `mcumgr-nv` command of 0.1.3 under its new name, with `--serial PATH [--baud N]` and `--pipe PROGRAM`
  beside `--tcp` and `--dev`.
- `update` and the `novo-update` command, which the toolchain runs as `novo update`: `novo update [--target BOARD] [--serial [PORT] | --ble
  [ADDRESS] | --tcp HOST:PORT] [OPTIONS] FILE` and the verbs `upload`, `data`, `list`, `test`, `confirm`, `reset`, `erase`, `params`, `echo`,
  `file` and `scan`.  The board's flash table is read from `novo bsp flash --target BOARD --json`; an image larger than slot B and a data image
  of another size than the data region are refused before the link is opened.  `--serial` and `--ble` without an address list the serial ports
  of attached USB boards and the devices that advertise the SMP service, and take the one there is.  `--test` and `--confirm` run the upload,
  the trial, the restart, the wait for the board and the confirmation.
- The wait for the board after the restart of `--test --confirm` lasts `--wait` seconds (120 by default, 30 first: the swap of a 340 KB image takes 63 s on the
  nRF54LM20-DK), counted on the clock, and each attempt may take the `--timeout` (10 s; an attempt of one second never saw an answer over Bluetooth, where
  the scan for the device and the connection come before the first request).
- `DevWire.send` writes a frame to its descriptor in pieces of at most 64 bytes, each a `write` of its own with no pause between them.  The
  nRF52840-DK's J-Link serial port drops a single write of more than one full-speed USB bulk packet, so every request longer than 64 bytes got no
  answer there (a bug the novo-lang tracker holds for the board).
  The pieces are bytes of one frame and the console framing is unchanged.  `write_pieces` and the `Sink` trait it writes to are public, so a test
  can record each piece.
- Four suites, 42 tests, at 100 % of the lines of `src/` (tests/coverage.sh).
