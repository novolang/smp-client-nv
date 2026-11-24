# Changelog

All notable changes to smp-client-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.3 — 2026-11-24

- The dependency on ble-central-nv is `^0.1.1`.  ble-central-nv 0.1.0 declared no `@tier` on its functions, and a project at the embedded tier that held smp-client-nv 0.1.2
  (`orbit/noble-examples` of the toolchain tree builds the Host on a PC against it) failed its build with E4000 on `uuid16`, `uuid_from_bytes` and `uuid_from_text`, which it
  never calls.  0.1.1 declares `@tier(app)` on its functions.  The only change in the sources is the version `novo update --version` prints.

## 0.1.2 — 2026-11-24

- `CallOptions.window` and `with_window`: an upload keeps up to `window` requests in flight (default 4, 1 is the stop-and-wait of before) and is go-back-N on the `off` of the
  responses; `Upload.resent` and `Report.resent` count the requests sent again; `Transport.keep_queued` (default: nothing) tells a transport to keep the answers that wait.
  `CallOptions` gained a field, so a program that builds one by its fields must add `window`; `options` and `with_chunk` do.  The flow's resume logic is unchanged: it reads
  the list at every step, and the first answer of an upload still says where the server resumes.

- `smpgatt`: the `Transport` over a `GattCentral` of ble-central-nv.  `open` connects a peer, discovers the SMP service, asks for the largest MTU
  and the short connection interval and subscribes; `attach` does the discovery and the subscription on a link the program connected.  `send`
  writes a packet to the characteristic without a response in pieces of the link's largest write and first drops the notifications that wait and
  the pieces of an unfinished packet; `recv` joins the notifications of a response by the length in its header, drops bytes at the front that cannot
  start a response, forgets a packet that stopped, and answers `None` on a timeout and on a link that ended.  `GattLink.lost`, `failure` and
  `stats` say what happened; `reopen` connects the same peer again on a new link.
- `Reopen`, a trait in `smptransport`: `lost()` and `reopen(ms)` for a transport whose link ends and is made again.  `GattLink` implements it.
- `smpdevice`: the server of mcumgr-nv over RAM slots with novoboot's start-up rule, for the suites of a program that speaks SMP, as a `GattCentral`
  over the double of ble-central-nv whose responses a test splits, loses, doubles and holds back and whose link a test cuts.
- A dependency on ble-central-nv ^0.1.0, written as a path to the novo-lang tree until that package is published; the line is `"^0.1.0"` at publish.
- `smpflow`: the upgrade of a device that runs novoboot and the SMP service as a state machine over a `Transport` that is also a `Reopen`.  `run` reads the
  image list, checks the candidate (not an image, larger than the slot the plan states, not newer when the plan asks), uploads with progress and resume,
  marks the image (`image test`, or `image confirm HASH` for a permanent image), restarts the device, waits for it with growing pauses, reads the list
  again and confirms, and makes the link again when it dropped.  `decide` is the table of the stages: the next step comes from the list and from the one fact
  the list cannot show, whether a restart was asked for, which `FlowState` holds (`new_state`, `state_text`, `state_parse`, `Journal`).  `Mode`
  (`TestAndConfirm`, `ConfirmOnly`, `TestOnly`), `Plan`, `Report` and `Outcome` (`Done`, `AlreadyInstalled`, `Reverted`, `NeverReturned`, `Refused`, `Cancelled`,
  `LinkLost`, `Failed`).  `Clock` with `System` and `Manual`.
- `smpdirect`: the device of `smpdevice` as a `Transport` and a `Reopen` in the same process, cut at a request.
- `tests/gatt_tests.nv`, 18 tests; `tests/flow_tests.nv`, 21 tests; `tests/proof_tests.nv`, 8 tests, the flow over `smpgatt` against the real server of mcumgr-nv.
- `tests/update_tests.nv` no longer imports `std.env`, which it did not use.

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
