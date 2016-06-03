# Joyent SmartDC Authentication Library

Utility functions to sign http requests to SmartDC services.

    var fs = require('fs');
    var auth = require('smartdc-auth');

    var signer = auth.privateKeySigner({
          key: fs.readFileSync(process.env.HOME + '/.ssh/id_rsa', 'utf8'),
          user: process.env.SDC_CLI_ACCOUNT
    });

## Overview

Authentication to SmartDC is built on top of Joyent's
[http-signature](https://github.com/joyent/node-http-signature) specification.
In most situations, you will only need to sign the value of the HTTP `Date`
header using your SSH private key; doing this allows you to create shell
functions to interact with SmartDC (see below).  All requests to SmartDC require an
HTTP Authorization header where the scheme is `Signature`.  Full details are
available in the `http-signature` specification, but a simple form is:

    Authorization: Signature keyId="/:login/keys/:fingerprint",algorithm="rsa-sha256" $base64_signature

The `keyId` for SmartDC is always `/$your_joyent_login/keys/$ssh_fingerprint`,
and the supported algorithms are: `rsa-sha1`, `rsa-sha256` and `dsa-sha1`.  You
then just append the base64 encoded signature.

## New API: KeyRing

### `new mod_sdcauth.KeyRing([options])`

Create a new SDC keyring. KeyRing instances use a list of plugins in order to
locate keys on the local system - via the filesystem, via the SSH agent, or any
other mechanism.

Parameters

- `options`: an Object containing properties:
  - `plugins`: an Array of String, names of plugins to enable

Any additional keys set in the `options` object will be passed through to
plugins as options for their processing.

Available plugins:
- `agent`: Gets keys from the OpenSSH agent. Options:
  - `sshAgentOpts`: an Object, options to be passed to `mod_sshpk_agent.Client`
- `homedir`: Gets keys from a directory on the filesystem. Options:
  - `keyDir`: a String, path to look in for keys, defaults to `$HOME/.ssh`

### `KeyRing#list(cb)`

Lists all available keys in all plugins, organised by their Key ID.

Parameters

- `cb`: a Function `(err, keypairs)` with parameters:
  - `err`: an Error or `null`
  - `keypairs`: an Object, keys: String key IDs, values: Array of instances of
    `KeyPair`

### `KeyRing#find(fingerprint, cb)`

Searches active plugins for an SSH key matching the given fingerprint. Calls
`cb` with an array of `KeyPair` instances that match, ordered arbitrarily.

Parameters:
 - `fingerprint`: an `sshpk.Fingerprint`
 - `cb`: a Function `(err, keypairs)`, with parameters:
   - `err`: an Error or `null`
   - `keypairs`: an Array of `KeyPair` instances

### `KeyRing#findSigningKeyPair(fingerprint, cb)`

Searches active plugins for an SSH key matching the given fingerprint. Chooses
the best available signing key of those available (preferably unlocked) and
calls `cb` with this single `KeyPair` instance.

Parameters:
 - `fingerprint`: an `sshpk.Fingerprint`
 - `cb`: a Function `(err, keypair)`, with parameters:
   - `err`: an Error or `null`
   - `keypair`: a `KeyPair` instance

## KeyPair

### `KeyPair.fromPrivateKey(privKey)`

Constructs a KeyPair unrelated to any keychain, based directly on a given
private key. This is mostly useful for compatibility purposes.

Parameters:
 - `privKey`: an `sshpk.PrivateKey`

### `KeyPair#plugin`

String, name of the plugin through which this KeyPair was found.

### `KeyPair#source`

String (may be `undefined`), human-readable name of the source that the KeyPair
came from when discovered (e.g. for a plugin that searches the filesystem, this
could be the path to the key file).

### `KeyPair#comment`

String, comment that was stored with the key, if any.

### `KeyPair#canSign()`

Returns Boolean `true` if this key pair is complete (has a private and public
key) and can be used for signing. Note that this returns `true` for locked
keys.

### `KeyPair#isLocked()`

Returns Boolean `true` if this key pair is locked and may be unlocked using
the `unlock()` method.

### `KeyPair#unlock(passphrase)`

Unlocks an encrypted key pair, allowing it to be used for signing and the
`getPrivateKey()` method to be called.

Parameters:
 - `passphrase`: a String, passphrase for decryption

### `KeyPair#getKeyId()`

Returns the String key ID for this key pair. This is specifically the key ID
as used in HTTP signature auth for SDC and Manta. Currently this is a
hex-format MD5 fingerprint of the key, but this may change in future.

### `KeyPair#getPublicKey()`

Returns the `sshpk.Key` object representing this pair's public key.

### `KeyPair#getPrivateKey()`

Returns the `sshpk.PrivateKey` object representing this pair's private key. If
unavailable, this method will throw an `Error`.

### `KeyPair#createRequestSigner(options)`

Creates an `http-signature` `RequestSigner` object for signing an HTTP request
using this key pair's private key.

Parameters:
 - `options`, an Object...

### `KeyPair#createSign(options)`

Creates a `sign()` function (matching the legacy `smartdc-auth` API) for
signing arbitrary data with this key pair's private key.

Parameters:
 - `options`, an Object...

## Authenticating Requests

When creating a smartdc client, you'll need to pass in a callback function for
the `sign` parameter.  smartdc-auth ships with three constructors that return
such functions, which may suit your need: `cliSigner`, `privateKeySigner` and
`sshAgentSigner`.

### `privateKeySigner(options);`

A basic signer which signs using a given PEM (PKCS#1) format private key only.
Ideal for simple use cases where the key is stored in a file on the filesystem
ready for use.

- `options`: an Object containing properties:
  - `key`: a String, PEM-format (PKCS#1) private key, for any supported algorithm
  - `user`: a String, SDC login name to be used in the full keyId, above
  - `subuser`: an optional String, SDC sub-user login name
  - `keyId`: optional String, the fingerprint of the `key` (not the same as the
             full keyId given to the server). Ignored unless it does not match
             the given `key`, then an Error will be thrown.

### `sshAgentSigner(options);`

Signs requests using a key that is stored in the OpenSSH agent. Opens and manages
a connection to the current session's agent during operation.

- `options`: an Object containing properties:
  - `keyId`: a String, fingerprint of the key to retrieve from the agent
  - `user`: a String, SDC login name to be used
  - `subuser`: an optional String, SDC sub-user login name
  - `sshAgentOpts`: an optional Object, any additional options to pass through
                    to the SSHAgent constructor (eg `timeout`)

### `cliSigner(options);`

Signs requests using a key located either in the OpenSSH agent, or found in
the filesystem under `$HOME/.ssh` (or its equivalent on your platform).

This is generally intended for use with CLI utilities (eg the `sdc-listmachines`
tool and family), hence the name.

- `options`: an Object containing properties:
  - `keyId`: a String, fingerprint of the key to retrieve or find
  - `user`: a String, SDC login name to be used
  - `subuser`: an optional String, SDC sub-user login name
  - `sshAgentOpts`: an optional Object, any additional options to pass through
                    to the SSHAgent constructor (eg `timeout`)
  - `algorithm`: DEPRECATED, an optional String, the signing algorithm to use.
                 If this does not match up with the algorithm of the key (once
                 it is located), an Error will be thrown.

(The `algorithm` option is deprecated as its backwards-compatible behaviour is
to apply only to keys that were found on disk, not in the SSH agent. If you have
a compelling use case for a replacement for this option in future, please open
an issue on this repo).

The `keyId` fingerprint does not necessarily need to be the exact format
(hex MD5) as sent to the server -- it can be in any fingerprint format supported
by the [`sshpk`](https://github.com/arekinath/node-sshpk) library.

As of version 2.0.0, an invalid fingerprint (one that can never match any key,
because, for example, it contains invalid characters) will produce an exception
immediately rather than returning a `sign` function.

Note that the `cliSigner` and `sshAgentSigner` are not suitable for server
applications, or any other system where the performance degradation necessary
to interact with SSH is not acceptable; put another way, you should only use
it for interactive tooling, such as the CLI that ships with node-smartdc.

### Writing your own signer

Should you wish to write a custom plugin, the expected implementation of the
`sign` callback is a function of the form `function (string, callback)`.
`string` is generated by node-smartdc (typically the value of the Date header),
and callback is of the form `function (err, object)`, where `object` has the
following properties:

    {
        algorithm: 'rsa-sha256',   // the signing algorithm used
        keyId: '7b:c0:5c:d6:9e:11:0c:76:04:4b:03:c9:11:f2:72:7f', // key fingerprint
        signature: $base64_encoded_signature,  // the actual signature
        user: 'mark'   // the user to issue the call as.
    }

Use-cases where you would need to write your own signer are things like signing
with a smart-card or other HSM, making remote calls to a central system, etc.

## License

MIT.

## Bugs

See <https://github.com/joyent/node-smartdc-auth/issues>.
