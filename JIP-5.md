# JIP-5: Secret key derivation

This JIP proposes a standard method for deriving a set of validator secret keys from a 32-byte
seed.

## Derivation method

Given a 32-byte seed `seed`, a set of secret keys is derived as follows:

    ed25519_secret = blake2b("jam_val_key_ed25519" ++ seed)
    bandersnatch_secret = blake2b("jam_val_key_bandersnatch" ++ seed)
    bandersnatch_scalar = decode_as_le(sha512(bandersnatch_secret)) % r

Where:

- The strings are ASCII-encoded with no terminator.
- `++` indicates concatenation.
- `blake2b` is the BLAKE2b hash function with 32-byte output.
- `sha512` is the SHA-512 hash function.
- `decode_as_le` is a function that decodes the input as a little-endian unsigned integer.
- `r` is the order of the prime subgroup of the Bandersnatch elliptic curve.
- `ed25519_secret` is a 32-byte Ed25519 secret key as defined in
  <https://ed25519.cr.yp.to/ed25519-20110926.pdf>.
- `bandersnatch_scalar` is a secret key scalar as defined in
  <https://github.com/davxy/bandersnatch-vrf-spec/blob/main/specification.pdf>.

## Trivial seeds

We define the following function for deriving a 32-byte seed from a 32-bit unsigned integer:

    trivial_seed(i) = repeat_8_times(encode_as_32bit_le(i))

Such seeds should be used for testing purposes only.

## Test vectors

    seed = trivial_seed(0) = 0000000000000000000000000000000000000000000000000000000000000000
    ed25519_secret = 996542becdf1e78278dc795679c825faca2e9ed2bf101bf3c4a236d3ed79cf59
    ed25519_public = 4418fb8c85bb3985394a8c2756d3643457ce614546202a2f50b093d762499ace
    bandersnatch_secret = 007596986419e027e65499cc87027a236bf4a78b5e8bd7f675759d73e7a9c799
    bandersnatch_public = ff71c6c03ff88adb5ed52c9681de1629a54e702fc14729f6b50d2f0a76f185b3

    seed = trivial_seed(1) = 0100000001000000010000000100000001000000010000000100000001000000
    ed25519_secret = b81e308145d97464d2bc92d35d227a9e62241a16451af6da5053e309be4f91d7
    ed25519_public = ad93247bd01307550ec7acd757ce6fb805fcf73db364063265b30a949e90d933
    bandersnatch_secret = 12ca375c9242101c99ad5fafe8997411f112ae10e0e5b7c4589e107c433700ac
    bandersnatch_public = dee6d555b82024f1ccf8a1e37e60fa60fd40b1958c4bb3006af78647950e1b91

    seed = trivial_seed(2) = 0200000002000000020000000200000002000000020000000200000002000000
    ed25519_secret = 0093c8c10a88ebbc99b35b72897a26d259313ee9bad97436a437d2e43aaafa0f
    ed25519_public = cab2b9ff25c2410fbe9b8a717abb298c716a03983c98ceb4def2087500b8e341
    bandersnatch_secret = 3d71dc0ffd02d90524fda3e4a220e7ec514a258c59457d3077ce4d4f003fd98a
    bandersnatch_public = 9326edb21e5541717fde24ec085000b28709847b8aab1ac51f84e94b37ca1b66

    seed = trivial_seed(3) = 0300000003000000030000000300000003000000030000000300000003000000
    ed25519_secret = 69b3a7031787e12bfbdcac1b7a737b3e5a9f9450c37e215f6d3b57730e21001a
    ed25519_public = f30aa5444688b3cab47697b37d5cac5707bb3289e986b19b17db437206931a8d
    bandersnatch_secret = 107a9148b39a1099eeaee13ac0e3c6b9c256258b51c967747af0f8749398a276
    bandersnatch_public = 0746846d17469fb2f95ef365efcab9f4e22fa1feb53111c995376be8019981cc

    seed = trivial_seed(4) = 0400000004000000040000000400000004000000040000000400000004000000
    ed25519_secret = b4de9ebf8db5428930baa5a98d26679ab2a03eae7c791d582e6b75b7f018d0d4
    ed25519_public = 8b8c5d436f92ecf605421e873a99ec528761eb52a88a2f9a057b3b3003e6f32a
    bandersnatch_secret = 0bb36f5ba8e3ba602781bb714e67182410440ce18aa800c4cb4dd22525b70409
    bandersnatch_public = 151e5c8fe2b9d8a606966a79edd2f9e5db47e83947ce368ccba53bf6ba20a40b

    seed = trivial_seed(5) = 0500000005000000050000000500000005000000050000000500000005000000
    ed25519_secret = 4a6482f8f479e3ba2b845f8cef284f4b3208ba3241ed82caa1b5ce9fc6281730
    ed25519_public = ab0084d01534b31c1dd87c81645fd762482a90027754041ca1b56133d0466c06
    bandersnatch_secret = 75e73b8364bf4753c5802021c6aa6548cddb63fe668e3cacf7b48cdb6824bb09
    bandersnatch_public = 2105650944fcd101621fd5bb3124c9fd191d114b7ad936c1d79d734f9f21392e

    seed = f92d680ea3f0ac06307795490d8a03c5c0d4572b5e0a8cffec87e1294855d9d1
    ed25519_secret = f21e2d96a51387f9a7e5b90203654913dde7fa1044e3eba5631ed19f327d6126
    ed25519_public = 11a695f674de95ff3daaff9a5b88c18448b10156bf88bc04200e48d5155c7243
    bandersnatch_secret = 06154d857537a9b622a9a94b1aeee7d588db912bfc914a8a9707148bfba3b9d1
    bandersnatch_public = 299bdfd8d615aadd9e6c58718f9893a5144d60e897bc9da1f3d73c935715c650
