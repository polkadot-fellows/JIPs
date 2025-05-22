# JIP-5: Secret key derivation

This JIP proposes a standard method for deriving a set of validator secret keys from a 32-byte
seed.

## Derivation method

Given a 32-byte seed `seed`, a set of secret keys is derived as follows:

    ed25519_secret = blake2b(personal = "jam-valk-ed25519", seed)
    bandersnatch_seed = blake2b(personal = "jam-valk-bsnatch", seed)
    bandersnatch_secret = decode_as_le(sha512(bandersnatch_seed)) % r

Where:

- The strings are ASCII-encoded with no terminator (both exactly 16 bytes long).
- `blake2b` is the BLAKE2b hash function with 32-byte output and given personalisation.
- `sha512` is the SHA-512 hash function.
- `decode_as_le` is a function that decodes the input as a little-endian integer.
- `r` is the order of the prime subgroup of the Bandersnatch elliptic curve.
- `ed25519_secret` is a 32-byte Ed25519 secret key as defined in
  <https://ed25519.cr.yp.to/ed25519-20110926.pdf>.
- `bandersnatch_secret` is a secret key scalar as defined in
  <https://github.com/davxy/bandersnatch-vrf-spec/blob/main/specification.pdf>.

## Trivial seeds

We define the following function for deriving a 32-byte seed from 32-bit unsigned integer:

    trivial_seed(i) = repeat_8_times(encode_as_32bit_le(i))

Such seeds should be used for testing purposes only.

## Test vectors

    seed = trivial_seed(0) = 0000000000000000000000000000000000000000000000000000000000000000
    ed25519_secret = d68962bd586d5f8bb216cf671161c8edeb23a094371d199a656024ae2feed20b
    ed25519_public = 39648547f495eaac909f7840330e73cdf837f8db8a4f583728742c958adac82b
    bandersnatch_seed = d5a7d25f2abbfb2122a71066423009f29eeeb76bec2afe3a0c639f094b1f8088
    bandersnatch_public = 8e46bce7abcf8d3d87109f2b6b1a8c941bf100435ad6b75bf0e92110241bbcf0

    seed = trivial_seed(1) = 0100000001000000010000000100000001000000010000000100000001000000
    ed25519_secret = 74a600d03c44aaf79316575cc1c9bf2787c9560f35ebfee60ce75d0d153a3e85
    ed25519_public = ebf47a0ff95ae6c7e8a6c0f97ff993d8b62fb9177d09fb088e5441cd9b2fe469
    bandersnatch_seed = bdd114c73e1012739f898ec016759f2cbfe326640749e20e36694db7a63a0156
    bandersnatch_public = d8bd18ead243182a535d3e1cffd7831dbab9d57f317bb90a0a43a208fd5927af

    seed = trivial_seed(2) = 0200000002000000020000000200000002000000020000000200000002000000
    ed25519_secret = 75a021a4f317243ce6b51630fb7d3fd6d94961b938b4aebd78f2b1d4ac70fbf5
    ed25519_public = daf8a2a8d9e13d03ef1b173019385895d17f3cfb7f7160cddebd598909853383
    bandersnatch_seed = fa3d89bdce2ee8c4080e0e6c4c713fce139e119de3a15c49362d8a4a3877c221
    bandersnatch_public = f634fa63f79303d3303a5d3595c804b61878fd6ca4ac3516000e82c4ff5bf49b

    seed = trivial_seed(3) = 0300000003000000030000000300000003000000030000000300000003000000
    ed25519_secret = d8d5d42a75538d46fd4e6499a3e571793ce9850b4be3537d5078436d1ad3edd6
    ed25519_public = d5e009718d549d240160927aaf6e81069380a3ec015d3c40775fc7c33fa09129
    bandersnatch_seed = ef6ec45768a7dd22cb0e748aa68fed7c2ce76d1bd5a5d4be5515b0f334d75e4c
    bandersnatch_public = 2618ddf8052545373359567d5dbdaef56513e19a6c0c5ecdb97bc2ac1877d107

    seed = trivial_seed(4) = 0400000004000000040000000400000004000000040000000400000004000000
    ed25519_secret = b911d5dabc029bf5002948992a6470af766187ed488be36a00b0fd5ce56313f4
    ed25519_public = f6bb95f9e6fc70c0d1aa4552e2594e31ecf5b88edffdb34ca9fa4539e5b34038
    bandersnatch_seed = 463295a80c5e28672cfa43bc1a8acba33ec0a67d0420f8f256282e2799bb197a
    bandersnatch_public = fd698db7d9e2a9edc8578019d40ce9d51c2b9183bf9324bab9a8d64c905befb1

    seed = trivial_seed(5) = 0500000005000000050000000500000005000000050000000500000005000000
    ed25519_secret = 27f002a0a179ca0cc13f68326fe4077fbd8504f4319f26dbb7da3ab4d4e153ec
    ed25519_public = 4155988e4a398eb6962bbeea98a7e79e28d42346fe7ebcf3b9476474343fc2f3
    bandersnatch_seed = 1d19b265b0c018177b81a953ad455c88c4721f5ab3015a340a116c0216a08892
    bandersnatch_public = d83c0731105d78c61323446c328d707a9ef7821e74e96ec494b2e67536946f4b

    seed = 3115701e281bea0af863ccd901d3e4c4e42e04869a80d8e8504593b070859dcb
    ed25519_secret = 0bb3c0dc9442b6e5d0c92e1572384d01ed34c261a6e157dd37ab01821c64a825
    ed25519_public = e5b453caac1935025cd38a75e65d811b6df936d8654ff9df19bcaaebcc1366c0
    bandersnatch_seed = 1f42a56c38b044f37a3e856aa08d8552e864b5b527a21a8e13e1d691f3dc70e7
    bandersnatch_public = d4751fd4ab89f09ccc7fc4ba51fdb06daa2fdeac3e0eef29f7ce1f9d4af3ed01
