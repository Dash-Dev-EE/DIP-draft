<pre>
DIP: 0011
Title: Activating old and implementing new opcodes
Author: Mart Mangus
Status: Draft
Type: Standard
Layer: Consensus (hard fork)
Created: 2020-06-01
License: MIT License
</pre>

# Abstract
<!-- A short (~200 word) description of the technical issue being addressed -->

This DIP describes reactivation of a disabled opcode (`OP_CAT`) and activation of new opcodes (`OP_SPLIT`, `OP_BIN2NUM`, `OP_NUM2BIN`, `OP_CHECKDATASIG` and `OP_CHECKDATASIGVERIFY`) to expand the use of Dash scripting system for transactions and enable the creation of user friendly trustless smart card payment system.


# Motivation
<!-- The motivation is critical for BIPs that want to change the Bitcoin protocol. It should clearly explain why the existing protocol is inadequate to address the problem that the BIP solves -->

Several opcodes were disabled in the Bitcoin scripting system due to discovery of series of bugs in early days of Bitcoin (2010-2011). The functionality of these opcodes has been re-examined by Bitcoin Cash developers and some of the disabled opcodes has been enabled, as well as re-designed ones to replace few original ones. In addition, they have implemented couple of completely new opcodes which will further expand the use of Bitcoin scripting system.

One use case is in trustless smart card payment system where the use of these new opcodes takes away the need for on card UTXO (unspent transaction output) management and makes these cards more usable and user friendly.

# Specification
<!-- The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Bitcoin platforms -->

|Word                  |OpCode |Hex |Input |Output  | Description                                           |
|----------------------|-------|----|------|--------|-------------------------------------------------------|
|OP_CAT                |126    |0x7e|x1 x2 |out     |Concatenates two byte arrays                           |
|OP_SPLIT              |127    |0x7f|x n   |x1 x2   |Split byte array *x* at position *n*                   |
|OP_NUM2BIN            |128    |0x80|a b   |out     |Convert numeric *a* into byte array of length *b*      |
|OP_BIN2NUM            |129    |0x81|x     |out     |Convert byte array *x* into numeric                    |
|OP_CHECKDATASIG       |186    |0xba|sig&nbsp;msg&nbsp;pk|out     |If signature *sig* is valid, output to the stack          |
|OP_CHECKDATASIGVERIFY |187    |0xbb|sig&nbsp;msg&nbsp;pk|        |If signature *sig* is valid, false will cause the script to fail|

## OP_CAT
`OP_CAT` takes two byte arrays from the stack, concates and pushes the result back to the stack.


`x1 x2 OP_CAT → out`


Example:
* `0x11 0x2233 OP_CAT → 0x112233`

The operator must fail if:
* `len(out) > MAX_SCRIPT_ELEMENT_SIZE`

## OP_SPLIT
`OP_SPLIT` inverse of `OP_CAT` and is a replacement operation for disabled opcode `OP_SUBSTR`, `OP_LEFT` and `OP_RIGHT`.

`OP_SPLIT` takes a byte array, splits it at the position `n` (a number) and returns

`x n OP_SPLIT → x1 x2`


Examples:
* `0x001122 0 OP_SPLIT → OP_0 0x001122`
* `0x001122 1 OP_SPLIT → 0x00 0x1122`
* `0x001122 2 OP_SPLIT → 0x0011 0x22`
* `0x001122 3 OP_SPLIT → 0x001122 OP_0`


`x` is split at position `n`, where `n` is the number of bytes from the beginning
`x1` will be the first `n` bytes of `x` and `x2` will be the remaining bytes
if `n == 0`, then `x1` is the empty array and `x2 == x`
if `n == len(x)` then `x1 == x` and `x2` is the empty array.
if `n > len(x)`, then the operator must fail.

The operator must fail if:
* `!isnum(''n'')`. Fail if `n` is not a number.
* `n < 0`. Fail if `n` is negative.
* `n > len(x)`. Fail if `n` is too high.


## OP_NUM2BIN
`OP_NUM2BIN` converts numeric value `n` to a byte array of length `m`.

`n m OP_NUM2BIN → x`


Examples:
* `0x02 4 OP_NUM2BIN → 0x00000002`
* `0x85 4 OP_NUM2BIN → 0x80000005`


The operator must fail if:
* `n` or `m` are not valid numeric values.
* `m < len(n)`. `n` is a valid numeric value, therefore it must already be in minimal representation, so it cannot fit into a byte array which is smaller than the length of `n`.
* `m` > `MAX_SCRIPT_ELEMENT_SIZE`. The result would be too large.

## OP_BIN2NUM
`OP_BIN2NUM` converts byte array value `x` into a numeric value.

`x1 OP_BIN2NUM → n`

if `x1` is any form of zero, including negative zero, then `OP_0` must be the result

Examples:
* `0x0000000002 OP_BIN2NUM → 0x02`
* `0x800005 OP_BIN2NUM → 0x85`

The operator must fail if:
* the numeric value is out of the range of acceptable numeric values

## OP_CHECKDATASIG and OP_CHECKDATASIGVERIFY
`OP_CHECKDATASIG` and `OP_CHECKDATASIGVERIFY` check whether a signature is valid with respect to a message and a public key. 

The operator must fail if stack is not well formed. To be well formed, the stack must contain at least three elements
[`sig`, `msg`, `pubKey`]
in this order where `pubKey` is the top element and
* `pubKey` must be a validly encoded public key
* `msg` can be any string
* `sig` must follow the strict DER encoding and the S-value of `sig` must be at most the curve order divided by 2

If the stack is well formed, then `OP_CHECKDATASIG` pops the top three elements [`sig`, `msg`, `pubKey`] from the stack and pushes true onto the stack if `sig` is valid with respect to the raw single-SHA256 hash of `msg` and `pubKey` using the secp256k1 elliptic curve. Otherwise, it pops three elements and pushes false onto the stack in the case that `sig` is the empty string and fails in all other cases.

`OP_CHECKDATASIGVERIFY` is equivalent to `OP_CHECKDATASIG` followed by `OP_VERIFY`. It leaves nothing on the stack, and will cause the script to fail immediately if the signature check does not pass.

# Compatibility
<!-- All BIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The BIP must explain how the author proposes to deal with these incompatibilities -->

This change will be a hard-fork to the protocol and older software has to be updated to continue to operate. 

# References
<!-- The reference implementation must be completed before any BIP is given status "Final", but it need not be completed before the BIP is accepted. It is better to finish the specification and rationale first and reach consensus on it before writing code. The final implementation must include test code and documentation appropriate for the Bitcoin protocol -->

* Bitcoin Cash specification of restored disabled opcodes: https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/may-2018-reenabled-opcodes.md
* Bitcoin Cash specification of OP_CHECKDATASIG and OP_CHECKDATASIGVERIFY: https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/op_checkdatasig.md
* Bitcoin Cash OP_CHECKDATASIG and OP_CHECKDATASIGVERIFY implementation: https://reviews.bitcoinabc.org/D1621 https://reviews.bitcoinabc.org/D1646 https://reviews.bitcoinabc.org/D1653 https://reviews.bitcoinabc.org/D1666
* Bitcoin Cash OP_CAT implementation: https://reviews.bitcoinabc.org/D1097
* Bitcoin Cash OP_SPLIT implementation: https://reviews.bitcoinabc.org/D1099
* Bitcoin Cash OP_BIN2NUM implementation: https://reviews.bitcoinabc.org/D1101
* Bitcoin Cash OP_NUM2BIN implementation: https://reviews.bitcoinabc.org/D1103

# Copyright
<!-- The BIP must be explicitly licensed under acceptable copyright terms -->
This document is licensed under the [MIT License](https://opensource.org/licenses/MIT).
