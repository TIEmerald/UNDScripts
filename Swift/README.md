# Swift
## Experiments
### It's said access from a property of an object is much quicker than access from a dictionary
```swift
func evaluate(problemBlock: () -> Void, repeatTime: Int = 1)
{
    let start = DispatchTime.now() // <<<<<<<<<< Start time
    for _ in 1...repeatTime {
        problemBlock()
    }
    let end = DispatchTime.now()   // <<<<<<<<<<   end time

    let nanoTime = end.uptimeNanoseconds - start.uptimeNanoseconds // <<<<< Difference in nano seconds (UInt64)
    let timeInterval = Double(nanoTime) / 1_000_000_000 // Technically could overflow for long running tests

    print("Time: \(timeInterval) seconds")
}

struct TestStruct {
    var a: Int = 0
    var b: Int = 0
}

struct TestItem {
    let structure = TestStruct()
    let dictionary = ["a":0, "b":0]
}

let testItem = TestItem()

evaluate(problemBlock: {
    testItem.structure.a
}, repeatTime: 100000)
evaluate(problemBlock: {
    testItem.dictionary["a"]
}, repeatTime: 100000)

// Print out: 
// Time: 3.689489776 seconds <<< for property
// Time: 5.629894115 seconds <<< for dictionary
```
