# Java Concurrency Interview Questions

This is a collection of questions that, in my opinion, are the most interesting to ask during an interview.
This is not a complete collection. I hope new interesting ideas for questions will be added in the future.

---

**Table of content**

- [Motivation](#motivation)
- [Synopsis](#synopsis)
- [Questions](#questions)
  - [1. The keyword `synchronized`](#q-1)
    - [1.1 Intrinsic lock reentrancy](#q-1-1)
    - [1.2 Synchronized method overriding](#q-1-2)
    - [1.3 Static fields synchronization](#q-1-3)
    - [1.4 Inner class lock](#q-1-4)
    - [1.5 Poor synchronization <sup>&laquo;tricky&raquo;</sup>](#q-1-5)
    - [1.6 Different locks <sup>&laquo;tricky&raquo;</sup>](#q-1-6)
    - [1.7 The &laquo;Print array&raquo; task](#q-1-7)
    - [1.8 The &laquo;Money transfer&raquo; deadlock]
    - [1.9 The &laquo;Swap value&raquo; deadlock]
  - [2. The keyword `volatile`](#q-2)
    - [2.1 `volatile` atomicity](#q-2-1)
    - [2.2 `volatile` visibility](#q-2-2)
    - [2.3 `volatile` object](#q-2-3)
  - [3. The class `java.lang.Thread` and synchronization primitives of the `Object` class](#q-3)
    - [3.1 Starting a thread]
    - [3.2 Starting a thread twice]
    - [3.3 Stopping a thread]
    - [3.4 Joining a thread]
    - [3.5 Another joining a thread <sup>&laquo;tricky&raquo;</sup>]
    - [3.6 Waiting without obtaining a lock <sup>&laquo;tricky&raquo;</sup>]
    - [3.7 `Thread.sleep` vs `Object.wait`]
    - [3.8 Thread interruption]
    - [3.9 `Object.notify` vs `Object.notifyAll`. Missed Signals]
    - [3.10 Options of `Object.notifyAll`]
    - [3.11 Spurious wakeups]
  - [4. The package `java.util.concurrent`](#q-4)
    - [4.1 Thread Pool shutdown]
  - [5. Thread safety](#q-5)
    - [5.1 Immutable classes]
    - [5.2 Singleton implementation]


## Motivation <a name="motivation"/>

After many interviews where I acted as an interviewer, I heard a lot of questions from my colleagues 
about concurrency in Java, which they asked interviewees. I found these interview questions and tasks fascinating.
And a few years ago I happened to see the origins of these interview questions and tasks while I was reading 
two ancient books about concurrency in Java.

There are those two books:

| Name                                                                          | Author      | Year of publication |
|-------------------------------------------------------------------------------|-------------|---------------------|
| Java Concurrency in Practice                                                  | Brian Goetz | 2006                |
| Concurrent Programming in Java Second Edition: Design Principles and Patterns | Doug Lea    | 1999                |

After reading these books a while later, I thought it would be great to accumulate 
all the interview questions in one place.

There are many quotes from these books here. To easily identify which quote is from which book, 
there are two main footnotes:

- `"quote" <sup>[JCP]</sup>` or `[JCP]: "quote"` - means the quote is from "Java Concurrency in Practice"
- `"quote" <sup>[CPJ]</sup>` or `[CPJ]: "quote"` - means the quote is from "Concurrent Programming in Java ..."

## Synopsis <a name="synopsis"/>

Some questions are too tricky to ask, but they have also been included here for fun. 
These questions will be marked as <sup>&laquo;tricky&raquo;</sup>. To simplify the questions,
some of them contain names of methods or classes that relate to the code below. 
Sometimes questions are accompanied by examples, then a question has a link to the code.

The answers follow the questions, but they are hidden to avoid spoilers.

## Questions <a name="questions"/>

There are 5 main topics that contain a number of questions:

- the keyword `synchornized`
- the keyword `volatile`
- the class `java.lang.Thread`
- the package `java.util.concurrent`
- thread safety

### 1. The keyword `synchronized` <a name="q-1"/>

#### 1.1 Intrinsic lock reentrancy <a name="q-1-1"/>

Q: *What happens if the synchronized `pressEngineButton` method calls the `startEngine` method which is also synchronized?*

```java
class Vehicle {
    public synchronized void startEngine() {
        // ...
    }
}

class ModernCar extends Vehicle {
    public synchronized void pressEngineButton() {
        super.startEngine();
    }
}
```

<details>
    <summary>The Answer</summary>

This is kind of a dumb question, but I find it interesting because lock reentry is not a default feature.

A: *Nothing bad will happen. The `startEngine` method will be invoked without problems.*

Intrinsic locking in Java, defined by the `synchronized` keyword, is reentrant. Otherwise, 
in the example above would cause a deadlock when the `pressEngineButton` calls the `startEngine`.

[JCP]:

> When a thread requests a lock that is already help by another thread, the requesting thread blocks.
> But because lock are *reentrant*, if a thread tries to acquire a lock that *it* already holds, 
> the requests succeeds. Reentrancy means that locks are acquired on a per-thread rather
> the per-invocation<sup>[7]</sup>. Reentrancy is implemented by associating with each lock 
> an acquisition count and an owning thread. When the count is zero, the lock is considered unheld.
> When a thread acquires a previously unheld lock, the JVM records the owner and sets the acquisition to one.
> If the same thread acquires the lock again, the count is incremented, and when the owning thread exits, 
> the `synchronized` block, the count is decremented. When the count reaches zero, the lock is released.
> 
> <sub>7. This differs from the default locking behaviour for pthreads (POSIX threads) mutexes, 
> which are granted on a per-invocation basis.</sub>

</details>

#### 1.2 Synchronized method overriding <a name="q-1-2"/>

Q: *What happens if the `move` method of the `Car` class is called by multiple threads at the same time?*

```java
class Vehicle {
    public synchronized void move() {
        // ...
    }
}

class Car extends Vehicle {
    @Override
    public void move() {
        // ...
    }
}
```

<details>
    <summary>The Answer</summary>

A: *Since `synchronized` modifier isn't inherited when subclasses override superclass methods, 
the `Car.move` method is not thread-safe.*

[CPJ]:

> The `synchronized` keyword is not considered to be part of a method's signature. So the `synchronized`
> modifier is not automatically inherited when subclasses override superclass methods, and methods 
> in `interfaces` cannot be declared as `synchronized`. Also, constructors cannot be qualified 
> as `synchronized` (although block synchronization can be used within constructors).

</details>

#### 1.3 Static fields synchronization <a name="q-1-3"/>

Q: *Is the `CountHolder` class thread-safe?*

```java
class CountHolder {
    private static int count = 0;
    
    public synchronized int getCount() {
        return count; 
    }
    
    public synchronized void setCount(int c) {
        count = c;
    }
}
```

<details>
    <summary>The Answer</summary>

A: *The `CountHolder` class isn't thread-sage, because object locking doesn't protect `static` fields.*

[CPJ]:

> Locking an object does *not* automatically protect access to `static` fields of that object's class
> or any of its superclasses. Access to `static` fields is instead protected via `synchronized static` methods
> and blocks. Static synchronization employs the lock processed by the `Class` object associated with the
> class the static methods are declared in.

</details>

#### 1.4 Inner class lock <a name="q-1-4"/>

Q: *Can an inner class use a lock of its outer class?*

<details>
    <summary>The Answer</summary>

A: *An inner class has independent locking of its outer class, but a non-static inner class can use 
a lock of its outer class with: `synchronized(OuterClass.this)`.*

[CPJ]:

> Synchronized instance methods in subclasses employ the same lock as those in their superclasses.
> But synchronization in an *inner* class method is independent of its outer class. However, a non-static
> inner class method can lock its containing class say `OuterClass`, via code blocks using:  
> `synchronized(OuterClass.this) { /* body */ }`

</details>

#### 1.5 Poor synchronization <sup>&laquo;tricky&raquo;</sup> <a name="q-1-5"/>

Q: *What is the danger of synchronization in the following code and is it wrong in the example below?*

```java
class Vehicle {
    private static int systemStatus;

    public void checkSystem() {
        synchronized (getClass()) {
            // ...
            systemStatus = 1; // OK
        }
    }
}

class Car extends Vehicle {
    private static int systemStatus;

    @Override
    public void checkSystem() {
        super.checkSystem();
        
        synchronized (getClass()) {
            // ...
            systemStatus = 1; // OK
        }
    }
}
```

<details>
    <summary>The Answer</summary>

A: *Using `synchronized(getClass())` increases complexity of the code. The above example is correct
because both `getClass()` expressions return corresponding `Class` instances.*

[CPJ]:

> `synchronized(getClass()) { /* body */ } // Do not use`  
> This locks the actual class, which might be different from (a subclass of) the class defining the
> `static` fields that need protecting.

</details>

#### 1.6 Multiple locks <sup>&laquo;tricky&raquo;</sup> <a name="q-1-6"/>

Q: "Is the `Vehicle` class thread-safe?"

```java
public class Vehicle {
    private final Object readLock = new Object();
    private final Object writeLock = new Object();

    private String name;
    
    public String getName() {
        synchronized (readLock) {
            return this.name;
        }
    }
    
    public void setName(String s) {
        synchronized (writeLock) {
            this.name = s;
        }
    }
}
```

<details>
    <summary>The Answer</summary>

A: *The `Vehicle` class isn't thread-safe, because different locks are used to read and write the `name` field.*

</details>

#### 1.7 The &laquo;Print array&raquo; task <a name="q-1-7"/>

Q: *The implementation of the `print` method isn't consistent: the method might print the same element 
twice if another thread rearranges elements. How to fix this method?*

```java
/**
 * Supports rearrange elements
 */
class RearrangeableArray<Integer> {
    // contains only unique elements
    private final List<Integer> array = new ArrayList<>();

    /**
     * Prints all elements without duplicates
     */
    public void print() {
        for (int i = 0; true; i++) {
            Integer item = null;
            synchronized(array) {
                if (i < array.size()) {
                  item = array.get(i);
                } else {
                    break;
                }
            }
            System.out.println(item);
        }
    }
}
```

<details>
    <summary>The Answer</summary>

A: *There are several ways to fix it: use synchronization before the print loop, 
which may significantly increase the waiting time of other threads, or make a copy of the array.*

```java
class RearrangeableArray<Integer> {
    private final List<Integer> array = new ArrayList<>();
    
    public void print() {
        synchronized (array) {
            for (int i = 0; i < array.size(); i++) {
              System.out.println(array.get(i));
            }
        }
    }
}
```

```java
class RearrangeableArray<Integer> {
    private final List<Integer> array = new ArrayList<>();
    
    public void print() {
        List<Integer> snapshot = null;
        synchronized (array) {
            snapshot = new ArrayList<>(array);
        }
        
        for (int i = 0; i < snapshot.size(); i++) {
            System.out.println(array.get(i));
        }
    }
}
```

</details>

#### 1.8 <a name="q-1-8"/>


#### 1.9 <a name="q-1-9"/>


### 2. The keyword `volatile` <a name="q-2"/>

#### 2.1 `volatile` atomicity <a name="q-2-1"/>

Q: **Is the `Counter` class thread-safe?

```java
class Counter {
    private volatile int count;
    
    public increment() {
        count++;
    }
}
```

<details>
    <summary>The Answer</summary>

A: *Since the `++` operation isn't atomic, the `Counter` class isn't thread-safe because volatile variables
can only guarantee visibility, not atomicity.*

[JCP]:

> Locking can guarantee both visibility and atomicity; volatile variables can only guarantee visibility.
> You can use volatile variables only when all the following criteria are met:
> - Writes to the variable do not depend on its current value, or you can ensure 
> that only a single thread ever updates the value;
> - The variable does not participate in invariants with other state variables; and
> - Locking is not required for any other reason while the variable is being accessed.

[JCP]:

> Volatile variables are a lighter-weight synchronization mechanism than locking 
> because they do not involve context switches or thread scheduling. However, 
> volatile variables have some limitations compared to locking: while they provide 
> similar visibility guarantees, they cannot be used to construct atomic compound actions. 
> This means that volatile variables cannot be used when one variable depends on another, 
> or when the new value of a variable depends on its old value. This limits when volatile 
> variables are appropriate, since they cannot be used to reliably implement common 
> tools such as counters or mutexes.  
> For example, while the increment operation (++i) may look like an atomic operation, 
> it is actually three distinct operations - fetch the current value of the variable, 
> add one to it, and then write the updated value back. In order to not lose an update, 
> the entire read-modify-write operation must be atomic

[CPJ]:

> Declaring a field as `volatile` differs only in that no locking is involved. In particular, composite
> read/write operations such as the "++" operation on `volatile` variables are not performed atomically.

</details>

#### 2.2 `volatile` visibility <a name="q-2-2"/>

Q: *What can `T2` print in "Example 1"? Will anything change if `b` becomes 
a `volatile` variable as in the "Example 2"?*

Example 1:

```
Given: 

int a = 0;
int b = 0;

| T1    | T2       |
| a = 1 | print(b) |
| b = 1 | print(a) |
```

Example 2:

```
Given: 

int a = 0;
volatile int b = 0;

| T1    | T2       |
| a = 1 | print(b) |
| b = 1 | print(a) |
```

<details>
    <summary>The Answer</summary>

A: *In "Example 1" there could be all the options because `T2` may see the changed `b` 
but not see the changed `a` or vice versa, or will not be able to see all the changes, 
or will be able to see all the changes. In "Example 2" only one option is available.*

Example 1:

| a | 0 | 0 | 1 | 1 |
|---|---|---|---|---|
| b | 0 | 1 | 0 | 1 |

Example 2:

| a | 1 |
|---|---|
| b | 1 |

[CPJ]: 

> The JMM defines a partial ordering2 called *happens-before* on all actions
> within the program. To guarantee that the thread executing action B can see the
> results of action A (whether or not A and B occur in different threads), there must
> be a *happens-before* relationship between A and B. In the absence of a *happens-before*
> ordering between two operations, the JVM is free to reorder them as it pleases
> 
> <sub>2. A partial ordering ≺ is a relation on a set that is antisymmetric, reflexive, and transitive, but for
> any two elements x and y, it need not be the case that x ≺ y or y ≺ x. We use partial orderings every
> day to express preferences; we may prefer sushi to cheeseburgers and Mozart to Mahler, but we don’t 
> necessarily have a clear preference between cheeseburgers and Mozart.</sub>

[CPJ]:

> The rules for *happens-before* are:
> - **Program order rule**. Each action in a thread *happens-before* every action in that thread 
> that comes later in the program order.
> - **Monitor lock rule**. An unlock on a monitor lock *happens-before* every
> subsequent lock on that same monitor lock.<sup>3</sup>
> - **Volatile variable rule**. A write to a volatile field *happens-before* every
> subsequent read of that same field.<sup>4</sup>
> - **Thread start rule**. A call to `Thread.start` on a thread *happens-before*
> every action in the started thread.
> - **Thread termination rule**. Any action in a thread *happens-before* any
> other thread detects that thread has terminated, either by successfully return 
> from `Thread.join` or by `Thread.isAlive` returning false.
> - **Interruption rule**. A thread calling interrupt on another thread
> *happens-before* the interrupted thread detects the interrupt (either by having 
> InterruptedException thrown, or invoking isInterrupted or interrupted).
> - **Finalizer rule**. The end of a constructor for an object *happens-before*
> the start of the finalizer for that object.
> - **Transitivity**. If A *happens-before* B, and B *happens-before* C, then A *happens-before* C.
> 
> <sub>3. Locks and unlocks on explicit `Lock` objects have the same memory semantics as intrinsic locks.</sub>  
> <sub>4. Reads and writes of atomic variables have the same memory semantics as volatile variables.</sub>

</details>

#### 2.3 `volatile` object <a name="q-2-3"/>

Q: *Is the `Vehicle` class thread-safe?*

```java
class ExtraInfo {
    String code;
}

class Vehicle {
    volatile ExtraInfo info;

    public setCode(String s) {
        info.code = s;
    }
}
```

<details>
    <summary>The Answer</summary>

A: *The `volatile` keyword doesn't guarantee "happens-before" inside objects.*

</details>

### 3. The class `java.lang.Thread` and synchronization primitives of the `Object` class <a name="q-3"/>

### 4. The package `java.util.concurrent` <a name="q-4"/>

### 5. Thread safety <a name="q-5"/>