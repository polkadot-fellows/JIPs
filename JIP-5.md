# JIP-5: Secret key derivation

This JIP proposes a standard method for deriving a set of validator secret keys from a 32-byte
seed.

## Derivation method

Given a 32-byte seed `seed`, a set of secret key seeds is derived as follows:

    ed25519_secret_seed = blake2b("jam_val_key_ed25519" ++ seed)
    bandersnatch_secret_seed = blake2b("jam_val_key_bandersnatch" ++ seed)

Where:

- The strings are ASCII-encoded with no terminator.
- `++` indicates concatenation.
- `blake2b` is the BLAKE2b hash function with 32-byte output.
- `ed25519_secret_seed` is a 32-byte Ed25519 secret key as defined in
  <https://ed25519.cr.yp.to/ed25519-20110926.pdf>.
- `bandersnatch_secret_seed` is the input seed of the secret key generation procedure in appendix
  A.3 of the Bandersnatch VRF-AD Specification, draft 35, with the suite
  `Bandersnatch-SHA512-ELL2-v1`:
  <https://github.com/davxy/bandersnatch-vrf-spec/blob/draft-35/specification.pdf>.
  The specification gives this procedure as one valid method. JIP-5 selects it, so that the same
  seed gives the same keys in all implementations.

## Trivial seeds

We define the following function for deriving a 32-byte seed from a 32-bit unsigned integer:

    trivial_seed(i) = repeat_8_times(encode_as_32bit_le(i))

Such seeds should be used for testing purposes only.

## Test vectors

    seed = trivial_seed(0) = 0000000000000000000000000000000000000000000000000000000000000000
    ed25519_secret_seed = 996542becdf1e78278dc795679c825faca2e9ed2bf101bf3c4a236d3ed79cf59
    ed25519_public = 4418fb8c85bb3985394a8c2756d3643457ce614546202a2f50b093d762499ace
    bandersnatch_secret_seed = 007596986419e027e65499cc87027a236bf4a78b5e8bd7f675759d73e7a9c799
    bandersnatch_public = 4fc1cc2146c5463a023a66d1fc72829025ef654f1e1dd5c917c40346c2b3aa06

    seed = trivial_seed(1) = 0100000001000000010000000100000001000000010000000100000001000000
    ed25519_secret_seed = b81e308145d97464d2bc92d35d227a9e62241a16451af6da5053e309be4f91d7
    ed25519_public = ad93247bd01307550ec7acd757ce6fb805fcf73db364063265b30a949e90d933
    bandersnatch_secret_seed = 12ca375c9242101c99ad5fafe8997411f112ae10e0e5b7c4589e107c433700ac
    bandersnatch_public = 4d5c855aebc996a88e818acd27d995f8e04c5dc49652c349f3f3204b39f1984a

    seed = trivial_seed(2) = 0200000002000000020000000200000002000000020000000200000002000000
    ed25519_secret_seed = 0093c8c10a88ebbc99b35b72897a26d259313ee9bad97436a437d2e43aaafa0f
    ed25519_public = cab2b9ff25c2410fbe9b8a717abb298c716a03983c98ceb4def2087500b8e341
    bandersnatch_secret_seed = 3d71dc0ffd02d90524fda3e4a220e7ec514a258c59457d3077ce4d4f003fd98a
    bandersnatch_public = 60a294af38653b7f0073767a0b9d10d8bf0efa6804782a9ecd17a9f5fd323b5b

    seed = trivial_seed(3) = 0300000003000000030000000300000003000000030000000300000003000000
    ed25519_secret_seed = 69b3a7031787e12bfbdcac1b7a737b3e5a9f9450c37e215f6d3b57730e21001a
    ed25519_public = f30aa5444688b3cab47697b37d5cac5707bb3289e986b19b17db437206931a8d
    bandersnatch_secret_seed = 107a9148b39a1099eeaee13ac0e3c6b9c256258b51c967747af0f8749398a276
    bandersnatch_public = 1e50be41fa601015847ae50ef0946ba5217027653d0ffd7a879c45ca21bab357

    seed = trivial_seed(4) = 0400000004000000040000000400000004000000040000000400000004000000
    ed25519_secret_seed = b4de9ebf8db5428930baa5a98d26679ab2a03eae7c791d582e6b75b7f018d0d4
    ed25519_public = 8b8c5d436f92ecf605421e873a99ec528761eb52a88a2f9a057b3b3003e6f32a
    bandersnatch_secret_seed = 0bb36f5ba8e3ba602781bb714e67182410440ce18aa800c4cb4dd22525b70409
    bandersnatch_public = b9b83ef77c1fda8f4b2909e531e6842bf26d4e6f69ed1405fc8d12d1350e6265

    seed = trivial_seed(5) = 0500000005000000050000000500000005000000050000000500000005000000
    ed25519_secret_seed = 4a6482f8f479e3ba2b845f8cef284f4b3208ba3241ed82caa1b5ce9fc6281730
    ed25519_public = ab0084d01534b31c1dd87c81645fd762482a90027754041ca1b56133d0466c06
    bandersnatch_secret_seed = 75e73b8364bf4753c5802021c6aa6548cddb63fe668e3cacf7b48cdb6824bb09
    bandersnatch_public = 7ef0acbb01d1ad5da7a1b576aef6ea9a91b979612ff339970c69fe0004d68e05

    seed = f92d680ea3f0ac06307795490d8a03c5c0d4572b5e0a8cffec87e1294855d9d1
    ed25519_secret_seed = f21e2d96a51387f9a7e5b90203654913dde7fa1044e3eba5631ed19f327d6126
    ed25519_public = 11a695f674de95ff3daaff9a5b88c18448b10156bf88bc04200e48d5155c7243
    bandersnatch_secret_seed = 06154d857537a9b622a9a94b1aeee7d588db912bfc914a8a9707148bfba3b9d1
    bandersnatch_public = 7273ec723f5f99e8ecb0d00417d0138eee582c1e0effbdfb38a425572d815009

## Gray Paper 0.8.0 and earlier

Gray Paper 0.8.0 and earlier releases refer to draft 29 or earlier of the Bandersnatch VRF-AD
Specification. With these releases, this document used appendix A.1 of draft 29: the secret scalar
is the SHA-512 hash of `bandersnatch_secret_seed`, read as a little-endian integer and reduced
modulo the order of the scalar field. That method gives different Bandersnatch public keys. The
other values do not change. The test vectors of that version of this document are at
<https://github.com/polkadot-fellows/JIPs/blob/7048f79edf4f4eb8bfe6fb42e6bbf61900f44c65/JIP-5.md>.
