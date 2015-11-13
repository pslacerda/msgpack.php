# msgpack.php

[![Build Status](https://travis-ci.org/rybakit/msgpack.php.svg?branch=master)](https://travis-ci.org/rybakit/msgpack.php)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/rybakit/msgpack.php/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/rybakit/msgpack.php/?branch=master)
[![Code Coverage](https://scrutinizer-ci.com/g/rybakit/msgpack.php/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/rybakit/msgpack.php/?branch=master)


## Installation

The recommended way to install the library is through [Composer](http://getcomposer.org):

```sh
$ composer require rybakit/msgpack
```


## Usage

### Packer

```php
use MessagePack\Packer;

$packer = new Packer();

...

$packed = $packer->pack($value);
```


### Unpacker

```php
use MessagePack\Unpacker;

$unpacker = new Unpacker();

...

$unpacked = $unpacker->unpack($data);
```

### BufferUnpacker

```php
use MessagePack\BufferUnpacker;

$unpacker = new BufferUnpacker();

...

$unpacker->append($data);
$unpacked = $unpacker->unpack($data);

...

$unpacker->reset($data);
$unpackedBlocks = $unpacker->tryUnpack();
```

### Extensions

```php
use MessagePack\Ext;
use MessagePack\Packer;
use MessagePack\Unpacker;

$packerd = (new Packer())->pack(new Ext(42, "\xaa"));
$ext = (new Unpacker())->unpack($packed);

$extType = $ext->getType(); // 42
$extData = $ext->getData(); // "\xaa"
```


## Tests

The easiest way to run tests is with Docker. First, build an image using the [dockerfile.sh](dockerfile.sh) generator:

```sh
$ ./dockerfile.sh | docker build -t msgpack -
```

Then run unit tests:

```sh
$ docker run --rm --name msgpack -v $(pwd):/msgpack -w /msgpack msgpack
```


## License

The library is released under the MIT License. See the bundled [LICENSE](LICENSE) file for details.