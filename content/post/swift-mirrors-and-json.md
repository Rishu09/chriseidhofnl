---
aliases:
  - /posts/swift-mirrors-and-json.html
date: 2015-06-15
title: Swift Mirrors and JSON
headline: Generating JSON dictionaries with reflection
---


Inspired by [Mike Ash's tweet](https://twitter.com/mikeash/status/609575730024984576), I tried generating JSON dictionaries in Swift by using the new reflection features. First of all, let's create two structs for holding a person and an address:

    struct Address {
        var street: String
    }
    
    struct Person {
        var name: String = "John"
        var age: Int = 42
        var dutch: Bool = false
        var address: Address? = Address(street: "Market St.")
    }
    
    let john = Person()

We can introspect values using the `reflect` function. It returns a `MirrorType`, which is a value we can use to inspect the type:

    reflect(john)

Every `MirrorType` value has a property `count`, which counts the number of children (e.g. the address struct has a single child, `street`, whereas the person struct has four children, one for every field). We can extend `MirrorType` with a simple `children` property that loops over the children and returns them in an array:

    extension MirrorType {
        var children: [(String,MirrorType)] {
            var result: [(String, MirrorType)] = []
            for i in 0..<self.count {
                result.append(self[i])
            }
            return result
        }
    }

We will implement our solution in two steps. First, we'll create a protocol `JSON` which converts any value into a JSON-serializable object. Because serialization might fail, we'll mark it as `throws`: 

    protocol JSON {
        func toJSON() throws -> AnyObject?
    }

Now comes the big trick I copied from Mike. We can provide a default implementation by extending the protocol. This default implementation reflects a value, and loops over the children, recursively serializing them as well. If the type doesn't have any children, we assume it's a primitive (e.g. a String or an Int) and don't serialize it.

    extension JSON {
        func toJSON() throws -> AnyObject? {
            let mirror = reflect(self)
            if mirror.count > 0  {
                var result: [String:AnyObject] = [:]
                for (key, child) in mirror.children {
                    if let value = child.value as? JSON {
                      result[key] = try value.toJSON()
                    } else {
                        throw CouldNotSerializeError.NoImplementation(source: self, type: child)
                    }
                }
                return result
            }
            return self as? AnyObject
        }
    }

Because of the implementation above, we can now easily derive JSON generation for structs and simple values, without having to implement it:

    extension Address: JSON { }
    extension Person: JSON { }
    extension String: JSON { }
    extension Int: JSON { }
    extension Bool: JSON { }

For optional values, we want to return nil in case the value isn't there, and otherwise try to serialize it. This is how you can override the `toJSON` function for a more specific type:

    extension Optional: JSON {
        func toJSON() throws -> AnyObject? {
            if let x = self {
                if let value = x as? JSON {
                    return try value.toJSON()
                }
                throw CouldNotSerializeError.NoImplementation(source: x, type: reflect(x))
            }
            return nil
        }
    }

Finally, to test that it worked:

    do {
        try john.toJSON()
    } catch {
        print(error)
    }

The full code is in a [gist](https://gist.github.com/chriseidhof/48243eb549481bc38d58). It runs fine in a real app, but seems to not always work in a playground. After posting the gist on Twitter yesterday, Rich pointed out that I was late to the party: Matthew Cheok already implemented [JSONCodable](https://github.com/matthewcheok/JSONCodable), which is a library that's ready for use!

