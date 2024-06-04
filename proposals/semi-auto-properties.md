# `field` keyword in properties

## Summary

Extend all properties to allow them to reference an automatically generated backing field using the new contextual keyword `field`. Properties may now also contain an accessor _without_ a body alongside an accessor _with_ a body.

## Motivation

Auto properties only allow for directly setting or getting the backing field, giving some control only by placing access modifiers on the accessors. Sometimes there is a need to have additional control over what happens in one or both accessors, but this confronts users with the overhead of declaring a backing field. The backing field name must then be kept in sync with the property, and the backing field is scoped to the entire class which can result in accidental bypassing of the accessors from within the class.

There are two common scenarios in particular: applying a constraint on the setter to ensuring the validity of a value, and raising an event such as `INotifyPropertyChanged.PropertyChanged`.

In these cases by now you always have to create an instance field and write the whole property yourself.  This not only adds a fair amount of code, but it also leaks the backing field into the rest of the type's scope, when it is often desirable to only have it be available to the bodies of the accessors.

## Glossary

- **Auto property**: Short for "automatically implemented property" ([§15.7.4](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/classes.md#1574-automatically-implemented-properties)). Accessors on an auto property have no body. The implementation and backing storage are both provided by the compiler.

- **Auto accessor**: Short for "automatically implemented accessor." This is an accessor that has no body. The implementation and backing storage are both provided by the compiler.

- **Full accessor**: This is an accessor that has a body. The implementation is not provided by the compiler, though the backing storage may still be (as in the example `set => field = value;`).

## Detailed design

For properties with an `init` accessor, everything that applies below to `set` would apply instead to the `init` accessor.

**Principle 1:** Every property declaration can be thought of as having a backing field by default, which is elided when not used. The field is referenced using the keyword `field` and its visibility is scoped to the accessor bodies.

**Principle 2:** `get;` will now be considered syntactic sugar for `get => field;`, and `set;` will now be considered syntactic sugar for `set => field = value;`.

Both of these principles only apply under the same conditions where `{ get; }` or `{ get; set; }` already declares an auto property ([§15.7.4](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/classes.md#1574-automatically-implemented-properties)). For example, abstract and interface properties are excluded. Indexers also remain unaffected.

"Auto property" will continue to mean a property whose accessors have no bodies. None of the examples below will be considered auto properties.

This means that properties may now mix and match auto accessors with full accessors. For example:

```cs
{ get; set => Set(ref field, value); }
```

```cs
{ get => field ?? parent.AmbientValue; set; }
```

Both accessors may be full accessors with either one or both making use of `field`:

```cs
{ get => field; set => field = value; }
```

```cs
{ get => field; set => throw new InvalidOperationException(); }
```

```cs
{ get => overriddenValue; set => field = value; }
```

```cs
{
    get;
    set
    {
        if (field == value) return;
        field = value;
        OnXyzChanged();
    }
}
```

Expression-bodied properties and properties with only a `get` accessor may also use `field`:

```cs
public string LazilyComputed => field ??= Compute();
```

```cs
public string LazilyComputed { get => field ??= Compute(); }
```

As with auto properties, a setter that uses a backing field is disallowed when there is no getter. This restriction could be loosened in the future to allow the setter to do something only in response to changes, by comparing `value` to `field` (see open questions).

```cs
// ❌ Error, will not compile
{ set => field = value; }
```

### Breaking changes

The existence of the `field` contextual keyword within property accessor bodies is a potentially breaking change, proposed as part of a larger [Breaking Changes](https://github.com/dotnet/csharplang/issues/7964) feature.

Since `field` is a keyword and not an identifier, it can only be "shadowed" by an identifier using the normal keyword-escaping route: `@field`. All identifiers named `field` declared within property accessor bodies can safeguard against breaks when upgrading from C# versions prior to 13 by adding the initial `@`.

### Field-targeted attributes

As with auto properties, any property that uses a backing field in one of its accessors will be able to use field-targeted attributes:

```cs
[field: Xyz]
public string Name => field ??= Compute();

[field: Xyz]
public string Name { get => field; set => field = value; }
```

A field-targeted attribute will remain invalid unless an accessor uses a backing field:

```cs
// ❌ Error, will not compile
[field: Xyz]
public string Name => Compute();
```

### Property initializers

Properties with initializers may use `field`. The backing field is directly initialized rather than the setter being called ([LDM decision](https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-03-02.md#open-questions-in-field)).

Calling a setter for an initializer is not an option; initializers are processed before calling base constructors, and it is illegal to call any instance method before the base constructor is called. This is also important for default initialization/definite assignment of structs.

This yields flexible control over initialization. If you want to initialize without calling the setter, you use a property initializer. If you want to initialize by calling the setter, you use assign the property an initial value in the constructor.

Here's an example of where this is useful. We believe the `field` keyword will find a lot of its use with view models because of the elegant solution it brings for the `INotifyPropertyChanged` pattern. View model property setters are likely to be databound to UI and likely to cause change tracking or trigger other behaviors. The following code needs to initialize the default value of `IsActive` without setting `HasPendingChanges` to `true`:

```cs
class SomeViewModel
{
    public bool HasPendingChanges { get; private set; }

    public bool IsActive { get; set => Set(ref field, value); } = true;

    private bool Set<T>(ref T location, T value)
    {
        if (RuntimeHelpers.Equals(location, value))
            return false;

        location = value;
        HasPendingChanges = true;
        return true;
    }
}
```

This difference in behavior between a property initializer and assigning from the constructor can also be seen with virtual auto properties in previous versions of the language:

```cs
using System;

// Nothing is printed; the property initializer is not
// equivalent to `this.IsActive = true`.
_ = new Derived();

class Base
{
    public virtual bool IsActive { get; set; } = true;
}

class Derived : Base
{
    public override bool IsActive
    {
        get => base.IsActive;
        set
        {
            base.IsActive = value;
            Console.WriteLine("This will not be reached");
        }
    }
}
```

### Constructor assignment

As with auto properties, assignment in the constructor calls the setter if it exists, and if there is no setter it falls back to directly assigning to the backing field.

```cs
class C
{
    public C()
    {
        P1 = 1; // Assigns P1's backing field directly
        P2 = 2; // Assigns P2's backing field directly
        P3 = 3; // Calls P3's setter
        P4 = 4; // Calls P4's setter
    }

    public int P1 => field;
    public int P2 { get => field; }
    public int P4 { get => field; set => field = value; }
    public int P3 { get => field; set; }
}
```

### Definite assignment in structs

Even though they can't be referenced in the constructor, backing fields denoted by the `field` keyword are subject to default-initialization and disabled-by-default warnings under the same conditions as any other struct fields ([LDM decision 1](https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-03-02.md#property-assignment-in-structs), [LDM decision 2](https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-05-02.md#definite-assignment-of-manually-implemented-setters)). For example:

```cs
public struct S
{
    public S()
    {
        _ = P1; // Disabled-by-default warning
        P2 = 5; // Disabled-by-default warning
    }

    public int P1 { get => field; }
    public int P2 { get => field; set => field = value; }
}
```

### Nullability

When `{ get; }` is written as `{ get => field; }`, or `{ get; set; }` is written as `{ get => field; set => field = value; }`, a similar warning should be produced when a non-nullable property is not initialized:

```cs
class C
{
    // ⚠️ CS8618: Non-nullable property 'P' must contain a
    // non-null value when exiting constructor.
    public string P { get => field; set => field = value; }
}
```

No warning should be produced if the property is initialized to a non-null value via constructor assignment or property initializer:

```cs
class C
{
    public C() { P = ""; }

    public string P { get => field; set => field = value; }
}
```

```cs
class C
{
    public string P { get => field; set => field = value; } = "";
}
```

In the same vein as how `var` infers as nullable for reference types, the `field` type is the nullable type of the property whenever the property's type is not a value type. Otherwise, `field ??` would appear to be followed by dead code, and it avoids producing a misleading warning in the following example:

```cs
public string AmbientValue
{
    get => field ?? parent.AmbientValue;
    set
    {
        if (value == parent.AmbientValue)
            field = null; // No warning here. Resume following the parent's value.
        else
            field = value; // Stop following the parent's value
    }
}
```

`var` was designed to declare nullability so that subsequent assignments to the variable could be nullable, due to established patterns in C#. It's expected that the same rationale would apply to property backing fields.

To land in this sweet spot implicitly, without having to write an attribute each time, nullability analysis will combine an inherent nullability of the field with the behavior of `[field: NotNull]`. This allows maybe-null assignments without warning, which is desirable as shown above, while simultaneously allowing a scenario like `=> field.Trim();` without requiring an intervention to silence a warning that `field` could be null. Making sure `field` has been assigned is already covered by the warning that ensures non-nullable properties are assigned by the end of each constructor.

This sweet spot does come with the downside that there would be no warning in this situation:

```cs
public string AmbientValue
{
    get => field; // No warning, but could return null!
    set
    {
        if (value == parent.AmbientValue)
            field = null;
        else
            field = value;
    }
}
```

Open question: Should flow analysis combine the maybe-null end state for `field` from the setter with the "depends on nullness of `field`" for the getter's return, enabling a warning in the scenario above?

### `nameof`

In places where `field` is a keyword, `nameof(field)` will fail to compile ([LDM decision](https://github.com/dotnet/csharplang/blob/main/meetings/2024/LDM-2024-05-15.md#usage-in-nameof)), like `nameof(nint)`. It is not like `nameof(value)`, which is the thing to use when property setters throw ArgumentException as some do in the .NET core libraries. In contrast, `nameof(field)` has no expected use cases.

### Overrides

Overriding properties may use `field`. Such usages of `field` refer to the backing field for the overriding property, separate from the backing field of the base property if it has one. There is no ABI for exposing the backing field of a base property to overriding classes since this would break encapsulation.

Like with auto properties, properties which use the `field` keyword and override a base property must override all accessors ([LDM decision](https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-05-02.md#partial-overrides-of-virtual-properties)).

### Captures

`field` should be able to be captured in local functions and lambdas, and references to `field` from inside local functions and lambdas are allowed even if there are no other references ([LDM decision 1](https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-03-21.md#open-question-in-semi-auto-properties), [LDM decision 2](https://github.com/dotnet/csharplang/blob/main/meetings/2024/LDM-2024-05-15.md#should-field-and-value-be-considered-keywords-in-lambdas-and-local-functions-within-property-accessors)):

```cs
public class C
{
    public static int P
    {
        get
        {
            Func<int> f = static () => field;
            return f();
        }
    }
}
```

## Specification changes

[§14.7.1](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/classes.md#1471-general) *Properties - General*

> A *property_initializer* may only be given for ~~an automatically implemented property, and~~ **a property that has a backing field that will be emitted and the property either does not have a setter, or its setter is auto-implemented. The *property_initializer*** causes the initialization of the underlying field of such properties with the value given by the *expression*.

[§14.7.4](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/classes.md#1474-automatically-implemented-properties) *Automatically implemented properties*

> An automatically implemented property (or ***auto-property*** for short), is a non-abstract non-extern
> property with ~~semicolon-only accessor bodies. Auto-properties must have a get accessor and can optionally~~
> ~~have a set accessor.~~ **either or both of:**
> 1. **an accessor with a semicolon-only body**
> 2. **usage of the `field` contextual keyword within the accessors or**
>    **expression body of the property. The `field` identifier is only considered the `field` keyword when there is**
>    **no existing symbol named `field` in scope at that location.**
> 
> When a property is specified as an ~~automatically implemented property~~ **auto-property**, a hidden **unnamed** backing field is automatically
> available for the property ~~, and the accessors are implemented to read from and write to that backing field~~.
> **For auto-properties, any semicolon-only `get` accessor is implemented to read from, and any semicolon-only**
> **`set` accessor to write to its backing field.**
> 
> **The backing field can be referenced directly using the `field` keyword**
> **within all accessors and within the property expression body. Because the field is unnamed, it cannot be used in a**
> **`nameof` expression.**
> 
> If the auto-property has ~~no set accessor~~ **only a semicolon-only get accessor**, the backing field is considered `readonly` ([§14.5.3](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/classes.md#1453-readonly-fields)).
> Just like a `readonly` field, a getter-only auto property **(without a set accessor or an init accessor)** can also be assigned to in the body of a constructor
> of the enclosing class. Such an assignment assigns directly to the ~~readonly~~ backing field of the property.
> 
> **An auto-property is not allowed to only have a single semicolon-only `set` accessor without a `get` accessor.**

The following example:
```csharp
// No 'field' symbol in scope.
public class Point
{
    public int X { get; set; }
    public int Y { get; set; }
}
```
is equivalent to the following declaration:
```csharp
// No 'field' symbol in scope.
public class Point
{
    public int X { get { return field; } set { field = value; } }
    public int Y { get { return field; } set { field = value; } }
}
```
which is equivalent to:
```csharp
// No 'field' symbol in scope.
public class Point
{
    private int __x;
    private int __y;
    public int X { get { return __x; } set { __x = value; } }
    public int Y { get { return __y; } set { __y = value; } }
}
```

The following example:
```csharp
// No 'field' symbol in scope.
public class LazyInit
{
    public string Value => field ??= ComputeValue();
    private static string ComputeValue() { /*...*/ }
}
```
is equivalent to the following declaration:
```csharp
// No 'field' symbol in scope.
public class Point
{
    private string __value;
    public string Value { get { return __value ??= ComputeValue(); } }
    private static string ComputeValue() { /*...*/ }
}
```

## Open LDM questions

1. Which of these scenarios should be allowed to compile? Assume that the "field is never read" warning would apply just like with a manually declared field.

   1. `{ set; }` - Disallowed today, continue disallowing
   1. `{ set => field = value; }`
   1. `{ get => unrelated; set => field = value; }`
   1. `{ get => unrelated; set; }`
   1. ```cs
      {
          set
          {
              if (field == value) return;
              field = value;
              SendEvent(nameof(Prop), value);
          }
      }
      ```
   1. ```cs
      {
          get => unrelated;
          set
          {
              if (field == value) return;
              field = value;
              SendEvent(nameof(Prop), value);
          }
      }
      ```

1. Should nullable flow analysis provide a warning for non-nullable properties when a setter allows `field` to be null and the getter returns something whose nullability depends on `field`?

## LDM history:
- https://github.com/dotnet/csharplang/blob/main/meetings/2021/LDM-2021-03-10.md#field-keyword
- https://github.com/dotnet/csharplang/blob/main/meetings/2021/LDM-2021-04-14.md#field-keyword
- https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-01-12.md#open-questions-for-field
- https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-02-16.md#open-questions-in-field
- https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-03-02.md#open-questions-in-field
- https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-03-21.md#open-question-in-semi-auto-properties
- https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-05-02.md#field-questions
