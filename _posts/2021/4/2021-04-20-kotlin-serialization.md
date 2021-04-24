---
layout: post
date: 2021-04-20
title: "Writing our own compiler plugin for serialization using Kotlin"
author: akuleshov7
description: |
  How to create your own serializer using kotlinx.serialization
keywords:
  - kotlin
  - serialization
---

**Serialization** and **deserialization** - are two sides of the same mechanism for storing (persisting)
of objects or sending these objects over the network. There are a lot of different formats used
for serialization. In general they can be split into two main groups: 
binary protocols ([pickle](https://docs.python.org/3/library/pickle.html), [protobuf](https://developers.google.com/protocol-buffers) and many other) 
and protocols with a string representation like [json](https://en.wikipedia.org/wiki/JSON),
[toml](https://toml.io/en/), [yaml](https://en.wikipedia.org/wiki/YAML),
[csv](https://en.wikipedia.org/wiki/Comma-separated_values), e.t.c. We know a lot of different libraries for the serialization in Java,
but in Kotlin it was decided to create a common framework for the serialization
called [kotlinx.serialization](https://github.com/Kotlin/kotlinx.serialization). 
Let's have a look how this framework works and try to write our own serializer. 
<!--more-->
  
### Kotlinx.serialization
Let's start from the most important thing: from documentation!
Actually kotlinx.serialization has [perfect huge guide](https://github.com/Kotlin/kotlinx.serialization/blob/master/docs/serialization-guide.md) about the serialization.
But we will try to be short and practical. We will try to create a deserialization library for the deserialization of our map-like format.
*Spoiler:* we will try to reimplement deserialization library for map-like format using kotlin compiler. Recently such functionality was implemented in [Properties](https://kotlin.github.io/kotlinx.serialization/kotlinx-serialization-properties/kotlinx-serialization-properties/kotlinx.serialization.properties/-properties/index.html) library.

All code that you will see in this post can be found on [github](https://github.com/akuleshov7/kotlinx-serialization-map)

### Let's start!

Imagine that we have the following simple map-like format (similar to `json` or any othe serialization format, isn't it?):
```text
(a: (b: "a"; c:5.0); d: 6; e: "my string")
```   

I am pretty sure that everyone understands that we **do not like** to work with such input using map:
```kotlin
val myMap: Map<String, Any> = parse("(a: (b: "a"; c:5.0); d: 6; e: \"my string\")")
// this string will not be converted to Int
val bugNumberOne: Int? = myMap.get("e") as Int?
// in this map there is no key "f" and we will get unexpectedly null, also we will need to do casting of a type
val bugNumberTwo: String? = myMap.get("f")
```

At least because we would like compiler to check us and help us to prevent issues with
incorrect types or incorrect keys. We would like to have a perfect data class where
this input will be deserialized. We would like to have the following data structure:
```kotlin
@Serializable 
data class MyInnerClass(
    val a: String,
    val b: Double
) 

@Serializable 
data class MySerializationClass(
    val a: MyInnerClass,
    val b: Int,
    val e: String
)
```
 
Let's modify a little bit the schema you can see in [kotlinx documentation](https://github.com/Kotlin/kotlinx.serialization/blob/master/docs/basic-serialization.md#basics). 
We will do the following:

```text
+--------+ Parsing +-----+ Decoding +------------+  Deserialization +--------+
| Input  | --(1)-->| Map | ---(2)-->| Primitives |  -----(3)---->   | Object |
+--------+         +-----+          +------------+                  +--------+   
```

### Let's see some code
Let's skip step **(1)** - parsing - as it is obvious how to parse our input to a map. 

Imagine that we already have a map and move to the step **(2)**:
```kotlin
val innerMap = mapOf("b" to "a", "c" to 5.0)
val resMap = mapOf("a" to innerMap, "d" to 6, "e" to "my string")
// we would like to parse the resMap to MySerializationClass
val obj = MapSerialization.decodeFromString<MySerializationClass>(resMap)
```

### AbstractDecoder
The deserialization of primitives **(3)** will be done by the Koltin compiler, so we only need to do a step **(2)** and help the compiler
to decode the map to primitives. To do this we need to implement `AbstractDecoder` from `kotlinx.serialization.encoding.AbstractDecoder`:
```kotlin
// Serialization API is still in Alpha stage, so we need explicitly confirm that we understand what we are doing with the following annotation:
@OptIn(ExperimentalSerializationApi::class)
class MapDecoder(
    // our map that will be decoded
    val map: Map<*, *>, 
    // number of elements that are contained in the current data structure that we would like to deserialize
    var elementsCount: Int = 0, 
    // some  config that will be used in decoding process
    val config: Config = Config.default 
) : AbstractDecoder() {
    private var elementIndex = 0
    // ...
}
```

### Iteration through complex nested objects
We need to iterate through all objects including nested objects, to do this we need to implement `beginStructure()` method. 
The iteration will go through all fields and nested structures declared in the data class `MySerializationClass` in the same order how all fields were declared.

```kotlin
    // used to trigger the processing for structures (including nested)
    // when we use this method we go throw nested (non-primitive) structures IN THE CLASS
    override fun beginStructure(descriptor: SerialDescriptor): CompositeDecoder {
        // corner case at the beginning of the decoding
        if (elementIndex == 0) {
            // sanity check to valide that we have correct format of input
            validateDecodedInput(descriptor, map)
            return MapDecoder(map, descriptor.elementsCount)
        } else {
            // need to decrement element index, as unfortunately it was incremented in the iteration of `decodeElementIndex`
            return when (val innerMap = map.values.elementAt(elementIndex - 1)) {
                is Map<*, *> -> {
                    validateDecodedInput(descriptor, innerMap)
                    MapDecoder(innerMap, descriptor.elementsCount)
                }
                else -> throw MapDecodingException("Incorrect format of nested data provided." +
                        " Expected map, but received: <$innerMap>")
            }
        }
    }
``` 

`beginStructure()` - is an entry point for all non-trivial non-primitive values that will appear in the input.
Pay attention that the `descriptor: SerialDescriptor` contains all information about the current structure (number of fields, names of fields, e.t.c)

Also we will need to help the decoder to iterate through indexes and help to understand when the processing should be stopped. 
For this we need to implement `decodeElementIndex()` function. It should return the **index** of the field from the class
**where the value** from the input **should be inserted**. We need simply to do `descriptor.getElementIndex("fieldName")` for it.
In case such field `fieldName` will be missing in our class then the method will return `UNKNOWN_NAME` (equals to -3)

```
    /**
     * this method should be overridden to map the FIELD in your class to the VALUE from
     * the input that you would like to inject into this field
     */
    override fun decodeElementIndex(descriptor: SerialDescriptor): Int {
        if (elementIndex == map.size) return CompositeDecoder.DECODE_DONE

        val fieldName = map.keys.elementAt(elementIndex).toString()

        // index of the field from the class where we should inject our value
        val fieldWhereValueShouldBeInjected =
            descriptor.getElementIndex(map.keys.elementAt(elementIndex).toString())

        if (fieldWhereValueShouldBeInjected == CompositeDecoder.UNKNOWN_NAME) {
            if (config.isStrict) {
                throw MapDecodingException(
                    "Unknown property <$fieldName>." +
                            " To ignore unknown properties use 'isStrict = false' in the Config for the MapDecoder"
                )
            }
        }
        elementIndex++

        return fieldWhereValueShouldBeInjected
    }
```

From the code you can see that we will stop the process when we have iterated through all values in the current structure:
```
   if (elementIndex == map.size) return CompositeDecoder.DECODE_DONE
```

### decodeValue
We should also override a special method `decodeValue` that is simply used to return the value that we will
get using the `elementIndex` (incremented in `decodeElementIndex` method):
```
    override fun decodeValue(): Any {
        val keyAtTheCurrentIndex = map.keys.elementAt(elementIndex - 1)
        return map[keyAtTheCurrentIndex]!!
    }
```

### How it will work?
- We will iterate through all fields in our `MySerializationClass`,
trigger `beginStructure()` for every complex structure we will find in our data class (for example `MyInnerClass`);
- For each complex structure we will recursively create a new instance of `MapDecoder`;
- In each new `MapDecoder` we will have a counter `elementIndex` that will be incremented in `decodeElementIndex()`;
- `decodeElementIndex()` will return us the index of the field in data class where our value should be inserted to;
- `decodeValue()` will calculate and return the value after decoding. This value will be assigned to the
 corresponding field (that has the index returned by `decodeElementIndex()`)
 
So from this string: `(a: (b: "a"; c:5.0); d: 6; e: "my string")` instead of map we can have a pretty data class:
```
@Serializable 
data class MySerializationClass(
    val a: MyInnerClass, // MyInnerClass("a", 5.0)
    val b: Int, // 6
    val e: String // "my string"
)
```

So easy! Isn't it? Imagine how many hours you will have spent to implement this functionality without such awesome compiler framework. 
Also it is very important that `kotlinx.serialization` will work with [Kotlin Native](https://kotlinlang.org/docs/native-overview.html), and not only with Kotlin JVM!

### Sources
Full version of the code can be found on [github](https://github.com/akuleshov7/kotlinx-serialization-map/blob/main/src/mapSerializationMain/kotlin/com/akuleshov7/kotlinx/serialization/map/decoders/MapDecoder.kt)

Also many thanks to Kotlin community for outstanding kotlinx library and perfect documentation.

Also have a look at my library that I have created for deserialization of toml format: [github](https://github.com/akuleshov7/ktoml) 
