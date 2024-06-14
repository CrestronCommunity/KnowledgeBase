# SIMPL Sharp/C# Notes

## General Notes for S+/S# integration
* All C# classes that are used in S+ MUST have a default constructor. This even includes EventArgs classes used to pass information in a callback. They may have other constructors, but the default constructor must exist for S+ to recognize the class.
* `DIGITAL` or `INTEGER` or `ANALOG` in S+ is `ushort` in C#
* `SIGNED INTEGER` in S+ is `short` in C#
* `STRING` in S+ is `string` in C#
* Reference C# libraries with `#USER_SIMPLSHARP_LIBRARY "LibraryName"` in S+ (do not include the `.clz` extension)

### Printing TRACE messages
Normally printing to the console is the same as `PRINT()` in S+, but it is possible to format the string to appear as if it were from `TRACE()`. According to the SIMPL+ Help: "TRACE prepends the hex  characters `\xFA\xE0`, and suffixes the string with `\xFB`."

```csharp
void Print(string message) {
	CrestronConsole.PrintLine(message);
}

void Trace(string message) {
	CrestronConsole.Print($"\x00FA\x00E0{(char)InitialParametersClass.ApplicationNumber}{message}\x00FB");
}
```