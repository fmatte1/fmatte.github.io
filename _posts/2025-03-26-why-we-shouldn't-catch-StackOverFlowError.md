---
title: "Why we shouldn't catch StackOverflowError?"
date: 2025-03-26 00:00:00 +0800
categories: 
tags: 
---

![Background img](assets/img/Leaves.png)


Photo by <a href="https://unsplash.com/@erol?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Erol Ahmed</a>
      

# What is StackOverFlowError?


The HotSpot Java Virtual Machine (JVM) allocates a small amount of memory specifically for each thread to run its call stack. However, if this allocated memory is exceeded, a StackOverflowError can be observed. This is a Java runtime error that usually ocurs due to excessive recursion, where method calls itself repeatedly without a proper terminating condition, or the method has an excessively deep call chain.

Here's an example of how you might encounter this error in Java
```


public class StackDemo {

	public static void main(String[] args) {
		recursiveMethod();
	}
	
	public static void recursiveMethod() {
		recursiveMethod();
	}
}


```


In this example, the `recursiveMethod` method calls itself recursively without any terminating condition. This means that if you call `recursiveMethod()`, it will keep calling itself over and over again until the maximum limit of the call stack is reached, resulting in a StackOverFlowError

![Call stack progression](assets/img/stackoverflow.png)

```

Exception in thread "main" java.lang.StackOverflowError
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
	at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
at experimental.StackDemo.recursiveMethod(StackDemo.java:10)
```

A `StackOverflowError` is a type of `VirtualMachineError`, which is an error. This indicates a serious problem that a reasonable application should not attempt to catch.

Language Specification and the <a href="https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.3">Java Virtual Machine Specification </a> provide detailed information about the semantics of this error. Specifically the latter states: 

``
A Java Virtual Machine implementation throws an object that is an instance of a subclass of the class
VirtualMethodError when an internal error or resource limitation prevents it from implementing the semantics
described in this chapter. This specification cannot predict where internal errors or resource limitations 
may be encountered and does not mandate precisely when they can be reported. Thus, any of the 
VirtualMethodError subclasses defined below may be thrown at any time during the operation of the 
Java Virtual Machine:
``

It defines StackOverflowError as:

``
StackOverflowError: The Java Virtual Machine implementation has run out of stack space for a thread, typically because the thread is doing an unbounded number of recursive invocations as a result of a fault in the executing program.
``

# What can go wrong in catching StackOverflowError?

Let’s now consider the following program, which is indeed a flawed example that first catches and then ignores the triggered `StackOverflowError.

```

import java.util.ArrayList;
import java.util.List;

public class Example {

	List list = new ArrayList();
	boolean flag = true;
	public static void main (String[] args) {
		Example ex = new Example();
		do {
			try {
				ex.recursive();
			} catch(StackOverflowError err) {
				// IGNORE
			}
			
		} while (!ex.list.isEmpty());

	}
	
	private void recursive() throws Error {
		int arr[] = new int[1024];
		if (flag) {
			list.add(arr);
			recursive();	
		} else {
			flag = false;
		}
		if (list.contains(arr)) {
			list.remove(arr);
		}	
	}
}

catch(StackOverflowError err) {
	// IGNORE 
}

```

You may have already noticed that the variable `flag` was never set to `false`, causing a recursive call that would never terminate. As a result, this will lead to a `StackOverflowError`. However, we have a problematic catch block in place that ignores this error.

In this program, all allocations are being done in a global variable called `list`. Every recursive call adds 1KB of data to the list by calling `list.add(arr)`. And, as we ignore the `StackOverflowError`, the program continues with the execution and allocations, and we end up with an `OutOfMemoryError`. This is how the output looks like

```

Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at Example.recursive(Example.java:22)
	at Example.recursive(Example.java:25)
	at Example.recursive(Example.java:25)
	at Example.recursive(Example.java:25)
	at Example.recursive(Example.java:25)
	at Example.recursive(Example.java:25)
	at Example.recursive(Example.java:25)
	at Example.recursive(Example.java:25)
	at Example.recursive(Example.java:25)
	at Example.recursive(Example.java:25)
```

In large scale applications with million lines of code, ignoring `StackOverflowError` even in a single location, can have severe consequences. Debugging such issues is very difficult and frustrating.

# How to identify StackOverFlowError from HotSpot log files
When a Java application crashes, the Java Virtual Machine (JVM) generates a HotSpot error log file named `hs_error<%pid>.log`. These log files contain valuable information, including the number of times that `StackOverflowError`s were encountered before the crash. This is one indication that a stack overflow had occurred. The above program when run with the JVM option `-XX:+CrashOnOutOfMemoryError`, produces the following information in the `hs_err` log file about the number of occurrences of `StackOverflowError`. 

```
OutOfMemory and StackOverflow Exception counts:
StackOverflowErrors=154
```

The above information can also be fetched by running `jcmd VM.info` command

This information is highly useful in determining whether the application had encountered any `StackOverflowError` during its execution and, if so, whether an unexpected crash was caused by that.

# Summary
Having StackOverflowError, there’s likely not enough stack to do anything about it. One should not catch `StackOverflowError` in any situation. `StackOverflowError` indicates a serious problem that an application should not attempt to catch.

In a few coming posts, I’ll further expand with more examples. Stay tuned.