* Proposal: SE-XXXX
* Author: [Kelvin Ma](https://github.com/kelvin13) ([@*taylorswift*](https://forums.swift.org/u/taylorswift/summary))
* Review manager: 
* Status: *Awaiting review*
* Implementation: [apple/swift#19936](https://github.com/apple/swift/pull/19936)
* Threads: [1](https://forums.swift.org/t/prepitch-character-integer-literals/10442)

## Introduction

Character processing is everywhere - particularly when working with ASCII.  However, Swift's support for working with it is awkward and suboptimal.  This proposal seeks to improve character processing support by allowing Swift integers to be written using single quoted character literals, which is much more consistent with other languages and will reduce a point of friction in string processing code.

```swift
let niceInteger: Int8 = 'E'
```

## Motivation 

In C, `'a'`, is a `char` (`uint8_t`) literal, equivalent to `97`. Swift has no such equivalent, requiring awkward spellings like `("a" as Unicode.Scalar).value`, which may or may not need additional casts to convert it to the right integer type, or `UInt8(ascii: "a")` in the case of `UInt8`. Alternatives, like spelling out the values in hex or decimal directly, are even worse.  This harms readability of code, and is one of the sore points of string processing in Swift.

There are many examples where this causes harm.  First, consider this simple table in C:

```c
static char const hexcodes[16] = {
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 
    'a', 'b', 'c', 'd', 'e', 'f'
};
```

It has to be written like this in Swift:

```swift
// what do these numbers mean???
let hexcodes: [UInt8] = [
    48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 97, 98, 99, 100, 101, 102
] 
    
// This is the best we can get right now, while showing the ascii letter form.
let hexcodes = [
    UInt8(ascii: "0"), UInt8(ascii: "1"), UInt8(ascii: "2"), UInt8(ascii: "3"),
    UInt8(ascii: "4"), UInt8(ascii: "5"), UInt8(ascii: "6"), UInt8(ascii: "7"),
    UInt8(ascii: "8"), UInt8(ascii: "9"), UInt8(ascii: "a"), UInt8(ascii: "b"),
    UInt8(ascii: "c"), UInt8(ascii: "d"), UInt8(ascii: "e"), UInt8(ascii: "f")
]    
```

`UInt8` has a convenience initializer for converting from ASCII, but if you're working with other types like Int8 (common when dealing with C APIs that take `char`, it is much more awkward.  Consider scanning through a `char*` buffer as an `UnsafeBufferPointer<Int8>`:

```swift
for scalar in int8buffer {
    switch scalar {
    case Int8(UInt8(ascii: "a")) ... Int8(UInt8(ascii: "f")):
        // lowercase hex letter
    case Int8(UInt8(ascii: "A")) ... Int8(UInt8(ascii: "F")):
        // uppercase hex letter
    case Int8(UInt8(ascii: "0")) ... Int8(UInt8(ascii: "9")):
        // hex digit
    default:
        // something else
    }
}
```

## Proposed solution 

Let's do the obvious thing here, and allow `'x'` to be a literal corresponding to the unicode representation of the contained character value.  The standard library will adopt this syntax on integer types, `Character`, `Unicode.Scalar`, and types like `UTF16.CodeUnit`.  The default literal type for `var x = 'a'` will be `Character`.

With these changes, the above code can be written much more naturally:

```swift
let hexcodes: [UInt8] = [
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 
    'a', 'b', 'c', 'd', 'e', 'f'
] 

for scalar in int8buffer {
    switch scalar {
    case 'a' ... 'f':
        // lowercase hex letter
    case 'A' ... 'F':
        // uppercase hex letter
    case '0' ... '9':
        // hex digit
    default:
        // something else, perhaps an extended UTF8 digit.
    }
}
```

### Choice of single quotes

Use of single quotes for character/scalar literals is *heavily* precedented in other languages, including C, Objective-C, C++, Java, and Rust, although different languages have slightly differing ideas about what a “character” is.  We choose to use single quote syntax specifically because it reinforces the notion that strings and character values are different: the former is a sequence, the later is a scalar (and "integer-like").  Character types also don't support string literal interpolation, which is another reason to move away from double quotes.

One significant corner case is worth mentioning: some methods may be overloaded on both `Character` and `String`.  This design allows natural user-side syntax for differentiating between the two.

### Single quotes in Swift, a historical perspective

In Swift 1.0, we wanted to reserve single quotes for some yet-to-be determined syntactical purpose. However, today, pretty much all of the things that we once thought we might want to use single quotes for have already found homes in other parts of the Swift syntactical space.  For example, syntax for [multi-line string literals](https://github.com/apple/swift-evolution/blob/master/proposals/0168-multi-line-string-literals.md) uses triple quotes (`"""`), and string interpolation syntax uses standard double quote syntax. 

Current proposals for [raw string literals](https://github.com/apple/swift-evolution/blob/master/proposals/0200-raw-string-escaping.md) use r-prefixes (`r"`).  For [regex literals](https://forums.swift.org/t/string-update/7398/6), most people seem to prefer slashes (`/`), but they could also fall into the same syntax as raw strings.

At this point, it is clear that the early syntactic conservatism was unwarranted.  We do not forsee another use for this syntax, and given the strong precedent in other languages for characters, it is natural to use it.

### Existing double quote initializers for characters

We propose deprecating the double quote initializer for `Character` and unicode scalar types and slowly migrating them out of Swift.

```swift
let c2 = 'f'               // preferred
let c1 : Character = "f"   // deprecated
```

### Overflow checking

Just as the compiler and standard library work together to detect overflowing integer literals, overflowing character literals will be statically diagnosed:

```swift
let a: Int16 = 128 // ok
let b: Int8 = 128  // error: integer literal '128' overflows when stored into 'Int8' 

let c: Int16 = 'Ƥ' // ok
let d: Int8  = 'Ƥ' // error: character literal 'Ƥ' overflows when stored into 'Int8' 
```

## Detailed Design 

These new `character` literals will have default type `Character` and be statically checked to contain only a single extended grapheme cluster. They will be processed largely as if they were a short `String`.

When the `Character` is representable by a single UNICODE `codepoint` however, (a 20 bit number) they will be able to express a Unicode.Scalar and any of the integer types provided the codepoint value fits into that type.

As an example:

```
let a = 'a' // This will have type Character
let s: Unicode.Scalar = 'a' // is also possible
let i: Int8 = 'a' // takes the ASCII value
```
In order to implement this a new protocol `ExpressibleByCodepointLiteral` is created
and used for character literals that are also a single codepoint instead of `ExpressibleByExtendedGraphemeClusterLiteral`. Conformances to this protocol for `Unicode.Scalar`, `Character` and `String` will be added to the standard library so these literals can operate in any of those roles. In addition, conformances to `ExpressibleByCodepointLiteral` will be added to all integer types in the standard library so a character literal can initialize variables of integer type (subject to a compile time range check) or satisfy the type checker for arguments of these integer types. 

These new conformances and the existing operators defined in the Swift language will make the following code possible out of the box:

```
func unhex(_ hex: String) throws -> Data {
    guard hex.utf16.count % 2 == 0 else {
        throw NSError(domain: "odd characters", code: -1, userInfo: nil)
    }

    func nibble(_ char: UInt16) throws -> UInt16 {
        switch char {
        case '0' ... '9':
            return char - '0'
        case 'a' ... 'f':
            return char - 'a' + 10
        case 'A' ... 'F':
            return char - 'A' + 10
        default:
            throw NSError(domain: "bad character", code: -1, userInfo: nil)
        }
    }

    var out = Data(capacity: hex.utf16.count/2)
    var chars = hex.utf16.makeIterator()
    while let next = chars.next() {
        try out.append(UInt8((nibble(next) << 4) + nibble(chars.next()!)))
    }

    return out
}
```
One area which may involve ambiguity is the `+` operator which can mean either String concatenation or addition on integer values. Generally this wouldn't be a problem as most reasonable contexts will provide the type checker the information to make the correct decision.

```
print('1' + '1' as String) // prints 11
print('1' + '1' as Int) // prints 98
```

## Source compatibility 

This proposal could be done in a way that is strictly additive, but we feel it is best to deprecate the existing double quote initializers for characters, and the `UInt8.init(ascii:)` initializer.  

Here is a specific sketch of a deprecation policy: 
  1) continue accepting these in Swift 4 mode with no change.  
  2) Introduce the new syntax support into Swift 4.2 (if there is time).  
  3) Swift 5 mode would start producing deprecation warnings (with a fixit to change double quotes to single quotes).
  4) The Swift 4 to 5 migrator would change the syntax (by virtue of applying the deprecation fixits).
  4) Swift 6 would not accept the old syntax.

## Effect on ABI stability 

No effect as this is an additive change.  Heroic work could be done to try to prevent the `UInt8.init(ascii:)` initializer and other to-be-deprecated conformances from being part of the ABI.  This seems unnecessary though.

## Effect on API resilience 

None. 

## Alternatives considered 

### Why not make this apply to `Character` too?

**TODO** (from clattner): I really don't think this is a good idea.  We should have a consistent character model.

Lots of people suggested this in the prepitch thread for this proposal. Since `Character` *literals* and `Unicode.Scalar` *literals* would look exactly the same (since we would have to make `Character` take single quotes too), there would be zero sugaring benefit gained from this. 

Some people like the idea of making the quote type indicate the “length” (1 or greater than 1) of the literal object, in which case `Character` would take single quotes as well. This notational change is orthogonal to this proposal and can be done separately in another proposal. 

 It was not possible to re-use the `ExpressibleByUnicodeScalarLiteral` protocol as this is historically an ancestor of `ExpressibleByStringLiteral` and changing conformances to it would affect double quoted string literals.