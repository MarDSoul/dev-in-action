# Flexible Enums via Inline classes

- date: Jan 2026
- link: personal

When needs in some extended `enum` solution:

```kotlin
@JvmInline 
value class SomeConstList(val value: String) {
    companion object {
        val GOOGLE = SomeConstList("google")
        val EMAIL = SomeConstList("email")
        //etc
        fun custom(value: String) = SomeConstList(value)
    }
}
```
