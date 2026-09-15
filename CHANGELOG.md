# Changelog

All notable changes to libssl-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-09-15

The first release: forty-three entry points of the OpenSSL libssl C
API, one `@ffi` declaration each, and no logic.

### Added

- `libssl` — the whole surface, in six groups.
  - The library and its protocol methods: `OPENSSL_init_ssl`,
    `TLS_method`, `TLS_client_method` and `TLS_server_method`.
  - The context: `SSL_CTX_new`, `SSL_CTX_free`, `SSL_CTX_ctrl`, the
    two verification settings, the certificate and key loaders, the
    two trust-store loaders and the cipher list.
  - The connection: `SSL_new`, `SSL_free`, `SSL_ctrl`, `SSL_set_fd`,
    `SSL_get_fd`, `SSL_set1_host` and the two state setters.
  - The handshake: `SSL_connect`, `SSL_accept`, `SSL_do_handshake`,
    `SSL_is_init_finished` and `SSL_session_reused`.
  - Reading, writing and shutting down: `SSL_read`, `SSL_read_ex`,
    `SSL_write`, `SSL_write_ex`, `SSL_pending`, `SSL_has_pending`,
    `SSL_shutdown` and `SSL_get_shutdown`.
  - The result of it: `SSL_get_error`, `SSL_get_version`,
    `SSL_get_current_cipher`, `SSL_CIPHER_get_name`,
    `SSL_get_verify_result`, `SSL_get1_peer_certificate` and
    `SSL_get_servername`.
- `tests/libssl_tests.nv` — ten tests over the signatures. They call
  the C library, so they need libssl installed. No test opens a
  socket: a connection with no file descriptor attached is enough to
  exercise the handshake, transfer and shutdown entry points.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### Named as missing

**The macro surface.** OpenSSL spells most of its `SSL_set_*` and
`SSL_CTX_set_*` calls as C macros over `SSL_ctrl` and `SSL_CTX_ctrl`,
and a macro is not a symbol a binding can resolve. Both control entry
points are declared, and the README gives the command numbers for the
protocol version bounds and for the server name indication.

**Everything that takes a C function pointer.** The verification
callback, the passphrase callback, the ALPN and SNI selection callbacks
and the info callback are absent, because a novo-lang function is not a
C function pointer. `SSL_CTX_set_verify` is here with its callback
argument, which a caller passes 0 for.

**The certificate and key objects.** `X509`, `EVP_PKEY` and their
accessors live in libcrypto, and a `sys` package wraps one library.
`SSL_get1_peer_certificate` answers an address, and releasing it takes
`X509_free` from libcrypto-sys.
