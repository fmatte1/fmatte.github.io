---
title: "Why we should not ignore StackOverFlowErrors"
date: 2025-03-26 00:00:00 +0800
categories: 
tags: 
---

# What is StackOverflow Errors?


Java Virtual Machine (JVM) allocates a small amount of memory for every thread to run its call stack. StackoverflowError can be seen, when it reaches the maximum limit of the call stack. This is a java runtime error usually happens due to excessive recursion, where method calls itself without proper terminating condition or the method may have deep call chain.

Take a simple example
```


public class StackDemo {

	public static void main(String[] args) {
		call();
	}
	
	public static void call() {
		call();
	}
}


```


If you see the example above, the main() method is calling call(), and call() method internally calls itself recursively without any terminating condition.
The call will be executed on call stack, until it occupies entire call stack and gets into StackOverFlowError situation.

![Call stack progression](assets/img/stackoverflow.png)

```

Exception in thread "main" java.lang.StackOverflowError
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
	at experimental.StackDemo.call(StackDemo.java:10)
at experimental.StackDemo.call(StackDemo.java:10)
```

A StackOverflowError IS-A VirtualMachineError, which IS-AN Error. This indicates a serious problems that a reasonable application should not try to catch.

Language Specification and the <a href="https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.3">Java Virtual Machine Specification </a> provide information about the semantics of this exception. In particular, the latter says 

``
A Java Virtual Machine implementation throws an object that is an instance of a subclass of the class
VirtualMethodError when an internal error or resource limitation prevents it from implementing the semantics
described in this chapter. This specification cannot predict where internal errors or resource limitations 
may be encountered and does not mandate precisely when they can be reported. Thus, any of the 
VirtualMethodError subclasses defined below may be thrown at any time during the operation of the 
Java Virtual Machine:
``

It describes about the StackOverflowError

``
StackOverflowError: The Java Virtual Machine implementation has run out of stack space for a thread, typically because the thread is doing an unbounded number of recursive invocations as a result of a fault in the executing program.
``

# What can go wrong in catching StackOverFlowError?

Let us start with simple program, infact a buggy program and catching StackOverFlowError

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
	
	private int recursive() throws Error {
		int arr[] = new int[1024];
		if (flag) {
			list.add(arr);
			recursive();	
		} else {
			flag = false;
			return 0;
		}
		if (list.contains(arr)) {
			list.remove(arr);
		}	
		return 1;
	}
}
```

If you have already noticed. The variable ``flag`` was never set to false causing recursive call never terminating.
This will results into StackOverFlowError, but we do have catch block, ignoring the stack overflow error

```

catch(StackOverflowError err) {
	// IGNORE 
}
```

In this case, all the allocations are happenning in the `list` which is global variable. Every recursive call will add `1k bytes` of data into the list `list.add(arr)`. As we ignored the `StackOverflowError` has resulted into the `OutOfMemoryError`. This is how the output looks like

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

In large applications, having million/s lines of code, ignoring `StackOverflowError` even at single place, could result into severe issues. Debugging such issues are very very challenging.

# How to identify hidden StackOverflowError's from hotspot logs files
Whenever there is a crash, the Java Virtual Machine generates a hotspot error log file (`hs_error<%pid>.log`).
hs_error files have lot of information, one of the information is the number of times, there were stackoverflow errors generated before the crash. This is one of the clue to identify that there are stack overflow.
From the above program running with `-XX:+CrashOnOutOfMemoryError`, has gives below information about `StackOverflowError`

```
OutOfMemory and StackOverflow Exception counts:
StackOverflowErrors=154
```
