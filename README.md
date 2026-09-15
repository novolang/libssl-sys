# libssl-sys

Transport Layer Security (TLS) is the protocol that encrypts and
authenticates a network connection. OpenSSL implements it in two C
libraries: libcrypto holds the cryptographic primitives, and libssl
holds the protocol built on them. The library's interface is documented
in [the OpenSSL manual](https://docs.openssl.org/3.0/man3/). This
package declares forty-three of libssl's entry points to novo-lang, one
declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in libssl. The package contains no logic of
its own, and it does nothing without the C library installed. The
forty-three entry points are the ones a program needs to configure a
context, run a handshake over a socket it already has, transfer bytes
and close down; the section "What is not included" says what a program
still cannot do with them alone.

## What it is

A **context** is the settings shared by every connection a program
makes. It holds the protocol version bounds, the certificate and key
this side presents, the certificates it will accept from the other
side, and the cipher suites it will offer. `SSL_CTX_new` creates one
from a protocol method, and `SSL_CTX_free` releases it.

A **protocol method** says which side of the conversation the context
is for and which protocol family it speaks. `TLS_client_method` is the
side that connects, `TLS_server_method` the side that accepts, and
`TLS_method` either.

A **connection** is one TLS conversation. `SSL_new` creates one from a
context, and it takes a copy of the context's settings as they stand at
that moment. A later change to the context does not reach a connection
already created from it.

libssl does not open sockets. A program connects a socket itself and
hands the file descriptor to `SSL_set_fd`. libssl reads and writes
through that descriptor and never closes it.

The **handshake** is the exchange that agrees a cipher suite, checks
the other side's certificate and derives the keys. `SSL_connect` runs
it as a client and `SSL_accept` as a server. After it succeeds,
`SSL_read` and `SSL_write` carry ordinary bytes over the encrypted
connection.

**Verification** is the check that the peer's certificate is signed by
a certificate the program already trusts, and that it names the host
the program meant to reach. The two are separate:
`SSL_CTX_set_default_verify_paths` loads the trusted certificates, and
`SSL_set1_host` names the host. A client that loads the first and skips
the second accepts a valid certificate for any host.

A **close notify** is the alert each side sends to say it has finished.
`SSL_shutdown` sends it. A connection whose socket simply closes
without one may have been cut short by an attacker, which is what the
alert exists to rule out.

## Install

```
novo pkg add libssl-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its headers come from the system package `libssl-dev`:

```
sudo apt install libssl-dev
```

On macOS the Homebrew formula is `openssl@3`, and its libraries are not
on the default search path. On other systems OpenSSL builds from its own
source.

libssl depends on libcrypto, and pkg-config adds it, so a program that
links this package links both shared libraries.

## Example

A client handshake over a socket the program already opened:

```novo ignore
use libssl

fn main(fd: Int) [io, ffi]
    // The context holds the settings; the connection is made from it.
    let ctx = libssl.ssl_ctx_new(libssl.tls_client_method())
    // 1 is SSL_VERIFY_PEER, and 0 selects the built-in check.
    libssl.ssl_ctx_set_verify(ctx, 1, 0)
    let _ = libssl.ssl_ctx_set_default_verify_paths(ctx)
    let ssl = libssl.ssl_new(ctx)
    // The host the certificate must name.
    let _ = libssl.ssl_set1_host(ssl, "example.com")
    let _ = libssl.ssl_set_fd(ssl, fd)

    let rc = libssl.ssl_connect(ssl) as i32
    if rc != 1
        println("handshake failed, reason ${libssl.ssl_get_error(ssl, rc)}")
        return
    let cipher = libssl.ssl_get_current_cipher(ssl)
    println(ptr.read_str(libssl.ssl_get_version(ssl)) + " with "
            + ptr.read_str(libssl.ssl_cipher_get_name(cipher)))

    let request = ptr.alloc(64)
    ptr.write_bytes_buf(request, bytes.from_str("GET / HTTP/1.0\r\n\r\n"))
    let _ = libssl.ssl_write(ssl, request, 18)
    let reply = ptr.alloc(4096)
    let got = libssl.ssl_read(ssl, reply, 4096) as i32
    println("read ${got} byte(s)")

    let _ = libssl.ssl_shutdown(ssl)
    ptr.free(reply)
    ptr.free(request)
    libssl.ssl_free(ssl)
    libssl.ssl_ctx_free(ctx)
```

The example is fenced as an illustration rather than a compiled block
because it needs a connected socket, which this repository does not
open.

## What the package contains

| Module | Contents |
| --- | --- |
| `libssl` | Every entry point, in six groups: the library and its protocol methods, the context, the connection, the handshake, the transfer and shutdown, and the result of the handshake. |

The six groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Library and methods | 4 | Initialises libssl and answers the three protocol methods. |
| Context | 11 | Creates the shared settings, loads the certificate, the key and the trust store, and sets the version bounds and the cipher list. |
| Connection | 8 | Creates a connection, attaches a file descriptor and names the expected host. |
| Handshake | 5 | Runs the handshake as either side and reports whether it finished. |
| Transfer and shutdown | 8 | Reads and writes encrypted bytes, counts what is buffered, and sends the close notify. |
| Result | 7 | Names the reason for a failure, the protocol version, the cipher suite, the verification result and the peer. |

## How to choose an entry point

`SSL_read` and `SSL_write` answer a count, and a zero from them is
ambiguous: it may be a clean close or a failure. `SSL_read_ex` and
`SSL_write_ex` answer 1 or 0 and write the count into a slot, which
separates the two. Use the `_ex` pair in new code.

`SSL_connect` and `SSL_accept` fix the side. `SSL_do_handshake` uses
whichever side the connection is already in, so it is the call a
non-blocking program returns to after waiting on the socket.

`SSL_CTX_set_default_verify_paths` loads the operating system's trusted
certificates. `SSL_CTX_load_verify_locations` loads a file the program
names, which is for a private certificate authority.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** Every handle the C
   library returns arrives as the address it returned.
2. **An entry point that answers a C `int` answers it in 32 bits.**
   Write `as i32` before comparing the answer with a negative number,
   as the example does for `SSL_connect`.
3. **libssl does not open or close sockets.** The program connects the
   socket, passes the descriptor to `SSL_set_fd`, and closes it itself
   after `SSL_free`.
4. **`SSL_get_error` must be called before any other call on the
   connection.** It reports on the most recent one, and the next call
   replaces what it has to say. OpenSSL manual, `SSL_get_error`.
5. **A negative answer is not always a failure.** On a non-blocking
   socket, `SSL_get_error` answering 2, `SSL_ERROR_WANT_READ`, or 3,
   `SSL_ERROR_WANT_WRITE`, means wait for the socket and repeat the
   *same call with the same arguments*. Changing the arguments between
   attempts is undefined.
6. **Verifying the certificate and checking the host name are two
   decisions.** `SSL_CTX_set_verify` with mode 1 makes the handshake
   fail on an untrusted certificate. `SSL_set1_host` makes it fail on a
   certificate for the wrong host. A client needs both.
7. **`SSL_get_verify_result` answers 0 for a peer that sent no
   certificate.** Check `SSL_get1_peer_certificate` as well before
   treating a 0 as proof of anything. OpenSSL manual,
   `SSL_get_verify_result`.
8. **A macro is not a symbol.** `SSL_CTX_set_min_proto_version`,
   `SSL_set_tlsext_host_name` and most of their neighbours are C macros
   over `SSL_CTX_ctrl` and `SSL_ctrl`. The two control entry points are
   here, and these are their numbers.

   | Command | Number | Integer argument | Pointer argument |
   | --- | --- | --- | --- |
   | `SSL_CTRL_SET_TLSEXT_HOSTNAME` | 55 | 0, the host-name type | the host name, as a C string |
   | `SSL_CTRL_SET_MIN_PROTO_VERSION` | 123 | 771 for TLS 1.2, 772 for TLS 1.3 | 0 |
   | `SSL_CTRL_SET_MAX_PROTO_VERSION` | 124 | as above | 0 |

9. **A connection copies the context when it is created.** Configure
   the context fully before calling `SSL_new`.
10. **`SSL_shutdown` takes two calls.** The first sends this side's
    close notify and answers 0. The second waits for the peer's and
    answers 1. A program that does not need the peer's may stop after
    the first.
11. **The private key file must be unencrypted.** Decrypting one needs
    a passphrase callback, which is a C function pointer and is not in
    this package.

## What is not included

- **Every entry point that takes a C function pointer.** The
  verification callback, the passphrase callback, the ALPN selection
  callback, the SNI selection callback, the info callback and the
  session cache callbacks are all absent. A novo-lang function is not a
  C function pointer.
- **Every entry point that passes or returns a structure by value.**
  The novo-lang foreign function interface passes integers, floats and
  strings, and nothing else.
- **The certificate and key objects.** `X509`, `EVP_PKEY` and their
  accessors are in libcrypto, and one `sys` package wraps one library.
  `SSL_get1_peer_certificate` answers an address, and `X509_free` from
  libcrypto-sys releases it.
- **The BIO abstraction.** `BIO_new`, `SSL_set_bio` and their
  neighbours let libssl read and write through something other than a
  file descriptor, including a pair of memory buffers. They are in
  libcrypto.
- **DTLS.** `DTLS_method` and its neighbours are the datagram form of
  the protocol. They are left out of the first release.
- **Session resumption.** `SSL_get1_session`, `SSL_set_session` and the
  session cache let a second connection skip most of the handshake.
  They are left out of the first release; `SSL_session_reused` is here
  so a caller can see when resumption happened.
- **The error queue.** `ERR_get_error` and `ERR_error_string_n` turn a
  failure into text, and they are in libcrypto. `SSL_get_error` is
  here, and it answers a reason rather than a message.
- **The deprecated protocol methods.** `SSLv23_method`, `TLSv1_method`
  and their neighbours are superseded by `TLS_method` with version
  bounds.

## Related packages

`tls-nv` is the TLS implementation written in novo-lang, and the
standard library's TLS stands on this C library today. `tls-nv` is
planned and not published yet.

`libcrypto-sys` binds the other half of OpenSSL: message digests,
ciphers, HMAC and random bytes. A program that needs a certificate's
fields, the error queue or a BIO takes that package as well.

Choose `tls-nv` when the program must build for a microcontroller or
for WebAssembly, or when a C toolchain is not wanted. Choose this
package when the program needs OpenSSL's own behaviour, its
configuration files or its trust store.

## Tests

`tests/libssl_tests.nv` holds ten tests written against the signatures.
They call the C library, so `novo test` needs libssl installed and
linkable:

```
novo test tests/libssl_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

No test opens a socket. The suite asserts what the reference specifies
for a connection with no transport under it: that a context is created
and configured, that a cipher list naming no real cipher is refused,
that a missing certificate file is refused, that a connection before its
handshake has no cipher and no peer, and that `SSL_connect` on a
connection with no file descriptor fails with `SSL_ERROR_SYSCALL`. The
server name indication is set through `SSL_ctrl` and read back through
`SSL_get_servername`, which exercises the pointer argument of the
control call.

## Implementation status

| Group | State |
| --- | --- |
| Library and methods | Complete for TLS. DTLS is absent. |
| Context | Complete for the file-based certificate, key and trust store. |
| Connection | Complete for a file descriptor transport. |
| Handshake | Complete. |
| Transfer and shutdown | Complete. |
| Result | Complete except the session accessors. |
| Callbacks | Absent. Every one takes a C function pointer. |
| BIO | Absent. It is in libcrypto. |
| Session resumption | Absent. Left out of the first release. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

OpenSSL itself is distributed under the Apache 2.0 licence, and
installing it is the reader's own step.
