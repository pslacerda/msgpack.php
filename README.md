# msgpack.php

A pure PHP implementation of the MessagePack serialization format.

[![Build Status](https://travis-ci.org/rybakit/msgpack.php.svg?branch=master)](https://travis-ci.org/rybakit/msgpack.php)
[![Code Coverage](https://scrutinizer-ci.com/g/rybakit/msgpack.php/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/rybakit/msgpack.php/?branch=master)


## Features

 * Fully compliant with the latest [MessagePack specification](https://github.com/msgpack/msgpack/blob/master/spec.md),
   including **bin**, **str** and **ext** types
 * Supports [streaming unpacking](#unpacking)
 * Supports [unsigned 64-bit integers handling](#unsigned-64-bit-integers)
 * Supports [object serialization](#custom-types)
 * Works with PHP 5.4-7.x and HHVM 3.9+
 * [Fully tested](https://travis-ci.org/rybakit/msgpack.php)
 * [Relatively fast](#performance)


## Table of contents

 * [Installation](#installation)
 * [Usage](#usage)
   * [Packing](#packing)
     * [Type Detection Mode](#type-detection-mode)
   * [Unpacking](#unpacking)
     * [Unsigned 64-bit Integers](#unsigned-64-bit-integers)
 * [Extensions](#extensions)
 * [Custom Types](#custom-types)
 * [Exceptions](#exceptions)
 * [Tests](#tests)
    * [Performance](#performance)
 * [License](#license)


## Installation

The recommended way to install the library is through [Composer](http://getcomposer.org):

```sh
$ composer require rybakit/msgpack
```


## Usage

### Packing

To pack values you can either use an instance of `Packer`:

```php
use MessagePack\Packer;

$packer = new Packer();

...

$packed = $packer->pack($value);
```

or call the static method on the `MessagePack` class:

```php
use MessagePack\MessagePack;

...

$packed = MessagePack::pack($value);
```

In the examples above, the method `pack` automatically packs a value depending on its type.
But not all PHP types can be uniquely translated to MessagePack types. For example,
MessagePack format defines `map` and `array` types, which are represented by a single `array`
type in PHP. By default, the packer will pack a PHP array as a MessagePack array if it
has sequential numeric keys, starting from `0` and as a MessagePack map otherwise:

```php
$mpArr1 = $packer->pack([1, 2]);                   // MP array [1, 2]
$mpArr2 = $packer->pack([0 => 1, 1 => 2]);         // MP array [1, 2]
$mpMap1 = $packer->pack([0 => 1, 2 => 3]);         // MP map {0: 1, 2: 3}
$mpMap2 = $packer->pack([1 => 2, 2 => 3]);         // MP map {1: 2, 2: 3}
$mpMap3 = $packer->pack(['foo' => 1, 'bar' => 2]); // MP map {foo: 1, bar: 2}
```

However, sometimes you need to pack a sequential array as a MessagePack map.
To do this, use the `packMap` method:

```php
$mpMap = $packer->packMap([1, 2]); // {0: 1, 1: 2}
```

Here is a list of all low-level packer methods:

```php
$packer->packNil();                   // MP nil
$packer->packBool(true);              // MP bool
$packer->packArray([1, 2]);           // MP array
$packer->packMap([1, 2]);             // MP map
$packer->packExt(new Ext(1, "\xaa")); // MP ext
$packer->packFloat(4.2);              // MP float
$packer->packInt(42);                 // MP int
$packer->packStr('foo');              // MP str
$packer->packBin("\x80");             // MP bin
```

> Check ["Custom Types"](#custom-types) section below on how to pack arbitrary PHP objects.


#### Type detection mode

Automatically detecting an MP type of PHP arrays/strings adds some overhead which can be noticed
when you pack large (16- and 32-bit) arrays or strings. However, if you know the variable type
in advance (for example, you only work with UTF-8 strings or/and associative arrays), you can
eliminate this overhead by forcing the packer to use the appropriate type, which will save it
from running the auto detection routine:

```php
$packer = new Packer(Packer::FORCE_STR);
// or
// $packer->setTypeDetectionMode(Packer::FORCE_STR);
...
$packed = $packer->pack([$utf8string1, $utf8string2]);
```

Available modes are:

```php
Packer::FORCE_STR
Packer::FORCE_BIN
Packer::FORCE_ARR
Packer::FORCE_MAP
```

Of course, you can combine modes:

```php
// convert PHP strings to MP strings, PHP arrays to MP maps
$packer->setTypeDetectionMode(Packer::FORCE_STR | Packer::FORCE_MAP);

// convert PHP strings to MP binaries, PHP arrays to MP arrays
$packer->setTypeDetectionMode(Packer::FORCE_BIN | Packer::FORCE_ARR);

// this will throw \InvalidArgumentException
$packer->setTypeDetectionMode(Packer::FORCE_STR | Packer::FORCE_BIN);
$packer->setTypeDetectionMode(Packer::FORCE_MAP | Packer::FORCE_ARR);
```


### Unpacking

To unpack data you can either use an instance of `BufferUnpacker`:

```php
use MessagePack\BufferUnpacker;

$unpacker = new BufferUnpacker();

...

$unpacker->reset($packed);
$value = $unpacker->unpack();
```

or call the static method on the `MessagePack` class:

```php
use MessagePack\MessagePack;

...

$value = MessagePack::unpack($packed);
```

If the packed data is received in chunks (e.g. when reading from a stream), use the `tryUnpack`
method, which will try to unpack data and return an array of unpacked data instead of throwing an `InsufficientDataException`:

```php
$unpacker->append($chunk1);
$unpackedBlocks = $unpacker->tryUnpack();

$unpacker->append($chunk2);
$unpackedBlocks = $unpacker->tryUnpack();
```


#### Unsigned 64-bit Integers

The binary MessagePack format has unsigned 64-bit as its largest integer data type,
but PHP does not support such integers. By default, while unpacking `uint64` value
the library will throw an `IntegerOverflowException`.

You can change this default behavior to unpack `uint64` integer to a string:

```php
$unpacker->setIntOverflowMode(BufferUnpacker::INT_AS_STR);
```

Or to a `Gmp` number (make sure that [gmp](http://php.net/manual/en/book.gmp.php) extension is enabled):

```php
$unpacker->setIntOverflowMode(BufferUnpacker::INT_AS_GMP);
```


### Extensions

To define application-specific types use the `Ext` class:

```php
use MessagePack\Ext;
use MessagePack\MessagePack;

$packed = MessagePack::pack(new Ext(42, "\xaa"));
$ext = MessagePack::unpack($packed);

$extType = $ext->getType(); // 42
$extData = $ext->getData(); // "\xaa"
```


### Custom Types

In addition to [the basic types](https://github.com/msgpack/msgpack/blob/master/spec.md#type-system),
the library provides the functionality to serialize and deserialize arbitrary types.
To do this, you need to create a transformer, that converts your type to a type, which can be handled by MessagePack.

For example, the code below shows how to add `DateTime` object support:

```php
use MessagePack\TypeTransformer\TypeTransformer;

class DateTimeTransformer implements TypeTransformer
{
    private $id;

    public function __construct($id)
    {
        $this->id = $id;
    }

    public function getId()
    {
        return $this->id;
    }

    public function supports($value)
    {
        return $value instanceof \DateTime;
    }

    public function transform($value)
    {
        return $value->format(\DateTime::RFC3339);
    }

    public function reverseTransform($data)
    {
        return new \DateTime($data);
    }
}
```

```php
use MessagePack\BufferUnpacker;
use MessagePack\Packer;
use MessagePack\TypeTransformer\Collection;

$packer = new Packer();
$unpacker = new BufferUnpacker();

$coll = new Collection([new DateTimeTransformer(5)]);
// $coll->add(new AnotherTypeTransformer(42));

$packer->setTransformers($coll);
$unpacker->setTransformers($coll);

$packed = $packer->pack(['foo' => new DateTime(), 'bar' => 'baz']);
$value = $unpacker->reset($packed)->unpack();
```


## Exceptions

If an error occurs during packing/unpacking, a `PackingFailedException` or `UnpackingFailedException`
will be thrown, respectively.

In addition, there are two more exceptions that can be thrown during unpacking:

 * `InsufficientDataException`
 * `IntegerOverflowException`


## Tests

Run tests as follows:

```sh
$ phpunit
```

Also, if you already have Docker installed, you can run the tests in a docker container.
First, create a container:

```sh
$ ./dockerfile.sh | docker build -t msgpack -
```

The command above will create a container named `msgpack` with PHP 7.0 runtime.
You may change the default runtime by defining the `PHP_RUNTIME` environment variable:

```sh
$ PHP_RUNTIME='php:7.1-cli' ./dockerfile.sh | docker build -t msgpack -
```

> See a list of various runtimes [here](.travis.yml#L9-L16).

Then run the unit tests:

```sh
$ docker run --rm --name msgpack -v $(pwd):/msgpack -w /msgpack msgpack
```


#### Performance

To check the performance run:

```sh
$ php tests/bench.php
```

This command will output something like:

```
Filter: MessagePack\Tests\Perf\Filter\ListFilter
Rounds: 3
Iterations: 100000

=============================================
Test/Target            Packer  BufferUnpacker
---------------------------------------------
nil .................. 0.0091 ........ 0.0223
false ................ 0.0100 ........ 0.0207
true ................. 0.0094 ........ 0.0237
7-bit uint #1 ........ 0.0085 ........ 0.0167
7-bit uint #2 ........ 0.0094 ........ 0.0153
7-bit uint #3 ........ 0.0093 ........ 0.0161
5-bit sint #1 ........ 0.0103 ........ 0.0202
5-bit sint #2 ........ 0.0102 ........ 0.0201
5-bit sint #3 ........ 0.0102 ........ 0.0201
8-bit uint #1 ........ 0.0122 ........ 0.0341
8-bit uint #2 ........ 0.0122 ........ 0.0379
8-bit uint #3 ........ 0.0131 ........ 0.0344
16-bit uint #1 ....... 0.0166 ........ 0.0460
16-bit uint #2 ....... 0.0172 ........ 0.0466
16-bit uint #3 ....... 0.0174 ........ 0.0486
32-bit uint #1 ....... 0.0212 ........ 0.0595
32-bit uint #2 ....... 0.0211 ........ 0.0591
32-bit uint #3 ....... 0.0215 ........ 0.0598
64-bit uint #1 ....... 0.0326 ........ 0.0729
64-bit uint #2 ....... 0.0332 ........ 0.0728
8-bit int #1 ......... 0.0122 ........ 0.0404
8-bit int #2 ......... 0.0128 ........ 0.0407
8-bit int #3 ......... 0.0123 ........ 0.0431
16-bit int #1 ........ 0.0184 ........ 0.0527
16-bit int #2 ........ 0.0185 ........ 0.0514
16-bit int #3 ........ 0.0160 ........ 0.0523
32-bit int #1 ........ 0.0215 ........ 0.0679
32-bit int #2 ........ 0.0207 ........ 0.0681
32-bit int #3 ........ 0.0231 ........ 0.0707
64-bit int #1 ........ 0.0329 ........ 0.0756
64-bit int #2 ........ 0.0336 ........ 0.0780
64-bit int #3 ........ 0.0342 ........ 0.0759
64-bit float #1 ...... 0.0282 ........ 0.0644
64-bit float #2 ...... 0.0284 ........ 0.0618
64-bit float #3 ...... 0.0280 ........ 0.0623
fix string #1 ........ 0.0249 ........ 0.0225
fix string #2 ........ 0.0300 ........ 0.0324
fix string #3 ........ 0.0268 ........ 0.0349
fix string #4 ........ 0.0291 ........ 0.0328
8-bit string #1 ...... 0.0322 ........ 0.0563
8-bit string #2 ...... 0.0370 ........ 0.0568
8-bit string #3 ...... 0.0438 ........ 0.0569
16-bit string #1 ..... 0.0488 ........ 0.0685
16-bit string #2 ..... 3.2064 ........ 0.3242
32-bit string ........ 3.2056 ........ 0.3364
wide char string #1 .. 0.0288 ........ 0.0353
wide char string #2 .. 0.0341 ........ 0.0551
8-bit binary #1 ...... 0.0285 ........ 0.0474
8-bit binary #2 ...... 0.0293 ........ 0.0482
8-bit binary #3 ...... 0.0314 ........ 0.0509
16-bit binary ........ 0.0374 ........ 0.0638
32-bit binary ........ 0.3761 ........ 0.3347
fixext 1 ............. 0.0276 ........ 0.0739
fixext 2 ............. 0.0275 ........ 0.0794
fixext 4 ............. 0.0277 ........ 0.0788
fixext 8 ............. 0.0281 ........ 0.0789
fixext 16 ............ 0.0320 ........ 0.0802
8-bit ext ............ 0.0367 ........ 0.0880
16-bit ext ........... 0.0406 ........ 0.1001
32-bit ext ........... 0.3830 ........ 0.3734
fix array #1 ......... 0.0245 ........ 0.0233
fix array #2 ......... 0.0857 ........ 0.0900
16-bit array #1 ...... 0.2508 ........ 0.2913
16-bit array #2 ........... S ............. S
32-bit array .............. S ............. S
complex array ........ 0.3618 ........ 0.4304
fix map #1 ........... 0.1633 ........ 0.1900
fix map #2 ........... 0.0723 ........ 0.0684
fix map #3 ........... 0.0789 ........ 0.1155
fix map #4 ........... 0.0789 ........ 0.0901
16-bit map #1 ........ 0.4313 ........ 0.5105
16-bit map #2 ............. S ............. S
32-bit map ................ S ............. S
complex map .......... 0.5088 ........ 0.5348
=============================================
Total                 10.5549          6.6065
Skipped                     4               4
Failed                      0               0
Ignored                     0               0
```

You may change default benchmark settings by defining the following environment variables:

 * `MP_BENCH_TARGETS` (pure_p, pure_ps, pure_pa, pure_psa, pure_bu, pecl_p, pecl_u)
 * `MP_BENCH_ITERATIONS`/`MP_BENCH_DURATION`
 * `MP_BENCH_ROUNDS`
 * `MP_BENCH_TESTS`

For example:

```sh
$ export MP_BENCH_TARGETS=pure_p
$ export MP_BENCH_ITERATIONS=1000000
$ export MP_BENCH_ROUNDS=5
$ # a comma separated list of test names
$ export MP_BENCH_TESTS='complex array, complex map'
$ # or an regexp
$ # export MP_BENCH_TESTS='/complex (array|map)/'
$ php tests/bench.php
```

Another example, benchmarking both the library and [msgpack pecl extension](https://pecl.php.net/package/msgpack):

```
$ MP_BENCH_TARGETS=pure_ps,pure_bu,pecl_p,pecl_u php tests/bench.php

Filter: MessagePack\Tests\Perf\Filter\ListFilter
Rounds: 3
Iterations: 100000

================================================================================
Test/Target           Packer (str)  BufferUnpacker  msgpack_pack  msgpack_unpack
--------------------------------------------------------------------------------
nil ....................... 0.0083 ........ 0.0215 ...... 0.0089 ........ 0.0062
false ..................... 0.0102 ........ 0.0207 ...... 0.0077 ........ 0.0062
true ...................... 0.0086 ........ 0.0214 ...... 0.0076 ........ 0.0064
7-bit uint #1 ............. 0.0119 ........ 0.0164 ...... 0.0090 ........ 0.0065
7-bit uint #2 ............. 0.0094 ........ 0.0163 ...... 0.0083 ........ 0.0053
7-bit uint #3 ............. 0.0096 ........ 0.0159 ...... 0.0082 ........ 0.0061
5-bit sint #1 ............. 0.0103 ........ 0.0196 ...... 0.0081 ........ 0.0061
5-bit sint #2 ............. 0.0103 ........ 0.0199 ...... 0.0082 ........ 0.0069
5-bit sint #3 ............. 0.0104 ........ 0.0198 ...... 0.0080 ........ 0.0075
8-bit uint #1 ............. 0.0124 ........ 0.0342 ...... 0.0078 ........ 0.0067
8-bit uint #2 ............. 0.0125 ........ 0.0357 ...... 0.0082 ........ 0.0078
8-bit uint #3 ............. 0.0127 ........ 0.0354 ...... 0.0081 ........ 0.0077
16-bit uint #1 ............ 0.0196 ........ 0.0469 ...... 0.0096 ........ 0.0070
16-bit uint #2 ............ 0.0171 ........ 0.0458 ...... 0.0083 ........ 0.0067
16-bit uint #3 ............ 0.0186 ........ 0.0469 ...... 0.0082 ........ 0.0065
32-bit uint #1 ............ 0.0215 ........ 0.0600 ...... 0.0083 ........ 0.0068
32-bit uint #2 ............ 0.0211 ........ 0.0590 ...... 0.0082 ........ 0.0065
32-bit uint #3 ............ 0.0213 ........ 0.0586 ...... 0.0088 ........ 0.0073
64-bit uint #1 ............ 0.0327 ........ 0.0747 ...... 0.0095 ........ 0.0061
64-bit uint #2 ............ 0.0312 ........ 0.0714 ...... 0.0082 ........ 0.0065
8-bit int #1 .............. 0.0123 ........ 0.0413 ...... 0.0089 ........ 0.0065
8-bit int #2 .............. 0.0124 ........ 0.0401 ...... 0.0091 ........ 0.0066
8-bit int #3 .............. 0.0123 ........ 0.0402 ...... 0.0081 ........ 0.0066
16-bit int #1 ............. 0.0182 ........ 0.0502 ...... 0.0080 ........ 0.0068
16-bit int #2 ............. 0.0169 ........ 0.0523 ...... 0.0082 ........ 0.0067
16-bit int #3 ............. 0.0173 ........ 0.0504 ...... 0.0081 ........ 0.0066
32-bit int #1 ............. 0.0211 ........ 0.0691 ...... 0.0092 ........ 0.0063
32-bit int #2 ............. 0.0210 ........ 0.0690 ...... 0.0087 ........ 0.0068
32-bit int #3 ............. 0.0210 ........ 0.0696 ...... 0.0082 ........ 0.0067
64-bit int #1 ............. 0.0317 ........ 0.0736 ...... 0.0083 ........ 0.0064
64-bit int #2 ............. 0.0318 ........ 0.0762 ...... 0.0082 ........ 0.0078
64-bit int #3 ............. 0.0321 ........ 0.0765 ...... 0.0091 ........ 0.0078
64-bit float #1 ........... 0.0276 ........ 0.0620 ...... 0.0077 ........ 0.0065
64-bit float #2 ........... 0.0292 ........ 0.0659 ...... 0.0083 ........ 0.0065
64-bit float #3 ........... 0.0294 ........ 0.0672 ...... 0.0069 ........ 0.0071
fix string #1 ............. 0.0157 ........ 0.0210 ...... 0.0085 ........ 0.0063
fix string #2 ............. 0.0178 ........ 0.0348 ...... 0.0100 ........ 0.0080
fix string #3 ............. 0.0183 ........ 0.0351 ...... 0.0085 ........ 0.0090
fix string #4 ............. 0.0175 ........ 0.0335 ...... 0.0084 ........ 0.0081
8-bit string #1 ........... 0.0200 ........ 0.0579 ...... 0.0083 ........ 0.0092
8-bit string #2 ........... 0.0205 ........ 0.0605 ...... 0.0089 ........ 0.0079
8-bit string #3 ........... 0.0199 ........ 0.0600 ...... 0.0132 ........ 0.0084
16-bit string #1 .......... 0.0256 ........ 0.0709 ...... 0.0132 ........ 0.0090
16-bit string #2 .......... 0.3552 ........ 0.3236 ...... 0.3384 ........ 0.2617
32-bit string ............. 0.3617 ........ 0.3382 ...... 0.3358 ........ 0.2717
wide char string #1 ....... 0.0183 ........ 0.0334 ...... 0.0084 ........ 0.0080
wide char string #2 ....... 0.0201 ........ 0.0582 ...... 0.0087 ........ 0.0095
8-bit binary #1 ................ I ............. I ........... F ............. I
8-bit binary #2 ................ I ............. I ........... F ............. I
8-bit binary #3 ................ I ............. I ........... F ............. I
16-bit binary .................. I ............. I ........... F ............. I
32-bit binary .................. I ............. I ........... F ............. I
fixext 1 ....................... I ............. I ........... F ............. F
fixext 2 ....................... I ............. I ........... F ............. F
fixext 4 ....................... I ............. I ........... F ............. F
fixext 8 ....................... I ............. I ........... F ............. F
fixext 16 ...................... I ............. I ........... F ............. F
8-bit ext ...................... I ............. I ........... F ............. F
16-bit ext ..................... I ............. I ........... F ............. F
32-bit ext ..................... I ............. I ........... F ............. F
fix array #1 .............. 0.0251 ........ 0.0243 ...... 0.0158 ........ 0.0080
fix array #2 .............. 0.0774 ........ 0.0893 ...... 0.0209 ........ 0.0209
16-bit array #1 ........... 0.2440 ........ 0.2675 ...... 0.0402 ........ 0.0498
16-bit array #2 ................ S ............. S ........... S ............. S
32-bit array ................... S ............. S ........... S ............. S
complex array .................. I ............. I ........... F ............. F
fix map #1 ..................... I ............. I ........... F ............. I
fix map #2 ................ 0.0598 ........ 0.0675 ...... 0.0182 ........ 0.0191
fix map #3 ..................... I ............. I ........... F ............. I
fix map #4 ..................... I ............. I ........... F ............. I
16-bit map #1 ............. 0.4302 ........ 0.4706 ...... 0.0399 ........ 0.0679
16-bit map #2 .................. S ............. S ........... S ............. S
32-bit map ..................... S ............. S ........... S ............. S
complex map ............... 0.4706 ........ 0.5210 ...... 0.0702 ........ 0.0755
================================================================================
Total                       2.8615          4.2113        1.2680          1.0922
Skipped                          4               4             4               4
Failed                           0               0            17               9
Ignored                         17              17             0               8
```

> Note, that this is not a fair comparison as the msgpack extension (0.5.2+, 2.0) doesn't
support **ext**, **bin** and UTF-8 **str** types.


## License

The library is released under the MIT License. See the bundled [LICENSE](LICENSE) file for details.
