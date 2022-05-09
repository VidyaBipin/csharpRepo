# Numeric IntPtr

## Summary
[summary]: #summary

This is a revision on the initial native integers feature ([spec](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-9.0/native-integers.md)), where the `nint`/`nuint` types were distinct from the underlying types `System.IntPtr`/`System.UIntPtr`.
In short, we now treat `nint`/`nuint` as simple types aliasing `System.IntPtr`/`System.UIntPtr`, like we do for `int` in relation to `System.Int32`.

## Design
[design]: #design

### 8.3.5 Simple types

C# provides a set of predefined `struct` types called the simple types. The simple types are identified through keywords, but these keywords are simply aliases for predefined `struct` types in the `System` namespace, as described in the table below.

**Keyword** | **Aliased type**
----------- | ------------------
  `sbyte`   |   `System.SByte`
  `byte`    |   `System.Byte`
  `short`   |   `System.Int16`
  `ushort`  |   `System.UInt16`
  `int`     |   `System.Int32`
  `uint`    |   `System.UInt32`
  **`nint`**     |   **`System.IntPtr`**
  **`nuint`**    |   **`System.UIntPtr`**
  `long`    |   `System.Int64`
  `ulong`   |   `System.UInt64`
  `char`    |   `System.Char`
  `float`   |   `System.Single`
  `double`  |   `System.Double`
  `bool`    |   `System.Boolean`
  `decimal` |   `System.Decimal`

\[...]

### 8.3.6 Integral types

C# supports **eleven** integral types: `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, and `char`. \[...]

## 8.8 Unmanaged types

In other words, an __unmanaged_type__ is one of the following:
- `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, `char`, `float`, `double`, `decimal`, or `bool`.
- Any *enum_type*.
- Any user-defined *struct_type* that is not a constructed type and contains fields of *unmanaged_type*s only.
- In unsafe code, any *pointer_type*.

### 10.2.3 Implicit numeric conversions

The implicit numeric conversions are:

- From `sbyte` to `short`, `int`, **`nint`**, `long`, `float`, `double`, or `decimal`.
- From `byte` to `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, `float`, `double`, or `decimal`.
- From `short` to `int`, **`nint`**, `long`, `float`, `double`, or `decimal`.
- From `ushort` to `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, `float`, `double`, or `decimal`.
- From `int` to **`nint`**, `long`, `float`, `double`, or `decimal`.
- From `uint` to **`nuint`**, `long`, `ulong`, `float`, `double`, or `decimal`.
- **From `nint` to `long`, `float`, `double`, or `decimal`.**
- **From `nuint` to `ulong`, `float`, `double`, or `decimal`.**
- From `long` to `float`, `double`, or `decimal`.
- From `ulong` to `float`, `double`, or `decimal`.
- From `char` to `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, `float`, `double`, or `decimal`.
- From `float` to `double`.

\[...]

### 10.2.11 Implicit constant expression conversions

An implicit constant expression conversion permits the following conversions:

- A *constant_expression* of type `int` can be converted to type `sbyte`, `byte`, `short`, `ushort`, `uint`, **`nint`, `nuint`**, or `ulong`, provided the value of the *constant_expression* is within the range of the destination type.
\[...]

### 10.3.2 Explicit numeric conversions

The explicit numeric conversions are the conversions from a *numeric_type* to another *numeric_type* for which an implicit numeric conversion does not already exist:

- From `sbyte` to `byte`, `ushort`, `uint`, **`nuint`**, `ulong`, or `char`.
- From `byte` to `sbyte` or `char`.
- From `short` to `sbyte`, `byte`, `ushort`, `uint`, **`nuint`**, `ulong`, or `char`.
- From `ushort` to `sbyte`, `byte`, `short`, or `char`.
- From `int` to `sbyte`, `byte`, `short`, `ushort`, `uint`, **`nuint`**, `ulong`, or `char`.
- From `uint` to `sbyte`, `byte`, `short`, `ushort`, `int`, **`nint`**, or `char`.
- From `long` to `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `ulong`, or `char`.
- **From `nint` to `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, `nuint`, `ulong`, or `char`.**
- **From `nuint` to `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, `nint`, `long`, or `char`.**
- From `ulong` to `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, or `char`.
- From `char` to `sbyte`, `byte`, or `short`.
- From `float` to `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, `char`, or `decimal`.
- From `double` to `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, `char`, `float`, or `decimal`.
- From `decimal` to `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, `char`, `float`, or `double`.

\[...]

### 10.3.3 Explicit enumeration conversions

The explicit enumeration conversions are:

- From `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, `char`, `float`, `double`, or `decimal` to any *enum_type*.
- From any *enum_type* to `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, `char`, `float`, `double`, or `decimal`.
- From any *enum_type* to any other *enum_type*.

### 11.8.2 Unary plus operator

The predefined unary plus operators are:

```csharp
...
nint operator +(nint x);
nuint operator +(nuint x);
```

# 11 Expressions

### 11.7.10 Element access

\[...] The number of expressions in the *argument_list* shall be the same as the rank of the *array_type*, and each expression shall be of type `int`, `uint`, **`nint`, `nuint`**, `long`, or `ulong,` or shall be implicitly convertible to one or more of these types.

#### 11.7.10.2 Array access

### 11.7.14 Postfix increment and decrement operators

Unary operator overload resolution is applied to select a specific operator implementation. Predefined `++` and `--` operators exist for the following types: `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`,** `long`, `ulong`, `char`, `float`, `double`, `decimal`, and any enum type.

### 11.8.3 Unary minus operator

The predefined unary minus operators are:

- Integer negation:

  ```csharp
  ...
  nint operator –(nint x);
  ```  

### 11.7.14 Postfix increment and decrement operators

Predefined `++` and `--` operators exist for the following types: `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, `char`, `float`, `double`, `decimal`, and any enum type.

### 11.7.19 Default value expressions

In addition, a *default_value_expression* is a constant expression if the type is one of the following value types: `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, `char`, `float`, `double`, `decimal`, `bool,` or any enumeration type.

### 11.8.5 Bitwise complement operator

The predefined bitwise complement operators are:

```csharp
...
nint operator ~(nint x);
nuint operator ~(nuint x);
```

### 11.8.6 Prefix increment and decrement operators

Predefined `++` and `--` operators exist for the following types: `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, `char`, `float`, `double`, `decimal`, and any enum type.

## 11.9 Arithmetic operators

### 11.9.2 Multiplication operator

The predefined multiplication operators are listed below. The operators all compute the product of `x` and `y`.

- Integer multiplication:

  ```csharp
  ...
  nint operator *(nint x, nint y);
  nuint operator *(nuint x, nuint y);
  ```

### 11.9.3 Division operator

The predefined division operators are listed below. The operators all compute the quotient of `x` and `y`.

- Integer division:

  ```csharp
  ...
  nint operator /(nint x, nint y);
  nuint operator /(nuint x, nuint y);
  ```

### 11.9.4 Remainder operator

The predefined remainder operators are listed below. The operators all compute the remainder of the division between `x` and `y`.

- Integer remainder:

  ```csharp
  ...
  nint operator %(nint x, nint y);
  nuint operator %(nuint x, nuint y);
  ```

### 11.9.5 Addition operator

- Integer addition:

  ```csharp
  ...
  nint operator +(nint x, nint y);
  nuint operator +(nuint x, nuint y);
  ```

### 11.9.6 Subtraction operator

- Integer subtraction:

  ```csharp
  ...
  nint operator –(nint x, nint y);
  nuint operator –(nuint x, nuint y);
  ```

## 11.10 Shift operators

The predefined shift operators are listed below.

- Shift left:

  ```csharp
  ...
  nint operator <<(nint x, int count);
  nuint operator <<(nuint x, int count);
  ```

- Shift right:

  ```csharp
  ...
  nint operator >>(nint x, nint count);
  nuint operator >>(nuint x, nint count);
  ```

  The `>>` operator shifts `x` right by a number of bits computed as described below.

  When `x` is of type `int`, **`nint`** or `long`, the low-order bits of `x` are discarded, the remaining bits are shifted right, and the high-order empty bit positions are set to zero if `x` is non-negative and set to one if `x` is negative.

  When `x` is of type `uint`, **`nuint`** or `ulong`, the low-order bits of `x` are discarded, the remaining bits are shifted right, and the high-order empty bit positions are set to zero.

- Unsigned shift right:

  ```csharp
  ...
  nint operator >>>(nint x, nint count);
  nuint operator >>>(nuint x, nint count);
  ```

## 11.11 Relational and type-testing operators

### 11.11.2 Integer comparison operators

The predefined integer comparison operators are:

```csharp
...
bool operator ==(nint x, nint y);
bool operator ==(nuint x, nuint y);

bool operator !=(nint x, nint y);
bool operator !=(nuint x, nuint y);

bool operator <(nint x, nint y);
bool operator <(nuint x, nuint y);

bool operator >(nint x, nint y);
bool operator >(nuint x, nuint y);

bool operator <=(nint x, nint y);
bool operator <=(nuint x, nuint y);

bool operator >=(nint x, nint y);
bool operator >=(nuint x, nuint y);
```

## 11.12 Logical operators

### 11.12.2 Integer logical operators

The predefined integer logical operators are:

```csharp
...
nint operator &(nint x, nint y);
nuint operator &(nuint x, nuint y);

nint operator |(nint x, nint y);
nuint operator |(nuint x, nuint y);

nint operator ^(nint x, nint y);
nuint operator ^(nuint x, nuint y);
```

## 11.20 Constant expressions

A constant expression may be either a value type or a reference type. If a constant expression is a value type, it must be one of the following types: `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, **`nint`, `nuint`**, `long`, `ulong`, `char`, `float`, `double`, `decimal`, `bool,` or any enumeration type.

\[...]

An implicit constant expression conversion permits a constant expression of type `int` to be converted to `sbyte`, `byte`, `short`, `ushort`, `uint`, **`nint`, `nuint`,** or `ulong`, provided the value of the constant expression is within the range of the destination type.

TODO pointer arithmetic
TODO dynamic
TODO Possible breaking changes: IntPtr + int, IntPtr - int, (int)IntPtr, (IntPtr)long (and then of course various possible edge cases where user-defined vs language-defined operators subtly differ in behavior)
TODO Call out behavior with operators defined on IntPtr/UIntPtr
