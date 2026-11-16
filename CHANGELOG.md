# Changelog

All notable changes to smp-client-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
- Four suites, 42 tests, at 100 % of the lines of `src/` (tests/coverage.sh).
