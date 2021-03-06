[![StyleCI](https://styleci.io/repos/78642763/shield?branch=master)](https://styleci.io/repos/78642763) [![Build Status](https://travis-ci.org/waxim/vouchers.svg?branch=master)](https://travis-ci.org/waxim/vouchers)

# Vouchers Lib
A PHP library for generating and validating vouchers. We make no assumptions about storage and instead offer the concept of `Bags` which can take any number of `Vouchers`. These bags can validate vouchers, generate new vouchers and apply validation rules across the whole set.

## Example
```php

$model = new Vouchers\Voucher\Model([
    'owner' => [
        'required'  => true,
        'immutable' => true,
    ],
    'claimed_by' => [
        'required' => true,
    ]
]);

$collection = new Vouchers\Bag($model);
$collection->fill(1000);

$voucher = $collection->pick();

print $voucher; // FHUW-JSUJ-KSIQ-JDUI
```

## Vouchers
Vouchers can take on almost any form, however you can use `Vouchers\Voucher\Model` to enforce validation and structure. The only required attribute is `code` which by default is immutable.

```php
$voucher = new Vouchers\Voucher();
print $voucher; // ABCD-EFGH-IJKL
```

You may also pass an array to the voucher to set pre existing values to the voucher. Matching fields (including `code`) will be validated.

```php
$voucher = new Voucher(['code' => 'ALAN-COLE-CODE', 'claimed_by' => '', 'claimed_on' => '']);
print $voucher; // "ALAN-COLE-CODE"
```

Any value passed on voucher creation can be get and set using `get()` and `set()` on the voucher.

```php
$voucher->set('owner', 'Alan');
echo $voucher->get('owner'); // Alan
```

### Model
By creating a model you can set default values and validation on vouchers created or loaded. Models are passed as an array to `Vouchers\Voucher\Model`

```php
$model = new Vouchers\Voucher\Model([
    'owner' => [
        'required'  => true,
        'immutable' => true,
    ],
    'claimed_by' => [
        'required' => true,
    ]
]);
```

If you set a voucher attribute as `immutable` then `Voucher` will throw the `ImmutableData` exception.

### Code
You can change the way the code is generated by settings generator on a model. A generator must implement `Vouchers\Voucher\Code\GeneratorInterface`

```php
namespace My\Voucher\Generator;

use Vouchers\Voucher\Code\Interface as Generator;

class MyCode implements Generator
{
    public function part()
    {
        return bin2hex(openssl_random_pseudo_bytes(2));
    }

    public function generate()
    {
        return strtoupper(sprintf("%s-%s-%s", $this->part(), $this->part(), $this->part()));
    }

    public function validate()
    {
        return true;
    }
}

```

Then tell the model to use this generator.

```php
$model = new Vouchers\Voucher\Model([
    'code' => [
        'generator' => \My\Voucher\Generator\MyCode::class
    ]
]);
```

## Bags
Bags act as collections for vouchers and allow you to enforce validations on a whole set. Bags can also act as a selector for vouchers, allowing to you pick a voucher at random and enforce rules on that selection. Bags are also `Iterable` so they can be used in loops.

```php
$collection = new Vouchers\Bag();
$collection->fill(1000);

foreach($collection as $voucher) {
    print $voucher;
}
```

You can use `Vouchers\Voucher\Model` to enfore a model on all items in a bag by passing a model as the first attribute on construction.

```php
$collection = new Vouchers\Bag($model);
```

You can fill a model with existing vouchers by using `add()` add will only accept an instance of `Vouchers\Voucher`

```php
$vouchers = [$voucher1...$voucher100];
foreach ($vouchers as $voucher) {
    $collection->add(new Vouchers\Voucher($voucher));
}
```

You can also run a map on any array, mapping the return as new vouchers within the bag. This is handy if you need to transform data to fit a model.

```php
$collection->map($vouchers, function ($voucher) {
    return new Vouchers\Voucher($voucher);
});
```

You can get a voucher by code, which can be used to see if a voucher exists.

```
$collection = new Vouchers\Bag();
$voucher = new Vouchers\Voucher(['code' => 'special-voucher']);
$collection->add($voucher);

$v = $collection->find("special-voucher");

if ($v) {
    print (string)$v;
} else {
    print "Voucher does not exist.";
}
```

### Pick
You can have the bag pick you a voucher at random by using `pick()` on any bag.

```php
$collection = new Vouchers\Bag();
$collection->fill(1000);

$collection->pick();
```

If you wish to validate the selection you can pass a callback to pick which will run until it returns a `true` or throw an `Vouchers\Exceptions\NoValidVouchers` exception.

```php
$collection->pick(function ($voucher) {
    return (bool)$voucher->owner == "Alan";
});
```

```php
try {
    $collection->pick(function ($voucher) {
        return 2 == 1;
    });
} catch (Exception $e) {
    print $e->getMessage();
}
```

You may also ask `pick()` to check all validators this bag might have (see Validate) and only return a voucher that is valid. Again this will throw `Vouchers\Exceptions\NoValidVouchers` is it doesn't find a voucher.

```php
$collection->pickValid();
```

### Validate
You can add validators to a bag, these validators can be used to validate requirements of a voucher using `validate()` on a bag and passing the voucher code as a parameter.

```php
$collection->validate("ALAN-COLE-CODE");
```

Validators can be added as callbacks to the validator function, or as a class that implements `Vouchers\Voucher\Validator` here is an example that assumes a voucher has an `expire_date` and checks it has not passed.

```php
$collection->validator(function ($voucher) {
    return $voucher->expire_date > new DateTime();
}, "Sorry, this voucher is expired");

try {
    $collection->validate("ALAN-COLE-CODE");
} catch (\Vouchers\Exceptions\VoucherNotValid $e) {
    return $e->getMessage(); // "Sorry, this voucher is expired";
}
```

### Kitchen Sink
This shows how to get vouchers from the subscriptions api, take a requested voucher, validate it and the claim it on the API.

```php
$api = new Discovery\Subscriptions\Api();
$api->setApiKey(getenv("SUBS_API_KEY"));
$api->setAppId(getenv("SUBS_APP_ID"));

$vouchers = $api->getAllVouchers();

$bag = new Vouchers\Bag();
$bag->map($vouchers, function($voucher) {
    return new Vouchers\Voucher($voucher);
});

# Add some validators
$bag->validator(function ($voucher) {
    return $voucher->owner == "Eurosport";
}, "Sorry, this voucher was not valid.");

$bag->validator(function ($voucher) {
    return !$voucher->used;
}, "Sorry, this voucher has been used.");

$bag->validator(function ($voucher) {
    return new DateTime($voucher->valid_till) < new DateTime();
}, "Sorry, this voucher is expired.");

try {
    $voucher = $collection->validate(filter_val(INPUT_POST, "voucher_code", FILTER_SANITIZE_STRING));
    $voucher->set("used", true // not really needed.
    $api->putVoucherClaim($voucher); // because this takes care of it.
} catch (\Vouchers\Exceptions\VoucherNotValid $e) {
    echo $e->getMessage();
}
```
