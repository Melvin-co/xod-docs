---
title: C++ Node API Reference
version: 0.36.0
---

# C++ Node API Reference

## Basic types

#### `Number`

Represents a XOD number. In current implementations typedef’ed to IEEE 754 32-bit floating point data type.

#### `TimeMs`

Represents a big integer to store time values expressed in milliseconds. Typedef’ed to 32-bit unsigned integer.

#### `XString`

Represents XOD string. Typedef’ed to [`List<char>`](#List), so the same methods and functions may be used to manipulate it.

## Inside `node {}`

The `node` block describes the state and behaviour of a node. Under the hood, it is converted to a regular C++ `struct`, and should be treated as such. You can add fields and methods to it.

#### `meta {}`

The `meta` block used to group some part of code that will be placed inside a `struct Node` before the generated code.

Usually used to define a custom type.

#### `Context`

An opaque type to distinguish a particular instance of node among other nodes of the same type.

Required as an argument for most node functions. Can be thought as an explicit `this` in many OOP languages, or like the `self` in Python.

#### `State` (deprecated since 0.35)

A user-defined struct to store node’s data for a lifetime of the program. Each node type has its own `State` definition.

#### `typeof_$$$`

A type for each of node's pins. Useful to access the type of a pin if its symbol is hardly accessible in other ways.

`$$$` gets replaced by a pin label as is. For example, `typeof_IN`, `typeof_OUT`. Since pin names are unique, there is no destinction between inputs and outputs.

### Pin descriptors

Automatically generated descriptor for each of node’s pins. Never instantiated. Used as a template argument in functions to access data.

`$$$` gets replaced by a pin label as is. For example, `input_Smin`, `output_FOO`.

<a name="input_xxx"></a>

#### `input_$$$`

If a node has a single unlabeled input, `input_IN` is generated for it.

If a node has multiple unlabeled inputs, they are referred as `input_IN1`, `input_IN2`,…, `input_IN7`.

<a name="output_xxx"></a>

#### `output_$$$`

XOD runtime supports no more than seven outputs on C\++ nodes.

If a node has a single unlabeled output, `output_OUT` is generated for it.

If a node has multiple unlabeled outputs, they are referred as `output_OUT1`, `output_OUT2`,…, `output_OUT7`.

### Constant pin values

Known at a compile time. Might be used in static asserts to ensure correctness.

#### `constant_input_$$$`

A value of a constant input pin.

#### `constant_output_$$$`

The descriptor for constant output of the node.

Should be defined manually as

```cpp
static constexpr typeof_OUT constant_output_OUT = $$$;
```

`$$$` should be replaced with a valid value for the output.

In case of creating an "unpack" node for the custom type, which is a struct that contains a constant value (such as PORT), you might define it as:
```cpp
static constexpr typeof_OUT constant_output_OUT = typeof_DEV::port;
```

## `nodespace {}`

The `nodespace` block is used to place some code outside the `struct Node {}`, but inside a node namespace, so this code will be accessible only with that node.

It's a good place to [store some constants in PROGMEM](/libs/xod/graphics/heart-16x16-rgba/), make an [alias for a shared type](/libs/xod/json/number-value/) and so on.

## Node functions

<a name="evaluate"></a>

#### `void evaluate(Context ctx)`

Entry-point function for any C\++ node. Called by the runtime when a node requires re-evaluation for whatever reason: input data update, schedule timeout, etc. Each C\++ node must implement `evaluate`.

<a name="getValue"></a>

#### `ValueT getValue<input_$$$|output_$$$>(Context ctx)`

Returns the most recent value on an input or output pin. The result type `ValueT` depends on pin data type and could be `bool`, `Number`, etc.

Note, the pulse type has no values. It’s meaningless to call `getValue` on a pulse-type pin. To know if a pulse was fired use [`isInputDirty`](#isInputDirty) instead.

A node owns its output values, so in simple cases, they could serve both as values for downstream nodes _and_ as node’s state storage. In such cases use `getValue<output_$$$>(ctx)` to access the last emitted output value.

<a name="emitValue"></a>

#### `void emitValue<output_$$$>(Context ctx, ValueT value)`

Sets new value on an output pin. The argument type `ValueT` depends on pin data type and can be `bool`, `Number`, etc.

A call to `emitValue` causes immediate downstream nodes to be re-evaluated within the current transaction even if the output value has not changed. If it’s cheap to check, avoid sending duplicate values.

To emit a pulse, use `true` as the `value`.

<a name="isInputDirty"></a>

#### `bool isInputDirty<input_$$$>(Context ctx)`

Returns `true` if and only if the input pin specified has got a newly emitted value during the current transaction.

<a name="getState"></a>

#### `State* getState(Context ctx)` (deprecated since 0.35)

Returns a pointer to the persistent state storage for the current node. The [`State`](#State) structure is defined by the node author. Fields could be updated directly, with the pointer returned, no commit is required after a change.

<a name="isSettingUp"></a>

#### `bool isSettingUp()`

Returns `true` if the evaluation takes place inside the very first transaction. Use the function to run some initialization code once on boot, or prevent undesirable initial values emission.

<a name="transactionTime"></a>

#### `TimeMs transactionTime()`

Returns time of the current transaction since the program started, in milliseconds. Stays constant for the whole duration of any particular transaction. Prefer this function to `millis` to avoid time difference error accumulation.

<a name="setTimeout"></a>

#### `void setTimeout(Context ctx, TimeMs timeout)`

Schedules a forced re-evaluation of a node past `timeout` milliseconds after the current transaction time.

A node may have at most one scheduled update. If called multiple times, the subsequent calls cancel the previous schedule for the node.

If you need to schedule evaluation right after the current transaction completes, use [`setImmediate`](#setImmediate) instead.

<a name="clearTimeout"></a>

#### `void clearTimeout(Context ctx)`

Cancels scheduled evaluation of a node (if present). Safe to call even if the update was not scheduled.

<a name="isTimedOut"></a>

#### `bool isTimedOut(Context ctx)`

Returns `true` if node’s schedule timed out right at the current transaction. Unless scheduled again will return `false` since the next transaction.

<a name="setImmediate"></a>

#### `void setImmediate()`

Schedules a forced re-evaluation of the node at the next transaction.

<a name="raiseError"></a>

#### `void raiseError<output_$$$>(Context ctx)`

Raises an error on an output pin. For value types, the error will persist until a valid value will be emitted with [`emitValue`](#emitValue). For pulse types, the error will be cleared right before the node's next evaluation.

<a name="raiseError_for_all"></a>

#### `void raiseError(Context ctx)`

Raises errors on all output pins.

<a name="getError"></a>

#### `bool getError<input_$$$>(Context ctx)`

Returns `true` if a pin has an upstream error, and `false` if it has none.

## List objects

Lists in XOD are not classical linked lists, nor vectors. They are smart (or dumb) _views_ over existing data. The only useful thing a list can do is to create an `Iterator<T>` allowing to enumerate list values from the head to the tail.

**Usage example**

```cpp
Number sum(List<Number> numbers) {
    Number result = 0;
    for (Iterator<Number> it = numbers.iterate(); it; ++it)
        result += *it;
    return result;
}
```

<a name="List"></a>

#### `List<T>`

A list of elements of type `T`.

A `List<T>` is a thin [Pimpl](https://en.wikipedia.org/wiki/Opaque_pointer)-wrapper around a `ListView<T>`. Since internally a `List<T>` contains just a single pointer passing a list by value is a cheap operation.

#### `List<T>::List()`

Constructs a new nil (null element) list.

#### `List<T>::List(const ListView<T>* view)`

Wraps a `view` into a list. The `view` must be kept alive for the whole list lifetime. Otherwise, the program will crash. This requirement is _not_ checked automatically.

<a name="List--iterate"></a>

#### `Iterator<T> List<T>::iterate() const`

Creates a new iterator pointing to the first element of the list.

<a name="Iterator"></a>

#### `Iterator<T>`

An iterator of `List<T>`. Iterators are expected to be short-living objects to enumerate elements of a list. They should not be cloned or passed around. An iterator points either to a valid element or it points out of list bounds. The later is used to denote iteration completed.

<a name="Iterator--bool"></a>

#### `Iterator<T>::operator bool() const`

Implicit cast to `bool` type. Returns `true` if the iterator points to a valid element, i.e. if the iterator is valid.

<a name="Iterator--deref"></a>

#### `T Iterator<T>::operator*() const`

Returns the element pointed by the iterator. Mimics pointer dereference. Returns a default value for type `T` if the iterator is not valid.

<a name="Iterator--value"></a>

#### `bool Iterator<T>::value(T* out) const`

Checks validity and retrieves element at once. Returns `true` if the iterator is valid, writes the element into `out` in that case. If the iterator is invalid returns `false` and keeps `out` intact.

<a name="Iterator--increment"></a>

#### `Iterator& Iterator<T>::operator++()`

Advances the iterator to the next element. May make it invalid. Conventionally returns self.

## List functions

<a name="length"></a>

#### `size_t length(List<T> xs)`

Returns a length of the list. To compute it iterates over all elements, so it’s not a trivial operation, so prefer caching the result over calling `length` multiple times.

<a name="dump"></a>

#### `size_t dump(List<T> xs, T* outBuff)`

Copies elements from list `xs` into the flat `outBuff`. A caller must guarantee that the buffer pointed has enough size to contain all elements of the list. Otherwise, the program will crash.

<a name="foldl"></a>

#### `TR foldl(List<T> xs, TR (*func)(TR, T), TR acc)`

Performs [left fold](<https://en.wikipedia.org/wiki/Fold_(higher-order_function)>) (also known as “reduce”) of the list `xs` using `func` reducer function and `acc` as starting value for the accumulator.

## Type converters

#### `size_t formatNumber(Number value, int prec, char* str)`

Converts a `Number` to a string. Like `dtostrf` / `sprintf` / `to_string`, but RAM-efficient enough to be used on microcontrollers, works uniformly on any platform, and properly handles NaNs and infinity.

Returns the string length without the terminating null character.

Examples:

<table class="ui compact small collapsing line table">
  <thead>
    <tr>
      <th class="right aligned">value</th>
      <th class="right aligned">prec</th>
      <th>str</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td class="right aligned">-0.321</td>
      <td class="right aligned">0</td>
      <td>"0"</td>
    </tr>
    <tr>
      <td class="right aligned">-0.321</td>
      <td class="right aligned">2</td>
      <td>"-0.32"</td>
    </tr>
    <tr>
      <td class="right aligned">-0.64</td>
      <td class="right aligned">0</td>
      <td>"-1"</td>
    </tr>
    <tr>
      <td class="right aligned">123.456</td>
      <td class="right aligned">0</td>
      <td>"123"</td>
    </tr>
    <tr>
      <td class="right aligned">123.456</td>
      <td class="right aligned">2</td>
      <td>"123.46"</td>
    </tr>
    <tr>
      <td class="right aligned">NaN</td>
      <td class="disabled right aligned">any</td>
      <td>"NaN"</td>
    </tr>
    <tr>
      <td class="right aligned">Inf</td>
      <td class="disabled right aligned">any</td>
      <td>"Inf"</td>
    </tr>
    <tr>
      <td class="right aligned">-Inf</td>
      <td class="disabled right aligned">any</td>
      <td>"-Inf"</td>
    </tr>
    <tr>
      <td class="right aligned">99000000000</td>
      <td class="disabled right aligned">any</td>
      <td>"OVF"</td>
    </tr>
    <tr>
      <td class="right aligned">-99000000000</td>
      <td class="disabled right aligned">any</td>
      <td>"-OVF"</td>
    </tr>
  </tbody>
</table>

<style>
/* Shift the content to assist better visual structure perception */
.ui.text.container p,
.ui.text.container table,
.ui.text.container pre {
  margin-left: 4ex;
}
</style>
