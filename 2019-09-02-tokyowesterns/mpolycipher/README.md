# M-Poly-Cipher (re/crypto, 279p, 26 solved)

In the challenge we get [a binary](cipher) to reverse.
We also get a [public key](public.key) and [encrypted flat](flag.enc).
After hours of blood, sweat and tears we finally reach a [working python implementation](code.py).

The key part is:

```python
def encrypt(pt_file: str, pk_file: str, result_file: str) -> None:
    with open(pt_file, 'rb') as ctf:
        pt = ctf.read()
        pt = (pt + b'\x00' * (0x40 - len(pt)))[:0x40]
        pt = deserialise_vectors8(pt)

    with open(pk_file, 'rb') as pkf:
        pk = pkf.read()
        pk0 = parse_vectors(pk[0:256])
        pk1 = parse_vectors(pk[256:512])
        pk2 = parse_vectors(pk[512:768])

    seed = get_random_vectors()
    stage0 = matrix_mult(seed, pk0)
    stage1 = matrix_mult(seed, pk1)
    stage2 = matrix_mult(seed, pk2)
    combined = add_rows(stage2, pt)

    result = b''
    result += serialise_vectors32(stage0)
    result += serialise_vectors32(stage1)
    result += serialise_vectors32(combined)

    with open(result_file, 'wb') as outf:
        outf.write(result)
```

This shows what is the relation between public key we have and the encrypted output.
We didn't even try to understand the private key generation, we decided it will be far easier to simply reverse the encryption process.

It's quite clear that we need to invert `add_rows` operation on third output chunk, and this would give us plaintext.
In order to do that we need the `seed` vector.
We don't know it, however we know the `stage0` and `stage1` vectors.
And those are created from some matrix multiplication based on `pk0` and `pk2`, which we know!

So the idea is to invert the `matrix_mult` operation to recover the `seed`, and then use seed to invert `add_rows`.

What we got as matrix_mult is:

```python
def matrix_mult(data: List[int], key: List[int]) -> List[int]:
    result = []
    for j in range(8):
        for i in range(8):
            val = 0
            for k in range(8):
                val = (val + (data[j * 8 + k] * key[k * 8 + i])) % 0xFFFFFFFB
            result.append(val)
    return result
```

Which is not very readable, so we asked sage to show us the actual matrix:

```python
def matrix_mult(data, key):
    result = []
    for j in range(8):
        for i in range(8):
            val = 0
            for k in range(8):
                val = (val + (data[j * 8 + k] * key[k * 8 + i]))
            result.append(val)
    return result

seed_sym = [var('s' + str(i)) for i in range(64)]
pk_sym = [var('pk' + str(i)) for i in range(64)]

print(matrix_mult(seed_sym, pk_sym))
```

And we got back:

```
[pk0*s0 + pk8*s1 + pk16*s2 + pk24*s3 + pk32*s4 + pk40*s5 + pk48*s6 + pk56*s7,
 pk1*s0 + pk9*s1 + pk17*s2 + pk25*s3 + pk33*s4 + pk41*s5 + pk49*s6 + pk57*s7,
 pk2*s0 + pk10*s1 + pk18*s2 + pk26*s3 + pk34*s4 + pk42*s5 + pk50*s6 + pk58*s7,
 pk3*s0 + pk11*s1 + pk19*s2 + pk27*s3 + pk35*s4 + pk43*s5 + pk51*s6 + pk59*s7,
 pk4*s0 + pk12*s1 + pk20*s2 + pk28*s3 + pk36*s4 + pk44*s5 + pk52*s6 + pk60*s7,
 pk5*s0 + pk13*s1 + pk21*s2 + pk29*s3 + pk37*s4 + pk45*s5 + pk53*s6 + pk61*s7,
 pk6*s0 + pk14*s1 + pk22*s2 + pk30*s3 + pk38*s4 + pk46*s5 + pk54*s6 + pk62*s7,
 pk7*s0 + pk15*s1 + pk23*s2 + pk31*s3 + pk39*s4 + pk47*s5 + pk55*s6 + pk63*s7,
 pk16*s10 + pk24*s11 + pk32*s12 + pk40*s13 + pk48*s14 + pk56*s15 + pk0*s8 + pk8*s9,
 pk17*s10 + pk25*s11 + pk33*s12 + pk41*s13 + pk49*s14 + pk57*s15 + pk1*s8 + pk9*s9,
 pk18*s10 + pk26*s11 + pk34*s12 + pk42*s13 + pk50*s14 + pk58*s15 + pk2*s8 + pk10*s9,
 pk19*s10 + pk27*s11 + pk35*s12 + pk43*s13 + pk51*s14 + pk59*s15 + pk3*s8 + pk11*s9,
 pk20*s10 + pk28*s11 + pk36*s12 + pk44*s13 + pk52*s14 + pk60*s15 + pk4*s8 + pk12*s9,
 pk21*s10 + pk29*s11 + pk37*s12 + pk45*s13 + pk53*s14 + pk61*s15 + pk5*s8 + pk13*s9,
 pk22*s10 + pk30*s11 + pk38*s12 + pk46*s13 + pk54*s14 + pk62*s15 + pk6*s8 + pk14*s9,
 pk23*s10 + pk31*s11 + pk39*s12 + pk47*s13 + pk55*s14 + pk63*s15 + pk7*s8 + pk15*s9,
 pk0*s16 + pk8*s17 + pk16*s18 + pk24*s19 + pk32*s20 + pk40*s21 + pk48*s22 + pk56*s23,
 pk1*s16 + pk9*s17 + pk17*s18 + pk25*s19 + pk33*s20 + pk41*s21 + pk49*s22 + pk57*s23,
 pk2*s16 + pk10*s17 + pk18*s18 + pk26*s19 + pk34*s20 + pk42*s21 + pk50*s22 + pk58*s23,
 pk3*s16 + pk11*s17 + pk19*s18 + pk27*s19 + pk35*s20 + pk43*s21 + pk51*s22 + pk59*s23,
 pk4*s16 + pk12*s17 + pk20*s18 + pk28*s19 + pk36*s20 + pk44*s21 + pk52*s22 + pk60*s23,
 pk5*s16 + pk13*s17 + pk21*s18 + pk29*s19 + pk37*s20 + pk45*s21 + pk53*s22 + pk61*s23,
 pk6*s16 + pk14*s17 + pk22*s18 + pk30*s19 + pk38*s20 + pk46*s21 + pk54*s22 + pk62*s23,
 pk7*s16 + pk15*s17 + pk23*s18 + pk31*s19 + pk39*s20 + pk47*s21 + pk55*s22 + pk63*s23,
 pk0*s24 + pk8*s25 + pk16*s26 + pk24*s27 + pk32*s28 + pk40*s29 + pk48*s30 + pk56*s31,
 pk1*s24 + pk9*s25 + pk17*s26 + pk25*s27 + pk33*s28 + pk41*s29 + pk49*s30 + pk57*s31,
 pk2*s24 + pk10*s25 + pk18*s26 + pk26*s27 + pk34*s28 + pk42*s29 + pk50*s30 + pk58*s31,
 pk3*s24 + pk11*s25 + pk19*s26 + pk27*s27 + pk35*s28 + pk43*s29 + pk51*s30 + pk59*s31,
 pk4*s24 + pk12*s25 + pk20*s26 + pk28*s27 + pk36*s28 + pk44*s29 + pk52*s30 + pk60*s31,
 pk5*s24 + pk13*s25 + pk21*s26 + pk29*s27 + pk37*s28 + pk45*s29 + pk53*s30 + pk61*s31,
 pk6*s24 + pk14*s25 + pk22*s26 + pk30*s27 + pk38*s28 + pk46*s29 + pk54*s30 + pk62*s31,
 pk7*s24 + pk15*s25 + pk23*s26 + pk31*s27 + pk39*s28 + pk47*s29 + pk55*s30 + pk63*s31,
 pk0*s32 + pk8*s33 + pk16*s34 + pk24*s35 + pk32*s36 + pk40*s37 + pk48*s38 + pk56*s39,
 pk1*s32 + pk9*s33 + pk17*s34 + pk25*s35 + pk33*s36 + pk41*s37 + pk49*s38 + pk57*s39,
 pk2*s32 + pk10*s33 + pk18*s34 + pk26*s35 + pk34*s36 + pk42*s37 + pk50*s38 + pk58*s39,
 pk3*s32 + pk11*s33 + pk19*s34 + pk27*s35 + pk35*s36 + pk43*s37 + pk51*s38 + pk59*s39,
 pk4*s32 + pk12*s33 + pk20*s34 + pk28*s35 + pk36*s36 + pk44*s37 + pk52*s38 + pk60*s39,
 pk5*s32 + pk13*s33 + pk21*s34 + pk29*s35 + pk37*s36 + pk45*s37 + pk53*s38 + pk61*s39,
 pk6*s32 + pk14*s33 + pk22*s34 + pk30*s35 + pk38*s36 + pk46*s37 + pk54*s38 + pk62*s39,
 pk7*s32 + pk15*s33 + pk23*s34 + pk31*s35 + pk39*s36 + pk47*s37 + pk55*s38 + pk63*s39,
 pk0*s40 + pk8*s41 + pk16*s42 + pk24*s43 + pk32*s44 + pk40*s45 + pk48*s46 + pk56*s47,
 pk1*s40 + pk9*s41 + pk17*s42 + pk25*s43 + pk33*s44 + pk41*s45 + pk49*s46 + pk57*s47,
 pk2*s40 + pk10*s41 + pk18*s42 + pk26*s43 + pk34*s44 + pk42*s45 + pk50*s46 + pk58*s47,
 pk3*s40 + pk11*s41 + pk19*s42 + pk27*s43 + pk35*s44 + pk43*s45 + pk51*s46 + pk59*s47,
 pk4*s40 + pk12*s41 + pk20*s42 + pk28*s43 + pk36*s44 + pk44*s45 + pk52*s46 + pk60*s47,
 pk5*s40 + pk13*s41 + pk21*s42 + pk29*s43 + pk37*s44 + pk45*s45 + pk53*s46 + pk61*s47,
 pk6*s40 + pk14*s41 + pk22*s42 + pk30*s43 + pk38*s44 + pk46*s45 + pk54*s46 + pk62*s47,
 pk7*s40 + pk15*s41 + pk23*s42 + pk31*s43 + pk39*s44 + pk47*s45 + pk55*s46 + pk63*s47,
 pk0*s48 + pk8*s49 + pk16*s50 + pk24*s51 + pk32*s52 + pk40*s53 + pk48*s54 + pk56*s55,
 pk1*s48 + pk9*s49 + pk17*s50 + pk25*s51 + pk33*s52 + pk41*s53 + pk49*s54 + pk57*s55,
 pk2*s48 + pk10*s49 + pk18*s50 + pk26*s51 + pk34*s52 + pk42*s53 + pk50*s54 + pk58*s55,
 pk3*s48 + pk11*s49 + pk19*s50 + pk27*s51 + pk35*s52 + pk43*s53 + pk51*s54 + pk59*s55,
 pk4*s48 + pk12*s49 + pk20*s50 + pk28*s51 + pk36*s52 + pk44*s53 + pk52*s54 + pk60*s55,
 pk5*s48 + pk13*s49 + pk21*s50 + pk29*s51 + pk37*s52 + pk45*s53 + pk53*s54 + pk61*s55,
 pk6*s48 + pk14*s49 + pk22*s50 + pk30*s51 + pk38*s52 + pk46*s53 + pk54*s54 + pk62*s55,
 pk7*s48 + pk15*s49 + pk23*s50 + pk31*s51 + pk39*s52 + pk47*s53 + pk55*s54 + pk63*s55,
 pk0*s56 + pk8*s57 + pk16*s58 + pk24*s59 + pk32*s60 + pk40*s61 + pk48*s62 + pk56*s63,
 pk1*s56 + pk9*s57 + pk17*s58 + pk25*s59 + pk33*s60 + pk41*s61 + pk49*s62 + pk57*s63,
 pk2*s56 + pk10*s57 + pk18*s58 + pk26*s59 + pk34*s60 + pk42*s61 + pk50*s62 + pk58*s63,
 pk3*s56 + pk11*s57 + pk19*s58 + pk27*s59 + pk35*s60 + pk43*s61 + pk51*s62 + pk59*s63,
 pk4*s56 + pk12*s57 + pk20*s58 + pk28*s59 + pk36*s60 + pk44*s61 + pk52*s62 + pk60*s63,
 pk5*s56 + pk13*s57 + pk21*s58 + pk29*s59 + pk37*s60 + pk45*s61 + pk53*s62 + pk61*s63,
 pk6*s56 + pk14*s57 + pk22*s58 + pk30*s59 + pk38*s60 + pk46*s61 + pk54*s62 + pk62*s63,
 pk7*s56 + pk15*s57 + pk23*s58 + pk31*s59 + pk39*s60 + pk47*s61 + pk55*s62 + pk63*s63]
```

This is how `stage0` and `stage1` are calculated from `pk0` and `pk1` combined with `seed`.
Now we want to find such matrix `M0` that `M0*seed = stage0`.

The matrix we want is:

```
[pk0,pk8,pk16,pk24,pk32,pk40,pk48,pk56,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[pk1,pk9,pk17,pk25,pk33,pk41,pk49,pk57,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[pk2,pk10,pk18,pk26,pk34,pk42,pk50,pk58,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[pk3,pk11,pk19,pk27,pk35,pk43,pk51,pk59,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[pk4,pk12,pk20,pk28,pk36,pk44,pk52,pk60,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[pk5,pk13,pk21,pk29,pk37,pk45,pk53,pk61,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[pk6,pk14,pk22,pk30,pk38,pk46,pk54,pk62,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[pk7,pk15,pk23,pk31,pk39,pk47,pk55,pk63,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,pk0,pk8,pk16,pk24,pk32,pk40,pk48,pk56,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,pk1,pk9,pk17,pk25,pk33,pk41,pk49,pk57,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,pk2,pk10,pk18,pk26,pk34,pk42,pk50,pk58,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,pk3,pk11,pk19,pk27,pk35,pk43,pk51,pk59,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,pk4,pk12,pk20,pk28,pk36,pk44,pk52,pk60,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,pk5,pk13,pk21,pk29,pk37,pk45,pk53,pk61,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,pk6,pk14,pk22,pk30,pk38,pk46,pk54,pk62,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,pk7,pk15,pk23,pk31,pk39,pk47,pk55,pk63,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk0,pk8,pk16,pk24,pk32,pk40,pk48,pk56,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk1,pk9,pk17,pk25,pk33,pk41,pk49,pk57,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk2,pk10,pk18,pk26,pk34,pk42,pk50,pk58,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk3,pk11,pk19,pk27,pk35,pk43,pk51,pk59,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk4,pk12,pk20,pk28,pk36,pk44,pk52,pk60,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk5,pk13,pk21,pk29,pk37,pk45,pk53,pk61,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk6,pk14,pk22,pk30,pk38,pk46,pk54,pk62,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk7,pk15,pk23,pk31,pk39,pk47,pk55,pk63,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk0,pk8,pk16,pk24,pk32,pk40,pk48,pk56,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk1,pk9,pk17,pk25,pk33,pk41,pk49,pk57,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk2,pk10,pk18,pk26,pk34,pk42,pk50,pk58,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk3,pk11,pk19,pk27,pk35,pk43,pk51,pk59,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk4,pk12,pk20,pk28,pk36,pk44,pk52,pk60,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk5,pk13,pk21,pk29,pk37,pk45,pk53,pk61,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk6,pk14,pk22,pk30,pk38,pk46,pk54,pk62,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk7,pk15,pk23,pk31,pk39,pk47,pk55,pk63,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk0,pk8,pk16,pk24,pk32,pk40,pk48,pk56,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk1,pk9,pk17,pk25,pk33,pk41,pk49,pk57,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk2,pk10,pk18,pk26,pk34,pk42,pk50,pk58,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk3,pk11,pk19,pk27,pk35,pk43,pk51,pk59,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk4,pk12,pk20,pk28,pk36,pk44,pk52,pk60,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk5,pk13,pk21,pk29,pk37,pk45,pk53,pk61,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk6,pk14,pk22,pk30,pk38,pk46,pk54,pk62,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk7,pk15,pk23,pk31,pk39,pk47,pk55,pk63,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk0,pk8,pk16,pk24,pk32,pk40,pk48,pk56,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk1,pk9,pk17,pk25,pk33,pk41,pk49,pk57,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk2,pk10,pk18,pk26,pk34,pk42,pk50,pk58,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk3,pk11,pk19,pk27,pk35,pk43,pk51,pk59,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk4,pk12,pk20,pk28,pk36,pk44,pk52,pk60,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk5,pk13,pk21,pk29,pk37,pk45,pk53,pk61,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk6,pk14,pk22,pk30,pk38,pk46,pk54,pk62,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk7,pk15,pk23,pk31,pk39,pk47,pk55,pk63,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk0,pk8,pk16,pk24,pk32,pk40,pk48,pk56,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk1,pk9,pk17,pk25,pk33,pk41,pk49,pk57,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk2,pk10,pk18,pk26,pk34,pk42,pk50,pk58,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk3,pk11,pk19,pk27,pk35,pk43,pk51,pk59,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk4,pk12,pk20,pk28,pk36,pk44,pk52,pk60,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk5,pk13,pk21,pk29,pk37,pk45,pk53,pk61,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk6,pk14,pk22,pk30,pk38,pk46,pk54,pk62,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk7,pk15,pk23,pk31,pk39,pk47,pk55,pk63,0,0,0,0,0,0,0,0]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk0,pk8,pk16,pk24,pk32,pk40,pk48,pk56]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk1,pk9,pk17,pk25,pk33,pk41,pk49,pk57]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk2,pk10,pk18,pk26,pk34,pk42,pk50,pk58]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk3,pk11,pk19,pk27,pk35,pk43,pk51,pk59]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk4,pk12,pk20,pk28,pk36,pk44,pk52,pk60]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk5,pk13,pk21,pk29,pk37,pk45,pk53,pk61]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk6,pk14,pk22,pk30,pk38,pk46,pk54,pk62]
[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,pk7,pk15,pk23,pk31,pk39,pk47,pk55,pk63]
```

We come up with:

```python
def matrix_mult2(seed, pk):
    pkm = Matrix(Zmod(mod), [[0 for i in range(8*k)]+[pk[i] for i in range(j,64,8)]+[0 for i in range(8*(8-k-1))] for k in range(8) for j in range(8)])
    return list(pkm*vector(seed))
```

And it works exactly as we wanted.
Now we could just do:

```python
def recover_seed(result, pk):
    pkm = Matrix(Zmod(mod), [[0 for i in range(8*k)]+[pk[i] for i in range(j,64,8)]+[0 for i in range(8*(8-k-1))] for k in range(8) for j in range(8)])
    result_matrix = pkm.solve_right(vector(result))
    return list(result_matrix)
```

In order to recover the seed.
But it turned out the results are ambigious and for `pk0` and `stage0` we get different results than for `pk1` and `stage1`.
Fortunately we can combine those equations in a single system and solve at once, hopefully getting a single result:

```python
def recover_seed2(stage0, stage1, pk0, pk1):
    m1 = [[0 for i in range(8*k)]+[pk0[i] for i in range(j,64,8)]+[0 for i in range(8*(8-k-1))] for k in range(8) for j in range(8)]
    m2 = [[0 for i in range(8*k)]+[pk1[i] for i in range(j,64,8)]+[0 for i in range(8*(8-k-1))] for k in range(8) for j in range(8)]
    m = m1+m2
    pkm = Matrix(Zmod(mod), m)
    result_matrix = pkm.solve_right(vector(stage0+stage1))
    return list(result_matrix)
```

And from this we actually manage to recover the `seed`:

```
[3437097476, 1872232232, 3647344144, 3900179940, 289303261, 1125306664, 1781119250, 2685999413, 2926201689, 1794057147, 3762873198, 518522290, 3146643550, 2401122808, 2576451253, 4054234528, 2639110757, 2257625570, 2726372255, 909523980, 3279957714, 11808025, 2748448837, 3248636903, 1862461456, 2863118810, 2029738056, 204000072, 1150709971, 1366849197, 3380682274, 4048488032, 561673885, 2638095422, 604494451, 3029286890, 2174284642, 1281120732, 4271766672, 1413542622, 1380470061, 336824337, 4227867279, 49513556, 3316972952, 158722238, 2577376715, 1836198972, 1517374624, 2154122694, 2020093710, 3727061870, 1719521710, 4187510087, 4057609046, 3434783742, 1108797172, 61803915, 4134164703, 989888949, 2202917742, 2375475319, 626659464, 3913729267]
```

Now we can just calculate `stage2` by `stage2 = matrix_mult(seed, pk2)` and then invert `add_rows(stage2, pt)`:

```python
    recovered_seed = recover_seed2(stage0, stage1, pk0, pk1)    
    print('seed', recovered_seed)
    stage2 = matrix_mult(recovered_seed, pk2)
    print('stage2', stage2)
    
    ptr = sub_rows(combined, stage2)
    print("".join(map(chr,ptr)))
```

And from this we get `TWCTF{pa+h_t0_tomorr0w}`

Complete solver [here](solver.sage)