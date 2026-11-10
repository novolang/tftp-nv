# Changelog

All notable changes to tftp-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

The first release.

### Added

- **`tftpwire`**, the packets of TFTP with no allocation. RRQ, WRQ,
  DATA, ACK, ERROR (RFC 1350) and OACK (RFC 2347) are built into a
  `TftpPacket`, a `Vec[u8; 1028]` the caller owns, and read in place.
  The options `blksize` (RFC 2348), `timeout` and `tsize` (RFC 2349) are
  written and parsed with the ranges the RFCs give; option names and the
  mode are compared without regard to case; an option the package does not
  know, or one whose value is out of range, is skipped. `classify` checks
  that a datagram has the shape its opcode promises. `error_text` gives
  the standard text of the nine error codes.
- **`tftpxfer`**, one transfer as a step a caller drives: `client_read`,
  `client_write` and `server`; `start`, `on_packet` and `tick`; `compose`
  and `compose_request` for the packet a step asks for; `data_sent`;
  `set_limits`. The transfer is lock-step, a block shorter than the block
  size is the last, the sender drops a repeated ACK, the receiver answers
  a repeated DATA with the same ACK (also after the transfer is done),
  block numbers wrap from 65535 to 0, a timeout repeats the last packet
  up to a limit, and the options are negotiated in the way RFC 2347, 2348
  and 2349 describe.
- **`tests/embedded_probe.nv`**, a firmware program that runs a client and
  a server in one image and is built for `--target=nrf52-qemu`. It prints
  `PASS: tftp-embedded` under QEMU. The image is 22,820 bytes of flash and
  520 bytes of static RAM.
- Three test suites: `tests/tftpwire_tests.nv` (byte strings worked out
  by hand), `tests/tftpxfer_tests.nv` (the other end played by hand) and
  `tests/tftp_loop_tests.nv` (a client against a server, with packets
  lost).

### Not included

- Netascii translation. The mode is read and named; the bytes are the
  caller's.
- Block sizes above 1024 bytes. The protocol allows 65464.
- The windowsize option (RFC 7440).
- Sockets, timers and file access. The caller moves the datagrams, says
  how much time has passed and reads and writes the file.
