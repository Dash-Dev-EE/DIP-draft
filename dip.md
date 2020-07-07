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
This DIP describes reactivation of a disabled opcode (`OP_CAT`, `OP_AND`, `OP_OR`, `OP_XOR`, `OP_DIV`, `OP_MOD`) and activation of new opcodes (`OP_SPLIT`, `OP_BIN2NUM`, `OP_NUM2BIN`, `OP_CHECKDATASIG` and `OP_CHECKDATASIGVERIFY`) to expand the use of Dash scripting system for transactions.

# Motivation
Several opcodes were disabled in the Bitcoin scripting system due to discovery of series of bugs in early days of Bitcoin (2010-2011). The functionality of these opcodes has been re-examined by Bitcoin Cash developers few years ago and many of the disabled opcodes has been enabled, and few of them were re-designed to replace the original ones. In addition, they have implemented couple of completely new opcodes which will further expand the use of Bitcoin scripting system and enables to build new solutions.

One use case is a trustless smart card payment system where the use of these new opcodes removes the need for on card UTXO (unspent transaction output) management and results in more reliable and faster system.

# Specification
|Word                  |OpCode |Hex |Input |Output  | Description                                           |
|----------------------|-------|----|------|--------|-------------------------------------------------------|
|OP_CAT                |126    |0x7e|x1 x2 |out     |Concatenates two byte arrays                           |
|OP_SPLIT              |127    |0x7f|x n   |x1 x2   |Split byte array *x* at position *n*                   |
|OP_NUM2BIN            |128    |0x80|a b   |out     |Convert numeric *a* into byte array of length *b*      |
|OP_BIN2NUM            |129    |0x81|x     |out     |Convert byte array *x* into numeric                    |
|OP_AND                |132    |0x84|x1 x2 |out     |Boolean *AND* between each bit of the inputs           |
|OP_OR                 |133    |0x85|x1 x2 |out     |Boolean *OR* between each bit of the inputs            |
|OP_XOR                |134    |0x86|x1 x2 |out     |Boolean *EXCLUSIVE OR* between each bit of the inputs  |
|OP_DIV                |150    |0x96|a b   |out     |*a* is divided by *b*                                  |
|OP_MOD                |151    |0x97|a b   |out     |return the remainder after *a* is divided by *b*       |
|OP_CHECKDATASIG       |186    |0xba|sig&nbsp;msg&nbsp;pk|out     |If signature *sig* is valid, output to the stack|
|OP_CHECKDATASIGVERIFY |187    |0xbb|sig&nbsp;msg&nbsp;pk|-       |If signature *sig* is valid, false will cause the script to fail|

## OP_CAT
`OP_CAT` takes two byte arrays from the stack, concates and pushes the result back to the stack.


`x1 x2 OP_CAT → out`


Example:
* `0x11 0x2233 OP_CAT → 0x112233`

The operator must fail if:
* `len(out) > MAX_SCRIPT_ELEMENT_SIZE`.

## OP_SPLIT
`OP_SPLIT` is inverse of `OP_CAT` and a replacement operation for disabled opcodes `OP_SUBSTR`, `OP_LEFT` and `OP_RIGHT`.

`OP_SPLIT` takes a byte array, splits it at the position `n` (a number) and returns two byte arrays.

`x n OP_SPLIT → x1 x2`


Examples:
* `0x001122 0 OP_SPLIT → OP_0 0x001122`
* `0x001122 1 OP_SPLIT → 0x00 0x1122`
* `0x001122 2 OP_SPLIT → 0x0011 0x22`
* `0x001122 3 OP_SPLIT → 0x001122 OP_0`


Notes:
* `x` is split at position `n`, where `n` is the number of bytes from the beginning
* `x1` will be the first `n` bytes of `x` and `x2` will be the remaining bytes 
* if `n == 0`, then `x1` is the empty array and `x2 == x`
* if `n == len(x)` then `x1 == x` and `x2` is the empty array.
* if `n > len(x)`, then the operator must fail.

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

if `x1` is any form of zero, including negative zero, then `OP_0` must be the result.

Examples:
* `0x0000000002 OP_BIN2NUM → 0x02`
* `0x800005 OP_BIN2NUM → 0x85`

The operator must fail if:
* the numeric value is out of the range of acceptable numeric values.


## OP_CHECKDATASIG
`OP_CHECKDATASIG` checks whether a signature is valid with respect to a message and a public key. It allows Script to validate arbitrary messages from outside the blockchain.

`sig msg pubKey OP_CHECKDATASIG → out`

If the stack is well formed, then `OP_CHECKDATASIG` pops the top three elements [`sig`, `msg`, `pubKey`] from the stack and pushes true onto the stack if `sig` is valid with respect to the raw single-SHA256 hash of `msg` and `pubKey` using the secp256k1 elliptic curve. Otherwise, it pops three elements and pushes false onto the stack in the case that `sig` is the empty string and fails in all other cases.

The operator must fail if:
* Stack is not well formed. To be well formed, the stack must contain at least three elements
[`sig`, `msg`, `pubKey`]
in this order where `pubKey` is the top element and
  * `pubKey` must be a validly encoded public key
  * `msg` can be any string
  * `sig` must follow the strict DER encoding and the S-value of `sig` must be at most the curve order divided by 2.


## OP_CHECKDATASIGVERIFY
`OP_CHECKDATASIGVERIFY` is equivalent to `OP_CHECKDATASIG` followed by `OP_VERIFY`. It leaves nothing on the stack, and will cause the script to fail immediately if the signature check does not pass.


# Compatibility
This change will be a hard-fork to the protocol and older software has to be updated to continue to operate. 

# References
* Bitcoin Cash specification of restored disabled opcodes: https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/may-2018-reenabled-opcodes.md
* Bitcoin Cash specification of OP_CHECKDATASIG and OP_CHECKDATASIGVERIFY: https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/op_checkdatasig.md
* Bitcoin Cash OP_CHECKDATASIG and OP_CHECKDATASIGVERIFY implementation: https://reviews.bitcoinabc.org/D1621 https://reviews.bitcoinabc.org/D1646 https://reviews.bitcoinabc.org/D1653 https://reviews.bitcoinabc.org/D1666
* Bitcoin Cash OP_CAT implementation: https://reviews.bitcoinabc.org/D1097
* Bitcoin Cash OP_SPLIT implementation: https://reviews.bitcoinabc.org/D1099
* Bitcoin Cash OP_BIN2NUM implementation: https://reviews.bitcoinabc.org/D1101
* Bitcoin Cash OP_NUM2BIN implementation: https://reviews.bitcoinabc.org/D1103

# Copyright
This document is licensed under the [MIT License](https://opensource.org/licenses/MIT).
