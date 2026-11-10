# tftp-nv

TFTP, the Trivial File Transfer Protocol, moves one file between two
machines over UDP, one block at a time. It is what a microcontroller's
bootloader uses to fetch a firmware image from a server on the local
network, because it fits in a few hundred bytes of code and needs no
connection setup. This package brings TFTP to novo-lang: the packets of
[RFC 1350](https://www.rfc-editor.org/rfc/rfc1350), the option
negotiation of RFC 2347, 2348 and 2349, and the transfer itself as a
state machine that a program steps.

## What it is

The package has no socket and no clock. A program that uses it owns the
UDP socket, or the radio, or whatever moves a datagram, and it owns the
timer. The package decides what to send next.

Three things follow from that, and each is the reason for a design
choice in this package.

**The transfer is a step.** A `TftpXfer` is a small value that the
caller holds in a `var`. The caller gives it a datagram with
`on_packet`, or the passage of time with `tick`, and it answers with a
`TftpStep` that says which packet to send, whether a block of the file
has just arrived, and whether the transfer is over. The same value works
for a client and for a server, for a read and for a write.

**Nothing is allocated.** A packet is a `Vec[u8; 1028]` in the caller's
frame, and a transfer is a `@value` of 160 bytes. The transfer does not
hold a copy of the file. When a block has to be sent again the step
says so, and the caller reads the same block of the file again. That is
what lets a bootloader keep the image in flash and the transfer in a
few hundred bytes of RAM.

**A transfer can be tested without a network.** A test or a program can
run a client and a server against each other in one function, passing
the datagrams from one to the other and losing the ones it chooses to.
`tests/tftp_loop_tests.nv` does that for every length of file that
matters and for a lost packet at every position.

## Install

```
novo pkg add tftp-nv
```

## Example

This is the client side of a read, with the server played by the other
half of the package in the same program, so that it runs anywhere. On a
device the two `Vec` packets are the datagram buffers, and the
`on_packet` call is made when the UDP stack hands one over.

```novo
use tftpwire
use tftpxfer

// The server's file: five bytes, which is one short block.
fn serve_block(var out: tftpwire.TftpPacket) -> Int
    for b in [10, 20, 30, 40, 50]
        let _ = tftpwire.push_data(var out, b)
    5

fn main() [io]
    var client = tftpxfer.client_read(-1, -1, -1)
    var server = tftpxfer.client_read(-1, -1, -1)
    var to_server: tftpwire.TftpPacket = Vec.new()
    var to_client: tftpwire.TftpPacket = Vec.new()

    // 1. The client's first packet is the request.
    let ask = tftpxfer.start(var client)
    let _ = tftpxfer.compose_request(var client, var to_server, "fw.bin", "octet")

    // 2. The server reads the request and answers with DATA 1.
    server = tftpxfer.server(tftpwire.parse_request(var to_server), 512, 5)
    let reply = tftpxfer.start(var server)
    let _ = tftpxfer.compose(var server, reply, var to_client)
    let n = serve_block(var to_client)
    tftpxfer.data_sent(var server, n)

    // 3. The client takes the block and acknowledges it.
    let got = tftpxfer.on_packet(var client, var to_client)
    println("${ask.action} ${reply.action} ${got.action}")   // 1 3 2
    println("bytes ${got.len}, first ${to_client[4]}, done ${got.done}")   // bytes 5, first 10, done true
```

Build and test with `novo pkg build` and `novo test tests`.

## What the package contains

| Module | Contents |
| --- | --- |
| `tftpwire` | The six packets (RRQ, WRQ, DATA, ACK, ERROR and the OACK of RFC 2347) built into a caller's buffer and read in place, the three options `blksize`, `timeout` and `tsize`, the error codes and their standard text, and `classify`, which says whether a datagram has the shape its opcode promises. |
| `tftpxfer` | One transfer as a step: a client that reads, a client that writes, and a server for a request it has parsed; `start`, `on_packet` and `tick`; the packets a step asks for; the option negotiation; the retransmission and the limit on it. |

Every `pub` declaration is documented where it is declared, and
[the package's page](https://novo-lang.org/packages/tftp-nv) is that
reference.

## The loop a caller writes

1. `start` answers the first packet to send. For a client it is the
   request, and `compose_request` builds it. For a server it is the OACK,
   the first DATA or ACK 0, depending on the request.
2. Each datagram that arrives from the peer goes to `on_packet`.
3. When time passes with nothing arriving, `tick` is told how many
   milliseconds. It answers whether to send the last packet again.
4. Every answer is a `TftpStep`. `compose` builds the packet its `action`
   asks for, and the caller sends it.

Two answers need more than a packet. A step with `deliver` set means the
DATA packet that just arrived holds `len` bytes of the file, from index 4,
and the caller must take them before it reads the next datagram. A step
whose action is `TFTP_ACT_DATA` means the caller reads block `block` of
the file, appends it to the packet that `compose` started, and tells
`data_sent` how many bytes it appended.

## The rules a user needs

1. **The caller checks the ports.** RFC 1350 identifies a transfer by
   the client's port and the server's. The package never sees a port. A
   datagram from any other port is not part of the transfer, and the RFC
   asks for an ERROR 5 back to its sender; `tftpwire.build_error` builds
   it. A request sent again to the well-known port is the listener's to
   drop; handed to a running transfer, it ends it.
2. **A block shorter than the block size is the last.** A file whose
   size is a multiple of the block size ends with a block of no bytes.
   `data_sent` is how the transfer learns which block that is, and it
   has to be called after every DATA packet, including one sent again.
3. **The block size is 512 until an OACK says otherwise.** A server that
   does not know options answers a request that carried them as if it
   had not, and the transfer carries on at 512. A client takes an OACK
   that asks for a smaller block than it did, or the same, and refuses
   one that asks for more, a different timeout or an option it did not
   send, with ERROR 8.
4. **A block is at most 1024 bytes here.** The protocol allows 65464. A
   server built on this package answers a larger `blksize` with a smaller
   one in its OACK, which RFC 2348 permits, and a client that asks for
   more than 1024 is answered with 1024 or less.
5. **Block numbers wrap.** They are two bytes, and after 65535 comes 0.
   That is what implementations do for a file of more than 65535 blocks,
   32 MB at the default size, and RFC 1350 is silent about it.
6. **The sender does not answer a repeated ACK.** An ACK for a block
   that is not the one in flight is dropped, so a late acknowledgement
   does not start a second stream of DATA. This is the
   "sorcerer's apprentice" of RFC 1123.
7. **The receiver answers a repeated DATA with the same ACK.** It does so
   after the transfer is done as well, which is how a final ACK that was
   lost is recovered.
8. **The timer repeats the last packet.** `tick` accumulates milliseconds
   and, once the wait is over, answers the step it answered last. After
   five repeats with no answer (`set_limits` changes the number) the
   transfer fails with `TFTP_FAIL_TIMEOUT`. Any datagram that moves the
   transfer on resets the count; one that is dropped does not restart the
   wait. A `timeout` option the peer agrees to sets the wait.
9. **A fixed-capacity buffer can refuse.** A builder answers `false`, and
   leaves the buffer empty, when the packet does not fit. A request with
   a filename over about a thousand bytes is the case.
10. **Netascii is not translated.** The mode is read and named; the
    bytes of a DATA packet are the caller's. A file sent in netascii
    mode has to be converted before it is cut into blocks.

## Running on a microcontroller

Both modules are `@tier(embedded)` throughout and allocate nothing, and
the package declares no effects. `tests/embedded_probe.nv` is a firmware
program that runs a client and a server in one image: a client reads a
700-byte file with a negotiated 256-byte block, and a client writes the
same file at the default 512, and each is checked byte by byte. It is
compiled for a Cortex-M as part of this package's checks, so the claim is
built rather than asserted. Under QEMU it prints `PASS: tftp-embedded`.

The image is 22,820 bytes of flash and 520 bytes of static RAM, which
includes the runtime and the board's startup code. The probe's own
frame holds two packets of 1,040 bytes and two transfers of 160 bytes.
Of that flash, 9,994 bytes are the program (the two modules and the probe), 5,392 the runtime and 4,568 libgcc.

A bootloader that fetches an image keeps the transfer, one packet buffer
and a flash page, and no heap. The device effects are the caller's: the
package reads no register and no clock, so a call costs what its
caller's datagram transport costs.

## What is not included

- **Sockets and timers.** The caller moves the datagrams and says how
  much time has passed. A socket or a radio driver is another package.
- **Netascii translation.** See rule 10.
- **Block sizes above 1024.** See rule 4. A larger `TFTP_BLOCK_MAX`
  would be a change to the capacity of `TftpPacket`.
- **The windowsize option of RFC 7440 and the blksize2 option.** An
  option the package does not know is skipped when a request is read, so
  a server built on it answers a request that carries them as if it did
  not, and the transfer is lock-step.
- **Dallying.** A receiver that is done does not run a timer of its own.
  It answers a repeated final DATA, and the caller closes the transfer
  when it chooses.
- **The checks on the filename.** A server decides whether a name is
  allowed. `file_is` compares the name in a request with a string; a
  path check, a root directory and a permission belong to the program.
- **A file system.** The caller reads and writes the file.

## Related packages

- [cobs-nv](https://novo-lang.org/packages/cobs-nv) and
  [frame-nv](https://novo-lang.org/packages/frame-nv) frame a byte
  stream on a serial link, which TFTP, a datagram protocol, does not
  need.

## Tests

The suite is three files under `tests/`. What they assert comes from the
RFCs rather than from an implementation.

`tests/tftpwire_tests.nv` holds the packets to byte strings worked out by
hand from the layouts in RFC 1350, 2347 and 2349, including RFC 2349's own
example of an OACK, and covers every way a request, an option or a packet
can be refused.

`tests/tftpxfer_tests.nv` plays the other end by hand. It follows a read
of three blocks, a read of a file that is a whole number of blocks, a
write of three blocks, the negotiation in each direction and each way an
OACK is refused, the server's side for a read and for a write, the errors
and the packets that have no place, the timer, and the wrap of the block
number from 65535 to 0.

`tests/tftp_loop_tests.nv` runs a client against a server, in each
direction, for files of 0, 1, 511, 512, 513, 1023, 1024, 1025, 2048 and
5000 bytes, for block sizes of 8, 100, 512 and 1024, with the server
lowering the block size, with a write that is larger than the server has
room for, with each packet of a short exchange lost in turn and in pairs,
and with a peer that never answers.

`tests/embedded_probe.nv` is the microcontroller program described above.

## Licence

Apache-2.0. See `LICENSE`.
