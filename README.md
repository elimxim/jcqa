# Java Concurrency Interview Questions

This is a collection of questions that I heard during an interviews or found while reading.

---

**Table of content**

- [Motivation](#motivation)
- [Questions](#questions)
  - [1. The keyword `synchronized`](#q-1)
    - [1.1 Intrinsic lock reentrancy](#q-1-1)
    - [1.2 Synchronized method overriding](#q-1-2)
    - [1.3 Static fields synchronization](#q-1-3)
    - [1.4 Inner class lock](#q-1-4)
    - [1.5 Using `getClass()`](#q-1-5)
    - [1.6 Read-write locks](#q-1-6)
    - [1.7 Task &laquo;Print array&raquo;](#q-1-7)
    - [1.8 Task &laquo;Money transfer&raquo; <sup>&laquo;used&raquo;</sup>](#q-1-8)
    - [1.9 Task &laquo;Swap value&raquo;](#q-1-9)
  - [2. The keyword `volatile`](#q-2)
    - [2.1 `volatile` atomicity <sup>&laquo;used&raquo;</sup>](#q-2-1)
    - [2.2 `volatile` visibility <sup>&laquo;used&raquo;</sup>](#q-2-2)
    - [2.3 `volatile` object](#q-2-3)
  - [3. The class `java.lang.Thread` and synchronization primitives of the `Object` class](#q-3)
    - [3.1 Starting a thread <sup>&laquo;used&raquo;</sup>](#q-3-1)
    - [3.2 Running a thread <sup>&laquo;used&raquo;</sup>](#q-3-2)
    - [3.3 Reusing a thread](#q-3-3)
    - [3.4 Stopping a thread <sup>&laquo;used&raquo;</sup>](#q-3-4)
    - [3.5 Joining a thread](#q-3-5)
    - [3.6 Notifying a thread](#q-3-6)
    - [3.7 `Thread.sleep` vs `Object.wait` <sup>&laquo;used&raquo;</sup>](#q-3-7)
    - [3.8 `Object.notify` vs `Object.notifyAll` (missed signals) <sup>&laquo;used&raquo;</sup>](#q-3-8)
  - [4. The package `java.util.concurrent`](#q-4)
    - [4.1 Executor Service `shutdown` vs `shutdownNow` <sup>&laquo;used&raquo;</sup>](#q-4-1)
  - [5. Safe publication](#q-5)
    - [5.1 Immutable classes <sup>&laquo;used&raquo;</sup>](#q-5-1)
    - [5.2 Lazy loading <sup>&laquo;used&raquo;</sup>](#q-5-2)
    - [5.3 Double-checked locking <sup>&laquo;used&raquo;</sup>](#q-5-3)


## Motivation <a name="motivation"/>

After many interviews where I acted as an interviewer, I heard a lot of questions from my colleagues 
about concurrency in Java, which they asked interviewees. I found those interview questions and tasks fascinating.
And a few years ago I happened to see the origins of those interview questions and tasks while I was reading 
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

- `[JCP]: "quote"` - means the quote is from "Java Concurrency in Practice"
- `[CPJ]: "quote"` - means the quote is from "Concurrent Programming in Java ..."

## Questions <a name="questions"/>

There are questions of varying difficulty here. The questions that I heard or used during interviews
are marked as `a question <sup>&laquo;used&raquo;</sup>`. The answers follow the questions,
but they are hidden to avoid spoilers.

There are 5 main topics that contain a number of questions:

- the keyword `synchornized`
- the keyword `volatile`
- the class `java.lang.Thread`
- the package `java.util.concurrent`
- safe publication

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

#### 1.5 Using `getClass()` <a name="q-1-5"/>

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

#### 1.6 Read-write locks <a name="q-1-6"/>

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

#### 1.7 Task &laquo;Print array&raquo; <a name="q-1-7"/>

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

#### 1.8 Task &laquo;Money transfer&raquo; <sup>&laquo;used&raquo;</sup> <a name="q-1-8"/>

Q: *What problem might occur in the following code? How to fix it?*

```java
@NotTreadSafe
class Account {
    private BigDecimal balance;

    public void setInitialBalance(BigDecimal initial) {
        this.balance = initial;
    }
    
    public balance getBalance() {
        return balance;
    }
    
    public void debit(BigDecimal amount) {
        this.balance = balance.subtract(amount);
    }
    
    public void credit(BigDecimal amount) {
        this.balance = balance.add(amount);
    }
}

class Accountant {
    public void transferMoney(Account fromAccount, Account toAccount, BigDecimal amount) {
        synchronized (fromAccount) {
            synchronized (toAccount) {
                if (fromAccount.getBalance().compareTo(amount) < 0)
                    throw new RuntimeException("Insufficient funds");
                else {
                    fromAccount.debit(amount);
                    toAccount.credit(amount);
                }
            }
        }
    }
}
```

The above code is based on an example from [JCP].

<details>
    <summary>The Answer</summary>

A: *The above code is deadlock-prone. The solution is to avoid locking on method arguments.*

[JCP]:

> How can `transferMoney` deadlock? It may appear as if all the threads acquire their locks in the same order, 
> but in fact the lock order depends on the order of arguments passed to transferMoney, and these in turn might 
> depend on external inputs. Deadlock can occur if two threads call `transferMoney` at the same time, 
> one transferring from X to Y, and the other doing the opposite:
> 
> - A: `transferMoney(myAccount, yourAccount, 10);`
> - B: `transferMoney(yourAccount, myAccount, 20);`
> 
> With unlucky timing, A will acquire the lock on `myAccount` and wait for the lock on `yourAccount`, 
> while B is holding the lock on `yourAccount` and waiting for the lock on `myAccount`.

*For example using `ReentranLock`:*

```java
@NotTreadSafe
class Account {
    public final ReentrantLock lock = new ReentrantLock();
    
    private BigDecimal balance;

    public void setInitialBalance(BigDecimal initial) {
        this.balance = initial;
    }
    
    public BigDecimal getBalance() {
        return balance;
    }

    public void debit(BigDecimal amount) {
        this.balance = balance.subtract(amount);
    }

    public void credit(BigDecimal amount) {
        this.balance = balance.add(amount);
    }
}

@TreadSafe
class Accountant {
    public void transferMoney(Account fromAccount, Account toAccount, BigDecimal amount) {
        if (fromAccount == toAccount) { // alias check
            return;
        }
        
        if (fromAccount.lock.tryLock()) {
            try {
                if (toAccount.lock.tryLock()) {
                    try {
                        fromAccount.debit(amount);
                        toAccount.credit(amount);                       
                    } finally {
                        toAccount.lock.unlock();
                    }
                } else {
                    throw new RuntimeException("toAccount is currently busy");
                }               
            } finally {
                fromAccount.lock.unlock();
            }
        } else {
            throw new RuntimeException("fromAccount is currently busy");
        }
    }
}
```

</details>

#### 1.9 Task &laquo;Swap value&raquo; <a name="q-1-9"/>

Q: *What problem might occur in the following code? How to fix it?*

```java
@TreadSafe
class Cell {
    private long value;
    
    synchronized long getValue() { 
        return value; 
    }
    
    synchronized void setValue(long v) { 
        value = v; 
    }
    
    synchronized void swapValue(Cell other) {
        long t = getValue();
        long v = other.getValue();
        setValue(v);
        other.setValue(t);
    }
} 
```

The above code is based on an example from [CPJ].

<details>
    <summary>The Answer</summary>

A: *The above code is deadlock-prone. The solution is to avoid locking on method arguments.*

[CPJ]:

> SwapValue is a synchronized *multiparty* action — one that intrinsically acquires locks on multiple objects.
> Without further precautions, it is possible for two different threads, one running `a.swapValue(b)`,
> and the other running `b.swapValue(a)`, to deadlock when encountering the following trace:
>
> | Thread 1                                                            | Thread 2                                                            |
> |---------------------------------------------------------------------|---------------------------------------------------------------------|
> | acquire lock for `a` on entering `a.swapValue(b)`                   |                                                                     |
> | pass lock for `a` (since already held) on entering `t = getValue()` | acquire lock for `b` on entering `b.swapValue(a)`                   |
> | block waiting for lock for `b` on entering `v = other.getValue()`   | pass lock for `b` (since already held) on entering `t = getValue()` |
> |                                                                     | block waiting for lock for `a` on entering `v = other.getValue()`   |

*For example using `ReentranLock` and changing the method to return a `boolean` value:*

```java
@TreadSafe
class Cell {
    private final ReentrantLock lock = new ReentrantLock();
    
    private long value;
    
    long getValue() {
        lock.lock();
        try {
            return value;
        } finally {
            lock.unlock();
        }
    }
    
    void setValue(long v) {
        lock.lock();
        try {
            value = v;
        } finally {
            lock.unlock();
        }
    }
    
    synchronized boolean swapValue(Cell other) {
        if (other == this) {  // alias check
            return true;
        }
        
        if (lock.tryLock()) {
            long t = getValue();
            try {
                if (other.lock.tryLock()) {
                    try {
                        long v = other.getValue();
                        setValue(v);
                        other.setValue(t);
                        return true;
                    } finally {
                        other.lock.unlock();
                    }
                } else {
                    return false;
                }
            } finally {
                lock.unlock();
            }
        } else {
            return false;
        }
    }
} 
```

</details>

### 2. The keyword `volatile` <a name="q-2"/>

#### 2.1 `volatile` atomicity <sup>&laquo;used&raquo;</sup> <a name="q-2-1"/>

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

#### 2.2 `volatile` visibility <sup>&laquo;used&raquo;</sup> <a name="q-2-2"/>

Q: *What can `T2` print in "Example 1"? Will anything change if `b` becomes 
a `volatile` variable as in the "Example 2"?*

Example 1:

```
Given: 

int a = 0;
int b = 0;

Action:

| T1    | T2       |
| a = 1 | print(b) |
| b = 1 | print(a) |
```

Example 2:

```
Given: 

int a = 0;
volatile int b = 0;

Action:

| T1    | T2       |
| a = 1 | print(b) |
| b = 1 | print(a) |
```

<details>
    <summary>The Answer</summary>

A: *In "Example 1" there could be all the options because `T2` may see the changed `b` 
but not see the changed `a` or vice versa, or will not be able to see all the changes, 
or will be able to see all the changes. In "Example 2" only one option is available.*

*Example 1:*

| a | 0 | 0 | 1 | 1 |
|---|---|---|---|---|
| b | 0 | 1 | 0 | 1 |

*Example 2:*

| a | 1 |
|---|---|
| b | 1 |

[JCP]: 

> The JMM defines a partial ordering<sup>2</sup> called *happens-before* on all actions
> within the program. To guarantee that the thread executing action B can see the
> results of action A (whether or not A and B occur in different threads), there must
> be a *happens-before* relationship between A and B. In the absence of a *happens-before*
> ordering between two operations, the JVM is free to reorder them as it pleases
> 
> <sub>2. A partial ordering ≺ is a relation on a set that is antisymmetric, reflexive, and transitive, but for
> any two elements x and y, it need not be the case that x ≺ y or y ≺ x. We use partial orderings every
> day to express preferences; we may prefer sushi to cheeseburgers and Mozart to Mahler, but we don’t 
> necessarily have a clear preference between cheeseburgers and Mozart.</sub>

[JCP]:

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

[JCP]:

> Other *happens-before* orderings guaranteed by the class library include:
> - Placing an item in a thread-safe collection *happens-before* another thread retrieves that item from the collection;
> - Counting down on a `CountDownLatch` *happens-before* a thread returns from await on that latch;
> - Releasing a permit to a `Semaphore` *happens-before* acquiring a permit from that same `Semaphore`;
> - Actions taken by the task represented by a `Future` *happens-before* 
> another thread successfully returns from `Future.get`;
> - Submitting a `Runnable` or `Callable` to an `Executor` *happens-before* the task begins execution; and
> - A thread arriving at a `CyclicBarrier` or `Exchanger` *happens-before* the other threads are released 
> from that same barrier or exchange point. If `CyclicBarrier` uses a barrier action, 
> arriving at the barrier *happens-before* the barrier action, which in turn *happens-before* 
> threads are released from the barrier.

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

#### 3.1 Starting a thread <sup>&laquo;used&raquo;</sup> <a name="q-3-1"/>

Q: *How to start a new thread?*

<details>
    <summary>The Answers</summary>

A: *Create an instance of `Thread` class and call the `start` method.*

</details>

#### 3.2 Running a thread <sup>&laquo;used&raquo;</sup> <a name="q-3-2"/>

Q: *How many times “Hello!” will it be printed?*

```java
class Example {
    public static void main(String[] args) {
        Thread t = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                System.out.print("Hello!");
            }
        });

        t.interrupt();
        t.run();
    }
}
```

<details>
    <summary>The Answer</summary>

A: *"Hello" will be printed infinitely until the program is stopped because `Thread.run` doesn't start the thread.*

</details>>

#### 3.3 Reusing a thread <a name="q-3-3"/>

Q: *How many times “Hello!” will it be printed?*

```java
class Example {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            System.out.println("Hello!");
        });
        
        t.start();
        t.join();
        
        t.start();
        t.join();
    }
}
```

<details>
    <summary>The Answer</summary>

A: *"Hello!" will be printed once and an exception will be thrown because a thread cannot be started twice.*

</details>

#### 3.4 Stopping a thread <sup>&laquo;used&raquo;</sup> <a name="q-3-4"/>

Q: *How to stop a thread?*

<details>
    <summary>The Answer</summary>

A: *There is no safe way to stop a thread. But to stop a thread, a `volatile` flag or
interruption mechanism can be used.*

[JCP]:

> There is no safe way to preemptively stop a thread in Java, and therefore no safe way
> to preemptively stop a task. There are only cooperative mechanisms, by which the task
> and the code requesting cancellation follow an agreed-upon protocol.

</details>

#### 3.5 Joining a thread <a name="q-3-5"/>

Q: *What might the following code print? What can be printed if the `calculated` field is `volatile`?*

```java
class Example {
    private boolean calculated;
    
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            calculated = true;
        });
        
        Thread t2 = new Thread(() -> {
            if (calculated) {
                System.out.println("42");
            } else {
                System.out.println("calculating...");
            }
        });

        System.out.println(
                "The Answer to the Ultimate Question of Life, " +
                "the Universe, and Everything:"
        );
        
        t1.start();
        t1.join();
        t2.start();
    }
}
```

<details>
    <summary>The Answer</summary>

A: *Only "42" will be printed because any actions in a thread happens-before any other actions 
after `Thread.join`,regardless of whether the `calculated` field is `volatile` or not.*

[JCP]:

> The rules for happens-before are:
> - **Thread termination rule**. Any action in a thread *happens-before* any other thread detects
> that thread has terminated, either by successfully return from `Thread.join` or by 
> `Thread.isAlive` returning false.

</details>

#### 3.6 Notifying a thread <a name="q-3-6"/>

Q: *What will be printed?*

```java
class Example {
    private final Object monitor = new Object();
    
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Tread(() -> {
            System.out.println("Step 1");
            try {
                monitor.wait();
                System.out.println("Step 2");
            } catch (InterruptedException ignore) { }
        });
        
        Thread t2 = new Thread(() -> {
            monitor.notifyAll();
            System.out.println("Step 3");
        });
        
        t1.start();
        
        Thread.sleep(1000);
        
        t2.start();
    }
}
```

<details>
    <summary>The Answer</summary>

A: *"Step 1" will be printed and an exception will be thrown after it, 
because in order to call the `wait()` and `notifyAll()` methods on an object, 
a lock on that object must be acquired.*

```java
class Example {
    private final Object monitor = new Object();
    
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Tread(() -> {
            System.out.println("Step 1");
            try {
                synchronized(monitor) {
                    monitor.wait();
                }
                System.out.println("Step 2");
            } catch (InterruptedException ignore) { }
        });
        
        Thread t2 = new Thread(() -> {
            synchronized (monitor) {
                monitor.notifyAll();
                System.out.println("Step 3");
            }
        });
        
        t1.start();
        
        Thread.sleep(1000);
        
        t2.start();
    }
}
```

</details>

#### 3.7 `Thread.sleep` vs `Object.wait` <sup>&laquo;used&raquo;</sup> <a name="q-3-7"/>

Q: *What's the difference between `Thread.sleep` and `Object.wait`?*

<details>
    <summary>The Answer</summary>

From javadoc:

> Causes the currently executing thread to sleep (temporarily cease execution) 
> for the specified number of milliseconds, subject to the precision and accuracy of system 
> timers and schedulers. The thread does not lose ownership of any monitors.

[JCP]:

> Object.wait atomically releases the lock and asks the OS to suspend the current thread, 
> allowing other threads to acquire the lock and therefore modify the object state. 
> Upon waking, it reacquires the lock before returning. Intuitively, calling wait means 
> “I want to go to sleep, but wake me when something interesting happens”, and calling the 
> notification methods means “something interesting happened”.

[CPJ]:

> **Wait**. A `wait` invocation results in the following actions:
> - If the current thread has been interrupted, then the method exits immediately, throwing an
> `InterruptedException`. Otherwise, the current thread is blocked.
> - The JVM places the thread in the internal and otherwise inaccessible wait set associated with
> the target object.
> - The synchronization lock for the target object is released, but all other locks held by the
> thread are retained. A full release is obtained even if the lock is re-entrantly held due to nested
> synchronized calls on the target object. Upon later resumption, the lock status is fully
> restored.

</details>

#### 3.8 `Object.notify` vs `Object.notifyAll` (missed signals) <sup>&laquo;used&raquo;</sup> <a name="q-3-8"/>

Q: *What's the difference between `Object.notify` and `Object.motifyAll`? 
Are there any risks when using `Object.notify`?*

<details>
    <summary>The Answer</summary>

[JCP]:

> There are two notification methods in the condition queue API - `notify` and
> `notifyAll`. To call either, you must hold the lock associated with the condition
> queue object. Calling `notify` causes the JVM to select one thread waiting on that
> condition queue to wake up; calling `notifyAll` wakes up all the threads waiting
> on that condition queue. Because you must hold the lock on the condition queue
> object when calling notify or `notifyAll`, and waiting threads cannot return from
> wait without reacquiring the lock, the notifying thread should release the lock
> quickly to ensure that the waiting threads are unblocked as soon as possible.
> 
> Because multiple threads could be waiting on the same condition queue for
> different condition predicates, using `notify` instead of `notifyAll` can be dangerous, 
> primarily because single notification is prone to a problem akin to missed signals.
 
[JCP]:

> A missed signal occurs when a thread must wait for a specific condition that is already true, 
> but fails to check the condition predicate before waiting. Now the thread is waiting to be notified of an event
> that has already occurred. This is like starting the toast, going out to get the newspaper, 
> having the bell go off while you are outside, and then sitting down at the
> kitchen table waiting for the toast bell. You could wait a long time - potentially
> forever. Unlike the marmalade for your toast, notification is not “sticky” - if
> thread A notifies on a condition queue and thread B subsequently waits on that
> same condition queue, B does not immediately wake up - another notification
> is required to wake B. Missed signals are the result of coding errors like those
> warned against in the list above, such as failing to test the condition predicate
> before calling wait.

[CPJ]:

> **Notify**. A `notify` invocation results in the following actions:
> - If one exists, an arbitrarily chosen thread, say *T*, is removed by the JVM from the internal
> wait set associated with the target object. There is no guarantee about which waiting thread
> will be selected when the wait set contains more than one thread.
> - *T* must re-obtain the synchronization lock for the target object, which will *always* cause it to
> block at least until the thread calling `notify` releases the lock. It will continue to block if
> some other thread obtains the lock first.
> - *T* is then resumed from the point of its `wait`.
> 
> **NotifyAll**. A `notifyAll` works in the same way as `notify` except that the steps occur (in effect,
> simultaneously) for *all* threads in the wait set for the object. However, because they must acquire the
> lock, threads continue one at a time.

[CPJ]:

> Additionally, a liveness failure could result if `setCount` were written in 
> a non-atomic fashion, in particular as:
> ```java
> void badSetCount(long newValue) { // Do not use
>     synchronized(this) { notifyAll(); }
>     // (**)
>     synchronized(this) { count = newValue; }
> }
> ```
> Here, the method first acquires the lock to perform `notifyAll`, then releases it, 
> and then reacquires it to change `count`. This could result in a *missed signal*: 
> A thread executing at point `(**)` might start waiting *after* the signal intended 
> to wake it up was issued but before the condition was changed. This thread will 
> wait forever, or at least until the next notification is somehow produced.
> 
> Note that within `synchronized` methods, the order in which a `notifyAll` is placed does not 
> matter. No awakened threads will be able to continue until the synchronization lock is released. 
> Just as a matter of style, most people put notifications last in method bodies.

</details>

### 4. The package `java.util.concurrent` <a name="q-4"/>

#### 4.1 Executor Service `shutdown` vs `shutdownNow` <sup>&laquo;used&raquo;</sup> <a name="q-4-1"/>

Q: *What is the difference between `ExecutorService.shutdown` and `ExecutorService.shutdownNow`?*

<details>
    <summary>The Answer</summary>

From javadoc:

> The `shutdown()` method will allow previously submitted tasks to execute before terminating, 
> while the `shutdownNow()` method prevents waiting tasks from starting 
> and attempts to stop currently executing tasks. 
> Upon termination, an executor has no tasks actively executing, 
> no tasks awaiting execution, and no new tasks can be submitted.

[JCP]:

> The `shutdown` method initiates a graceful shutdown: no new tasks are accepted
> but previously submitted tasks are allowed to complete—including those that
> have not yet begun execution. The `shutdownNow` method initiates an abrupt shutdown: 
> it attempts to cancel outstanding tasks and does not start any tasks that
> are queued but not begun.

</details>

### 5. Safe publication <a name="q-5"/>

#### 5.1 Immutable classes <sup>&laquo;used&raquo;</sup> <a name="q-5-1"/>

Q1: *Does the `ElectricVehicle` class is immutable?
What are the advantages of immutable objects in concurrent environment?*

```java
class Battery {
    private int power;
    private int capacity;
    
    public Battery(int initialPower, int initialCapacity) {
        this.power = calculatePower(initialPower);
        this.capacity = calculateCapacity(initialCapacity);
    }
    
    // getters & setters
}

class ElectricVehicle {
    private final String name;
    private final Battery battery;
    
    public Vehicle(String name, Battery battery) {
        this.name = name;
        this.battery = battery;
    }
    
    // getters
}
```

<details>
    <summary>The Answer</summary>

A: *The `ElectricVehicle` class isn't thread-safe because it contains the not tread-safe `Battery` class.
Immutable objects are thread-safe and can be used by multiple threads.*

[JCP]:

> An immutable object is one whose state cannot be changed after construction.
> Immutable objects are inherently thread-safe; their invariants are established by
> the constructor, and if their state cannot be changed, these invariants always hold.
> - Immutable objects are always thread-safe.

[JCP]:

> An object is immutable if:
> - Its state cannot be modified after construction;
> - All its fields are `final`;
> - It is *properly constructed* (the `this` reference does not escape during construction).

[JCP]: 

> *Immutable* objects can be used safely by any thread without additional
> synchronization, even when synchronization is not used to publish them

</details>

#### 5.2 Lazy loading <sup>&laquo;used&raquo;</sup> <a name="q-5-2"/>

Q: *How to make the following lazy-load class thread-safe? The class is too large to preload.*

```java
@NotTreadSafe
public class UnsafeLazyInitialization {
    private static Resource resource;
    
    public static Resource getInstance() {
        if (resource == null) {
            resource = new Resource(); // unsafe publication
        }
        return resource;
    }
}
```

*This is not compliant:*

```java
@TreadSafe
public class EagerInitialization {
    private static final Resource resource = new Resource();
    
    public static Resource getInstance() {
        return resource;
    }
}
```

The above code is from [JCP].

<details>
    <summary>The Answer</summary>

A: *There are several options to make the lazy-load `Singleton` class thread-safe:*

*Example 1:*

```java
@TreadSafe
public class SafeLazyInitialization {
    private static Resource instance;
    
    public synchronized static Resource getInstance() {
        if (resource == null) {
            resource = new Resource();
        }
        return resource;
    }
}
```

The above code is from [JCP].

[JCP]:

> Because the code path through `getInstance` is fairly short (a test and a predicted branch), 
> if `getInstance` is not called frequently by many threads, there is little enough contention 
> for the class lock that this approach offers adequate performance

*Example 2:*

```java
@TreadSafe
public class ResourceFactory {
    private static class ResourceHolder {
        public static final Resource resource = new Resource();
    } 
    
    public static Resource getInstance() {
        return ResourceHolder.resource;
    }
}
```

The above code is from [JCP].

[JCP]:

> The JVM defers initializing the ResourceHolder class until it is actually used [JLS 12.4.1], and because the
> Resource is initialized with a static initializer, no additional synchronization is needed. 
> The first call to getResource by any thread causes ResourceHolder to be loaded and initialized, 
> at which time the initialization of the Resource happens through the static initializer.

</details>

#### 5.3 Double-checked locking <sup>&laquo;used&raquo;</sup> <a name="q-5-3"/>

Q: *Is the following class tread-safe?*

```java
public class DoubleCheckedLocking {
    private static Resource resource;
    
    public static Resource getInstance() {
        if (resource == null) {
            synchronized (DoubleCheckedLocking.class) {
                if (resource == null) {
                    resource = new Resource();
                }
            }
        }
        return resource;
    }
}
```

<details>
    <summary>The Answer</summary>

A: *The code above is dangerous, or at least was dangerous in earlier versions of the JDK.
The answer needs to be double-checked.*

[JCP]:

> No book on concurrency would be complete without a discussion of the infamous double-checked 
> locking (DCL) antipattern ... .  
> The real problem with DCL is the assumption that the worst thing that can happen when reading 
> a shared object reference without synchronization is to erroneously see a stale value 
> (in this case, null); in that case the DCL idiom compensates for this risk by trying again 
> with the lock held. But the worst case is actually considerably worse - it is possible to see 
> a current value of the reference but stale values for the object’s state, meaning that 
> the object could be seen to be in an invalid or incorrect state.
 
</details>