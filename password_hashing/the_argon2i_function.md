# The Argon2 memory-hard function

Since version 1.0.9, Sodium provides a password hashing scheme called Argon2.

Argon2 summarizes the state of the art in the design of memory-hard functions.

It aims at the highest memory filling rate and effective use of multiple computing units, while still providing defense against tradeoff attacks.

It prevents ASICs implementations from having a significant advantage over software implementations.

## Example 1: key derivation

```c
#define PASSWORD "Correct Horse Battery Staple"
#define KEY_LEN crypto_box_SEEDBYTES

unsigned char salt[crypto_pwhash_SALTBYTES];
unsigned char key[KEY_LEN];

randombytes_buf(salt, sizeof salt);

if (crypto_pwhash
    (key, sizeof key, PASSWORD, strlen(PASSWORD), salt,
     crypto_pwhash_OPSLIMIT_INTERACTIVE,
     crypto_pwhash_MEMLIMIT_INTERACTIVE, NULL) != 0) {
    /* out of memory */
}
```

## Example 2: password storage

```c
#define PASSWORD "Correct Horse Battery Staple"

char hashed_password[crypto_pwhash_STRBYTES];

if (crypto_pwhash_str
    (hashed_password, PASSWORD, strlen(PASSWORD),
     crypto_pwhash_OPSLIMIT_SENSITIVE,
     crypto_pwhash_MEMLIMIT_SENSITIVE) != 0) {
    /* out of memory */
}

if (crypto_pwhash_str_verify
    (hashed_password, PASSWORD, strlen(PASSWORD)) != 0) {
    /* wrong password */
}
```

## Key derivation

```c
int crypto_pwhash(unsigned char * const out,
                                       unsigned long long outlen,
                                       const char * const passwd,
                                       unsigned long long passwdlen,
                                       const unsigned char * const salt,
                                       unsigned long long opslimit,
                                       size_t memlimit,
                                       const crypto_pwhash_options *options);
```

The `crypto_pwhash()` function derives an `outlen` bytes long key from a password `passwd` whose length is `passwdlen` and a salt `salt` whose fixed length is `crypto_pwhash_SALTBYTES` bytes.

The computed key is stored into `out`.

`opslimit` represents a maximum amount of computations to perform. Raising this number will make the function require more CPU cycles to compute a key.

`memlimit` is the maximum amount of RAM that the function will use, in bytes.

`options` is reserved for future use and should be set to `NULL`.

For interactive, online operations, `crypto_pwhash_OPSLIMIT_INTERACTIVE` and `crypto_pwhash_MEMLIMIT_INTERACTIVE` provide base line for these two parameters. This requires 32 Mb of dedicated RAM. Higher values may improve security (see below).

Alternatively, `crypto_pwhash_OPSLIMIT_MODERATE` and `crypto_pwhash_MEMLIMIT_MODERATE` can be used. This requires 128 Mb of dedicated RAM, and takes about 0.7 seconds on a 2.8 Ghz Core i7 CPU.

For highly sensitive data and non-interactive operations, `crypto_pwhash_OPSLIMIT_SENSITIVE` and `crypto_pwhash_MEMLIMIT_SENSITIVE` can be used. With these parameters, deriving a key takes about 3.5 seconds on a 2.8 Ghz Core i7 CPU and requires 512 Mb of dedicated RAM.

The `salt` should be unpredictable. `randombytes_buf()` is the easiest way to fill the `crypto_pwhash_SALTBYTES` bytes of the salt.

Keep in mind that in order to produce the same key from the same password, the same salt, and the same values for `opslimit` and `memlimit` have to be used. Therefore, these parameters have to be stored for each user.

The function returns `0` on success, and `-1` if the computation didn't complete, usually because the operating system refused to allocate the amount of requested memory.

## Password storage

```c
int crypto_pwhash_str(char out[crypto_pwhash_STRBYTES],
                                           const char * const passwd,
                                           unsigned long long passwdlen,
                                           unsigned long long opslimit,
                                           size_t memlimit);
```

The `crypto_pwhash_str()` function puts an ASCII encoded string into `out`, which includes:
- the result of a memory-hard, CPU-intensive hash function applied to the password `passwd` of length `passwdlen`
- the automatically generated salt used for the previous computation
- the other parameters required to verify the password, including the algorithm identifier, its version, `opslimit` and `memlimit`.

`out` must be large enough to hold `crypto_pwhash_STRBYTES` bytes, but the actual output string may be shorter.

The output string is zero-terminated, includes only ASCII characters and can be safely stored into SQL databases and other data stores. No extra information has to be stored in order to verify the password.

The function returns `0` on success and `-1` if it didn't complete successfully.

```c
int crypto_pwhash_str_verify(const char str[crypto_pwhash_STRBYTES],
                                                  const char * const passwd,
                                                  unsigned long long passwdlen);
```

This function verifies that the password `str` is a valid password verification string (as generated by `crypto_pwhash_str()`) for `passwd` whose length is `passwdlen`.

`str` has to be zero-terminated.

It returns `0` if the verification succeeds, and `-1` on error.

## Guidelines for choosing the parameters

Start by determining how much memory the function can use. What will be the highest number of threads/processes evaluating the function simultaneously (ideally, no more than 1 per CPU core)? How much physical memory is guaranteed to be available?

Set `memlimit` to the amount of memory you want to reserve for password hashing.

Then, set `opslimit` to `3` and measure the time it takes to hash a password.

If this it is way too long for your application, reduce `memlimit`, but keep `opslimit` set to `3`.

If the function is so fast that you can afford it to be more computationally intensive without any usability issues, increase `opslimit`.

For online use (e.g. login in on a website), a 1 second computation is likely to be the acceptable maximum.

For interactive use (e.g. a desktop application), a 5 second pause after having entered a password is acceptable if the password doesn't need to be entered more than once per session.

For non-interactive use and infrequent use (e.g. restoring an encrypted backup), an even slower computation can be an option.

But the best defense against brute-force password cracking remains using strong passwords. Libraries such as [passwdqc](http://www.openwall.com/passwdqc/) can help enforce this.

## Constants

- `crypto_pwhash_SALTBYTES`
- `crypto_pwhash_STRBYTES`
- `crypto_pwhash_STRPREFIX`
- `crypto_pwhash_OPSLIMIT_INTERACTIVE`
- `crypto_pwhash_MEMLIMIT_INTERACTIVE`
- `crypto_pwhash_OPSLIMIT_MODERATE`
- `crypto_pwhash_MEMLIMIT_MODERATE`
- `crypto_pwhash_OPSLIMIT_SENSITIVE`
- `crypto_pwhash_MEMLIMIT_SENSITIVE`

## Notes

Do not forget to initialize the library with `sodium_init()`. `crypto_pwhash_*` will still work without doing so, but possibly way slower.

Do not use constants (including `crypto_pwhash_OPSLIMIT_*` and `crypto_pwhash_MEMLIMIT_*`) in order to verify a password. Save the parameters along with the hash instead, and use these saved parameters for the verification.

Alternatively, use `crypto_pwhash_str()` and `crypto_pwhash_str_verify()`, that automatically take care of including and extracting the parameters.

By doing so, passwords can be rehashed using different parameters if required later on.

Cleartext passwords should not stay in memory longer than needed.

It is highly recommended to use `sodium_mlock()` to lock memory regions storing cleartext passwords, and to call `sodium_munlock()` right after `crypto_pwhash_str()` and `crypto_pwhash_str_verify()` return.

`sodium_munlock()` overwrites the region with zeros before unlocking it, so it doesn't have to be done before calling this function.

## Algorithm details

- [Revision 1.3 of Argon2i](https://github.com/P-H-C/phc-winner-argon2/raw/master/argon2-specs.pdf)