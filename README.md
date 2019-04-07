# Brick\VarExporter

<img src="https://raw.githubusercontent.com/brick/brick/master/logo.png" alt="" align="left" height="64">

A PHP library that offers a pretty and powerful alternative to `var_export()`.

[![Build Status](https://secure.travis-ci.org/brick/varexporter.svg?branch=master)](http://travis-ci.org/brick/varexporter)
[![Coverage Status](https://coveralls.io/repos/brick/varexporter/badge.svg?branch=master)](https://coveralls.io/r/brick/varexporter?branch=master)
[![Latest Stable Version](https://poser.pugx.org/brick/varexporter/v/stable)](https://packagist.org/packages/brick/varexporter)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](http://opensource.org/licenses/MIT)

## Introduction

PHP's [var_export()](https://www.php.net/manual/en/function.var-export.php) function is a handy way to export a variable as executable PHP code.
 
It is particularly useful to store data that can be cached by OPCache, and later retrieved very fast, much faster than unserializing data using `unserialize()` or `json_decode()`.

It also suffers from several drawbacks:

- It outputs arrays as `array()` notation, instead of the short `[]` notation
- It outputs numeric arrays with explicit and unnecessary `0 => ...` key => value syntax
- It outputs invalid PHP code for `stdClass` objects, using `stdClass::__set_state()` which doesn't exist
- It cannot export custom objects that do not implement `__set_state()`
- `__set_state()` does not play well with private properties in parent classes, which makes the implementation tedious
- `var_export()` does not complain when exporting an object with overridden private properties, and outputs and array with duplicate keys

This library aims to provide a prettier, safer, and powerful alternative to `var_export()`.

### Installation

This library is installable via [Composer](https://getcomposer.org/):

```bash
composer require brick/varexporter
```

### Requirements

This library requires PHP 7.1 or later.

### Project status & release process

While this library is still under development, it is well tested and should be stable enough to use in production environments.

The current releases are numbered `0.x.y`. When a non-breaking change is introduced (adding new methods, optimizing existing code, etc.), `y` is incremented.

**When a breaking change is introduced, a new `0.x` version cycle is always started.**

It is therefore safe to lock your project to a given release cycle, such as `0.1.*`.

If you need to upgrade to a newer release cycle, check the [release history](https://github.com/brick/varexporter/releases) for a list of changes introduced by each further `0.x.0` version.

## Quickstart

This library offers a single method, `VarExporter::export()` which works pretty much like `var_export()`:

```php
use Brick\VarExporter\VarExporter;

echo VarExporter::export([1, 2, ['foo' => 'bar', 'baz' => []]]);
```

This code will output:

```php
[
    1,
    2,
    [
        'foo' => 'bar',
        'baz' => []
    ]
]
```

Compare this to the `var_export()` output:

```php
array (
  0 => 1,
  1 => 2,
  2 => 
  array (
    'foo' => 'bar',
    'baz' => 
    array (
    ),
  ),
)
```

Note: unlike `var_export()`, `export()` always returns the exported variable, and never outputs it.

## Exporting stdClass objects

You come across a `stdClass` object every time you cast an array to an object, or use `json_decode()` with the second argument set to `false` (which is the default).

While the output of `var_export()` for `stdClass` is syntactically valid PHP code:

```php
var_export(json_decode('
    {
        "foo": "bar",
        "baz": {
            "hello": "world"
        }
    }
'));
```

```php
stdClass::__set_state(array(
   'foo' => 'bar',
   'baz' => 
  stdClass::__set_state(array(
     'hello' => 'world',
  )),
))
```

it is totally useless as it assumes that `stdClass` has a static `__set_state()` method, when it doesn't:

> Error: Call to undefined method stdClass::__set_state()

### What does `VarExporter` do instead?

It outputs an array to object cast, which is syntactically valid, readable **and** executable:

```php
echo VarExporter::export(json_decode('
    {
        "foo": "bar",
        "baz": {
            "hello": "world"
        }
    }
'));
```

```php
(object) [
    'foo' => 'bar',
    'baz' => (object) [
        'hello' => 'world'
    ]
]
```

## Exporting custom objects

As we've seen above, `var_export()` assumes that every object has a static [__set_state()](https://www.php.net/manual/en/language.oop5.magic.php#object.set-state) method that takes an associative array of property names to values, and returns a object.

This means that if you want to export an instance of a class outside of your control, you're screwed up. This also means that you have to write boilerplate code for your classes, that looks like:

```php
class Foo
{
    public $a;
    public $b;
    public $c;

    public static function __set_state(array $array) : self
    {
        $object = new self;

        $object->a = $array['a'];
        $object->b = $array['b'];
        $object->c = $array['c'];

        return $object;
    }
}
```

Or the more dynamic, reusable, and less IDE-friendly version:

```php
public static function __set_state(array $array) : self
{
    $object = new self;

    foreach ($array as $key => $value) {
        $object->{$key} = $value;
    }

    return $object;
}
```

If your class has a parent with private properties, you may have to do some gymnastics to write the value, and if your class overrides a private property of one of its parents, you're out of luck as `var_export()` puts all properties in the same bag, outputting an array with a duplicate key.

### What does `VarExporter` do instead?

It determines the most appropriate method to export your object, in this order:

- If your custom class has a `__set_state()` method, `VarExporter` uses it by default, just like `var_export()` would do:

    ```php
    \My\CustomClass::__set_state([
        'foo' => 'Hello',
        'bar' => 'World'
    ])
    ```

    The array passed to `__set_state()` will be built with the same semantics used by `var_export()`; this library aims to be 100% compatible in this regard. The only difference is when your class has overridden private properties: `var_export()` will output an array that contains the same key twice (resulting in data loss), while `VarExporter` will throw an `ExportException` to keep you on the safe side.

    Unlike `var_export()`, this method will only be used if actually implemented on the class.

    You can disable exporting objects this way, even if they implement `__set_state()`, using the [`NO_SET_STATE`](#varexporterno_set_state) option.

- If your class has `__serialize()` and `__unserialize()` methods ([introduced in PHP 7.4](https://wiki.php.net/rfc/custom_object_serialization), but this library accepts them in previous versions of PHP!), `VarExporter` uses the output of `__serialize()` to export the object, and gives it as input to `__unserialize()` to reconstruct the object:

    ```php
    (static function() {
        $class = new \ReflectionClass(\My\CustomClass::class);
        $object = $class->newInstanceWithoutConstructor();
    
        $object->__unserialize([
            'foo' => 'Test',
            'bar' => 1234
        ]);
    
        return $object;
    })()
    ```

    This method is recommended for exporting complex custom objects: it is forward compatible with the new serialization mechanism introduced in PHP 7.4, flexible, safe, and composes very well under inheritance.

    If for any reason you do not want to export objects that implement `__serialize()` and `__unserialize()` using this method, you can opt out by using the [`NO_SERIALIZE`](#varexporterno_serialize) option.

- If the class does not meet any of the conditions above, it is exported through direct property access, which in its simplest form looks like:

    ```php
    (static function() {
        $object = new \My\CustomClass;

        $object->publicProp = 'Foo';
        $object->dynamicProp = 'Bar';

        return $object;
    })()
    ```

    If the class has a constructor, it will be bypassed using reflection:

    ```php
    (static function() {
        $class = new \ReflectionClass(\My\CustomClass::class);
        $object = $class->newInstanceWithoutConstructor();

        ...
    })()
    ```

    If the class has non-public properties, they will be accessed through closures bound to the object:

    ```php
    (static function() {
        $class = new \ReflectionClass(\My\CustomClass::class);
        $object = $class->newInstanceWithoutConstructor();

        $object->publicProp = 'Foo';
        $object->dynamicProp = 'Bar';

        (function() {
            $this->protectedProp = 'contents';
            $this->privateProp = 'contents';
        })->bindTo($object, \My\CustomClass::class)();

        (function() {
            $this->privatePropInParent = 'contents';
        })->bindTo($object, \My\ParentClass::class)();

        return $object;
    })()
    ```

    You can disable exporting objects this way, using the [`NOT_ANY_OBJECT`](#varexporternot_any_object) option.

If you attempt to export a custom object and all compatible exporters have been disabled, an `ExportException` will be thrown.

## Options

`VarExporter::export()` accepts a bitmask of options as a second parameter:

```php
VarExporter::export($var, VarExporter::ADD_RETURN | VarExporter::ADD_TYPE_HINTS);
```

Available options:

### `VarExporter::ADD_RETURN`

Wraps the output in a return statement:

```php
return (...);
```

This makes the code ready to be executed in a PHP file―or `eval()`, for that matter.

### `VarExporter::ADD_TYPE_HINTS`

Adds type hints to objects created through reflection, and to `$this` inside closures bound to an object.
This allows the resulting code to be statically analyzed by external tools and IDEs:

```php
/** @var \My\CustomClass $object */
$object = $class->newInstanceWithoutConstructor();

(function() {
    /** @var \My\CustomClass $this */
    $this->privateProp = ...;
})->bindTo($object, \My\CustomClass::class)();
```

### `VarExporter::SKIP_DYNAMIC_PROPERTIES`

Skips dynamic properties on custom classes in the output. Dynamic properties are properties that are not part of the class definition, and added to an object at runtime. By default, any dynamic property set on a custom class is
exported; if this option is used, dynamic properties are only allowed on `stdClass` objects, and ignored on other objects.

### `VarExporter::NO_SET_STATE`

Disallows exporting objects through `__set_state()`.

### `VarExporter::NO_SERIALIZE`

Disallows exporting objects through `__serialize()` and `__unserialize()`.

### `VarExporter::NOT_ANY_OBJECT`

Disallows exporting any custom object using direct property access and bound closures.

## Error handling

Any error occurring on `export()` will throw an `ExportException`:

```php
use Brick\VarExporter\VarExporter;
use Brick\VarExporter\ExportException;

try {
    VarExporter::export(fopen('php://memory', 'r'));
} catch (ExportException $e) {
    // Type "resource" is not supported.
}
```

## Limitations

- Exporting internal classes, including closures, is currently not supported. `VarExporter` will throw an `ExportException` if it finds one.

  To avoid hitting this brick wall, you can implement `__serialize()` and `__unserialize()` in classes that contain references to internal objects.

- Just like `var_export()`, `VarExporter` cannot currently maintain object identity (two instances of the same object, once exported, will create two equal (`==`) yet distinct (`!==`) objects).

- And just like `var_export()`, it cannot currently handle circular references, such as object `A` pointing to `B`, and `B` pointing back to `A`.

In pretty much every other case, it offers an elegant and very efficient way to cache data to PHP files, and a solid alternative to serialization.
