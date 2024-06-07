# Raising events in S# and handling them in S+

This is a template/example for raising an event in S# and handling it in S+.

## S#
```csharp
// Example event arguments class
public class MyEventArgs : EventArgs {
    // A value for this example object
    public int Value { get; set; }

    // NEED DEFAULT CONSTRUCTOR
    public MyEventArgs() { } 
    
    // Constructor for passing an initial value
    public MyEventArgs(int value) {
        Value = value;
    }
} 

// C# library used in S+
public class MyClass {
    public delegate void MyEventHandler(object sender, MyEventArgs e);
    public event MyEventHandler MyEvent;
    
    public void sendEvent(int value) {
        // Make sure there's an event handler registered
        if (MyEvent != null) {
            // Raise the event in SIMPL+ with the given value
            MyEvent(this, new MyEventArgs(value));
        }
    }
}
```

## S+
```c++
MyClass classInstance; // Instance of the S# class

// Event handler for C# event - this is the code that will run when the event is raised
EVENTHANDLER classInstanceMyEvent(MyClass sender, MyEventArgs e) {
    TRACE("Value: %i", e.Value);
}

FUNCTION Main() {
    // Register our event handler for this class instance
    REGISTEREVENT(classInstance, MyEvent, classInstanceMyEvent);
}
```