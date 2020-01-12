# Implementing SHA-1 in Javascript

## Why?

I was reading the [git pro book](https://git-scm.com/book/en/v2). And they have a really cool section [here](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects#_object_storage) that teaches you to implement a small piece of Ruby code that allows you to add an object to git's object store. For me that was one of the coolest things in the book. But since I do not know any Ruby I decided to follow the code in Javascript (// TODO: add link to the javascript code here). And it was pretty awesome to be able to tap into the inner workings of git without using any external libraries. Well, I had to use two node modules, [crypto](https://nodejs.org/api/crypto.html#crypto_crypto_createhash_algorithm_options) and [zlib](https://nodejs.org/api/zlib.html). Both are modules that come within node, so you do not need to npm install any extra packages or anything, but I would like to really do everything myself. The crypto module was used to generate the [SHA-1](https://en.wikipedia.org/wiki/SHA-1) hash digest (I will explain what that means later) of the contents I want to store in git, and zlib was used to compress my content using the deflate algorithm and save it to gits Object store, which is basically just a bunch of files. By the way, the git Object store is a lot less scary when you finish reading the Git Pro book. I really recommend it.

I also had an excuse that maybe if I wanted, for some reason, to run my code in the browser maybe those two modules would not be available. But the real reason is just that I wanted to see how things worked from inside. And later I even found that the web has this [digest](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/digest) function that can create a SHA-1 hash for you, and it even has a good browser support. But that does not really matter because all I want is to understand how things work. And the best way of doing that for me is by implementing it myself.

## What is SHA-1?

Before we go to the implementation I wanted just to explain what a hash is and some of the features of the SHA-1 hashing algorithm.

Well, a hash is a fixed length string of data, basically a bunch of bits (zeroes and ones), in SHA-1 specifically 160, that are calculated from a particular message. That message can be a file, a number a string, basically anything the computer can handle as bits. And the idea is that if you pass say a file to a hashing algorithm multiple times it always give you back the same hash. But if you change even a single bit in the file the hash of it will be completely different.

A good and secure hashing algorithm should have two main properties:

- It should provide a hash in such a way that is nearly impossible to work out what the original message was from the hash and algorithm alone.
- It should be nearly impossible to find more than one message that produces the same hash.

SHA stands for Secure Hashing Algorithm, and there different versions of the algorithm that produce differently sized hashes. We are going to use the official spec [here](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.180-4.pdf) during our implementation.

I the [spec](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.180-4.pdf) the actually have a really good explanation about a hash algorithm is. I think it is even better than my explanation above so I will quote it here:

> This Standard specifies secure hash algorithms - SHA-1, SHA-224, SHA-256,
> SHA-384, SHA-512, SHA-512/224 and SHA-512/256 - for computing a condensed
> representation of electronic data (message). When a message of any length less than 264 bits (for
> SHA-1, SHA-224 and SHA-256) or less than 2128 bits (for SHA-384, SHA-512, SHA-512/224
> and SHA-512/256) is input to a hash algorithm, the result is an output called a message digest.
> The message digests range in length from 160 to 512 bits, depending on the algorithm. Secure
> hash algorithms are typically used with other cryptographic algorithms, such as digital signature
> algorithms and keyed-hash message authentication codes, or in the generation of random
> numbers (bits).
>
> The hash algorithms specified in this Standard are called secure because, for a given algorithm, it
> is computationally infeasible
>
> 1. to find a message that corresponds to a given message digest, or
>
> 2. to find two different messages that produce the same message digest. Any change to a
>    message will, with a very high probability, result in a different message digest. This will result in
>    a verification failure when the secure hash algorithm is used with a digital signature algorithm or
>    a keyed-hash message authentication algorithm.

If you have the time you should really go there and read the docs, they are not difficult to understand, and will go into a lot more detail that you may not find here.

## Let's implement it then

Ok, let's get started.

The code that I am writing is supposed to run on Node.js, but should run in the browser with a few modifications, basically just removing the requires.

What I want is a function `sha1` to which I can pass a string and get a string with the hexadecimal representation of the SHA-1 hash of that string. And when I was writing the code I just used the node module crypto to check if my calculated hash would be right. Something like this:

```js
const crypto = require("crypto");
const assert = require("assert");

const message = "abc";

const cryptosResult = crypto
  .createHash("sha1")
  .update(message)
  .digest("hex");

// We still have to write the sha1 function
const myResult = sha1(message);

assert.equal(myResult, cryptosResult);
console.log("Success!");
```

Using "abc" as our testing message is good because the spec gives you a [debugging sheet](https://csrc.nist.gov/CSRC/media/Projects/Cryptographic-Standards-and-Guidelines/documents/examples/SHA1.pdf) that really helped me see where in my code I was going wrong.

Now we have a way of checking the correctness of our implementation so we just have to implement the sha1 function an that is it.

### Preparing the message

#### Converting to binary

SHA-1 calculates the hash based on the bits of our message. So if our message is not in binary form already we should transform it to binary first.
Since we want to pass a string as our message input, we need to transform our string message into raw bits.

In javascript the most basic way of representing raw binary data is an [ArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer). So are going to user the [TextEncoder](https://nodejs.org/api/util.html#util_class_util_textencoder) that also exists in the browser ([see here](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder/encode)), to encode our string in a [Uint8Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array) which is just a view on top a an ArrayBuffer. And we can access the buffer by accessing the `.buffer` property of the Uint8Array. Our `sha1` function starts like this:

```js
const TextEncoder = require("util").TextEncoder;

function sha1(message) {
  const buffer = new TextEncoder().encode(message).buffer;

  // More code will come here
}
```

#### Padding the message

As it says in the spec on section 5.1.1. We need to make sure that the total number of bits in our message is a multiple of 512. This is because the algorithm processes the message in blocks of 512 bit at a time. So in order to do that we have to complete our message with smallest quantity of zeroes at the end until the length in bits is a multiple of 512. But, before adding zeroes we add a 1 at the end and then the zeroes. I don't really know way this 1 needs to be there, but I guess it is in case we want to see where our original message ends we would have a clear indication that it is before the 1 tha precedes a bunch of zeroes.
and at the very end of the padded message we replace the last 64 bits with a 64 bit integer, or the 64 bit representation of it, that corresponds to the size of the message in bits.

So this is my code for it:

```js
function pad(buffer) {
  const blockSize = 64; // 64 bytes  (512 bits)
  // We need to put the length at the end of the message
  // and that is a 64 bit number or 8 bytes
  // but we also need to put a 1 before padding with 0
  // and since we can not add or remove individual bits
  // we add at least a whole byte that starts with 1
  // and then have 7 zeros.
  // so in the end we have to add at least 9 bytes
  // to the original message
  const paddedSize =
    (buffer.byteLength + 9) % blockSize === 0
      ? buffer.byteLength + 9
      : (Math.floor((buffer.byteLength + 9) / blockSize) + 1) * blockSize;

  // Here we create an Uint8Array with the padded size
  // and just set the original buffer in it.
  // All of the other spots in the buffer are already set to 0
  // for us as a nicety of javascript
  // I really wanted to user the transfer function
  // that would make this a one liner
  // But that is not implemented anywhere yet
  // https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer/transfer
  const destView = new Uint8Array(paddedSize);
  const sourceView = new Uint8Array(buffer);
  destView.set(sourceView);

  // Here I create a DataView on our resulting buffer
  // I like the DataView because it provides you a lot
  // of control over the underlying buffer, even regarding
  // the Endianness
  // https://developer.mozilla.org/en-US/docs/Glossary/Endianness
  // https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView
  const dataView = new DataView(destView.buffer);

  // Next we have to set the 1 at the end of our message
  // but since javascript does not allow us to manipulate
  // individual bits we have to write an entire byte that
  // has the first bit set to 1 and the rest set to 0
  // and that byte is the number 128 = 0b10000000
  // 0b here is just a way of writing binary numbers in js
  // it would have worked with the decimal form as well 128
  dataView.setUint8(buffer.byteLength, 0b10000000);

  // Then we calculate the size of our original message in bits
  // and set at the very end of the buffer occupying the last
  // 8 bytes (64 bits) of the padded message
  // Note the 'false' arg at the end
  // that indicates that we are using the Big Endian representation
  // of the number
  // https://developer.mozilla.org/en-US/docs/Glossary/Endianness
  const sizeBits = buffer.byteLength * 8;
  dataView.setBigUint64(paddedSize - 8, BigInt(sizeBits), false);

  // And finally we return our padded message
  return destView.buffer;
}
```

And on our original `sha1` function we just add a call to our new `pad` function:

```js
const TextEncoder = require("util").TextEncoder;

function sha1(message) {
  const buffer = new TextEncoder().encode(message).buffer;
  const paddedMessage = pad(buffer);

  // More code will come here
}
```

Ok, things are getting interesting!

#### Parsing the Message

Now on section 5.2.1 of the spec they tell us we have to parse our padded message into `n` blocks of 512 bits that later will be processed as 16 'words' of 32 bits each (32 \* 16 = 512).

That should be easy.

I decided to use the DataView again to represent the blocks. So the code is as follows:

```js
function parse(buffer) {
  const blockSize = 64; // 64 bytes / 512 bits
  const numOfBlocks = buffer.byteLength / blockSize;
  const blocks = [];
  for (let i = 0; i < numOfBlocks; i++) {
    const blockOffset = i * blockSize;
    const block = new DataView(
      buffer.slice(blockOffset, blockOffset + blockSize)
    );
    blocks.push(block);
  }
  return blocks;
}
```

This DataView thing is really awesome, and helped me a lot with some endianess problems. So much so that I intend to make a post dedicated only to the endianess thing. (// TODO: write JS endianess post and put link here)

Alright, now we add a call to `parse` in our original `sha1` function and we will be good to go to the meet of the algorithm:

```js
const TextEncoder = require("util").TextEncoder;

function sha1(message) {
  const buffer = new TextEncoder().encode(message).buffer;
  const paddedMessage = pad(buffer);
  const blocks = parse(paddedMessage);

  // More code will come here
}
```

### Initializing stuff

#### Initializing the hash

The SHA-1 hash is a sequence of 160 bits, or 20 bytes that are represented as 5 32 bit integers. These 32 bit integers are referenced in the spec as 'words'. Every 'word' is a 4 bytes number (32 bits). So our entire hash is represented as 5 words. Usually these words are represented in hexadecimal.

I am not going to go into too much detail here about number representations, but the hexadecimal representation uses 16 symbols to represent the numbers.
As a comparison the regular decimal representation uses ten symbols (0, 1, 2, 3, 4, 5, 6, 7, 8, 9) and hexadecimal uses (0, 1, 2, 3, 4, 5, 6, 7, 8, 9, a, b, c, d, e, f) while binary uses only two (0, 1). Just in the decimal system the zeroes to the left are irrelevant to the number value, but in computation we usually write them down because they occupy space in memory. That been said, here is a table show the equivalence of different number representations:

| Decimal | Padded Decimal | hex | Padded Hex | Binary | Padded Binary |
| ------: | -------------: | --: | ---------: | -----: | ------------: |
|       0 |           0000 |   0 |         00 |      0 |      00000000 |
|       1 |           0001 |   1 |         01 |      1 |      00000001 |
|       2 |           0002 |   2 |         02 |     10 |      00000010 |
|       3 |           0003 |   3 |         03 |     11 |      00000011 |
|       4 |           0004 |   4 |         04 |    100 |      00000100 |
|       5 |           0005 |   5 |         05 |    101 |      00000101 |
|       6 |           0006 |   6 |         06 |    110 |      00000110 |
|       7 |           0007 |   7 |         07 |    111 |      00000111 |
|       8 |           0008 |   8 |         08 |   1000 |      00001000 |
|       9 |           0009 |   9 |         09 |   1001 |      00001001 |
|      10 |           0010 |   a |         0a |   1010 |      00001010 |
|      11 |           0011 |   b |         0b |   1011 |      00001011 |
|      12 |           0012 |   c |         0c |   1100 |      00001100 |
|      13 |           0013 |   d |         0d |   1101 |      00001101 |
|      14 |           0014 |   e |         0e |   1110 |      00001110 |
|      15 |           0015 |   f |         0f |   1111 |      00001111 |
|      16 |           0016 |  10 |         10 |  10000 |      00010000 |
|      17 |           0017 |  11 |         11 |  10001 |      00010001 |
|      18 |           0018 |  12 |         12 |  10010 |      00010010 |
|      19 |           0019 |  13 |         13 |  10011 |      00010011 |
|      20 |           0020 |  14 |         14 |  10100 |      00010100 |
|      21 |           0021 |  15 |         15 |  10101 |      00010101 |
|      22 |           0022 |  16 |         16 |  10110 |      00010110 |
|      23 |           0023 |  17 |         17 |  10111 |      00010111 |
|      24 |           0024 |  18 |         18 |  11000 |      00011000 |
|      25 |           0025 |  19 |         19 |  11001 |      00011001 |
|      26 |           0026 |  1a |         1a |  11010 |      00011010 |
|      27 |           0027 |  1b |         1b |  11011 |      00011011 |
|      28 |           0028 |  1c |         1c |  11100 |      00011100 |
|      29 |           0029 |  1d |         1d |  11101 |      00011101 |
|      30 |           0030 |  1e |         1e |  11110 |      00011110 |
|      31 |           0031 |  1f |         1f |  11111 |      00011111 |
|     ... |            ... | ... |        ... |    ... |           ... |
|     255 |           0255 |  ff |         ff |  11111 |      11111111 |

As you can see we need two hexadecimal digits to represent a byte (8 bits). So our 20 byte hash can be represented with 40 hexadecimal digits. And every 4 byte word can be represented with 8 hex digits.

The section 5.3.1 of the spec says the initial values for the 5 words of the hash are as follows:

H0 = 6745230

H1 = efcdab89

H2 = 98badcfe

H3 = 10325476

H4 = c3d2e1f0

So I decided to represent this as a Uint32Array. This allows us to be sure that our values will be always represented as 32 bit unsigned integers.

```js
const hash = new Uint32Array([
  0x67452301,
  0xefcdab89,
  0x98badcfe,
  0x10325476,
  0xc3d2e1f0
]);
```

Again the 0x as the 0b allows us to use the hex and binary representations directly instead of having to use the decimal representation.

Now, our `sha1` function should look like this:

```js
const TextEncoder = require("util").TextEncoder;

function sha1(message) {
  const buffer = new TextEncoder().encode(message).buffer;
  const paddedMessage = pad(buffer);
  const blocks = parse(paddedMessage);

  const hash = new Uint32Array([
    0x67452301,
    0xefcdab89,
    0x98badcfe,
    0x10325476,
    0xc3d2e1f0
  ]);

  // More code will come here
}
```

#### Initializing the constants

#### Initializing the functions

### Hash computation

#### STEP 1: Prepare the message schedule W

#### STEP 2: Initialize the five working variables, a, b, c, d, and e, with the (i-1)st hash value:

#### STEP 3: For t=0 to 79 (maybe this could be done in the same loop as step 1)

#### STEP 4: Compute the ith intermediate hash value H(i)
