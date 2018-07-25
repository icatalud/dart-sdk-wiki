# Live program changes in the Dart VM ("Hot reload")

The Dart VM can apply changes to a running (live) program, which it calls _hot reload_. The semantics are very close to those of Smalltalk, which doesn't have a name for this feature since in most Smalltalk implementations program changes can only be made in a live programming environment.

The main principles of reload's behavior are

 * [The program behaves as if method lookup happens at every call.](#pervasive-late-binding)
 * [The "atoms" of reload are methods. Methods are never changed, but method dictionaries are updated with new methods.](#immutable-methods)
 * [Fields retain their values.](#state-is-retained)

It's also important to note that hot reloading only changes the behavior of the program going forward. It does not change the program's state to reflect what would have happened if the new program had been running from the beginning. The behavior of a hot reloaded program can differ from the behavior of both the old and the new program running without a hot reload.

## Pervasive late-binding

The program behaves as if method lookup happens at every call. Any call that runs after a reload, even those call sites that have previously run or are pending on the stack, should reflect the result of a lookup in the new program rather than the old program.

(In the implementation, the VM aggressively attempts to avoid method lookups by using caches and inlining. To preserve the semantics of hot reloadinline , caches are cleared and inlining is unfolded at the time of a reload.)

Before:
```
class C {
  foo() => "before";
}
main() {
  var c = new C();
  print(c.foo());
  reload();
  print(c.foo());
}
```

After:
```
class C {
  foo() => "after";
}
main() {
  var c = new C();
  print(c.foo());
  reload();
  print(c.foo());
}
```

Result:
```
before
after
```

## Immutable methods

The "atoms" of reload are methods. Methods are never mutated. Changes to a method declaration create a new method, mutating the class or library's method dictionary. The old method may still exist if has been captured by a closure or stack frame.

Closures capture their function when they are created. A closure always has the same function before and after a change, and all invocations of a given closure run the same function.

Before:
```
class C {
  deferredPrint() {
    return () => print("before");
  }
}
main() {
  var c = new C();
  var closure = c.deferredPrint();
  closure();
  reload();
  closure();
}
```

After:
```
class C {
  deferredPrint() {
    return () => print("after");
  }
}
main() {
  var c = new C();
  var closure = c.deferredPrint();
  closure();
  reload();
  closure();
}
```

Result:
```
before
before
```

Stack frames capture their function. A stack frame will continue to run the same function until it returns.

Before:
```
main() {
  print("before");
  reload();
  print("before");
}
```

After:
```
main() {
  print("after");
  reload();
  print("after");
}
```

Result:
```
before
before
```

## State is retained

Hot reload does not reset fields, neither the fields of instances nor those of classes or libraries. Resetting all fields would make a hot reload equivalent to a restart.

Before:
```
var foo = "before";
main() {
  print(foo);
  reload();
  print(foo);
}
```

After:
```
var foo = "after";
main() {
  print(foo);
  reload();
  print(foo);
}
```

Result:
```
before
before
```

Remember that statics fields in Dart are lazily initialized. If a field has not been accessed before a change, the new initializer will run if it is accessed after the change.

Before:
```
var foo = "before";
main() {
  reload();
  print(foo);
}
```

After:
```
var foo = "after";
main() {
  reload();
  print(foo);
}
```

Result:
```
after
```

## Limitations

The Dart VM will reject some types of changes that it cannot handle. Changes are atomic: they are never partially applied. If a change is rejected, the program continues as if no change had been attempted.

The cases where a reload is rejected are

- Changes between regular classes, enums and typedefs (`class C {}` <=> `enum C {}` <=> `typedef void C()`)

- Changes to the number of type arguments in a class. (`class C<T>` <=> `class C<T,S> {}`)

(Schema changes to type arguments are not safe because of the type argument prefix optimization.)

- Changes to a library with deferred imports (`import 'a' deferred as a`).

(This simply hasn't been implemented.)

- Changes to the number of native fields (`class C extends NativeFieldWrapperClass1 {}` <=> `class C extends NativeFieldWrapperClass2 {}`).

(Changing the shape of native wrappers generally requires corresponding changes to the native code.)

The Dart VM also lacks a way to migrate field values during a change. In particular, there is no way to communicate the intention of a renamed field to carry the value over from the old name to the new name. The Dart VM only sees addition and removal of unrelated fields.

## Defects

In the following cases, the VM will accept changes, but it can produce incorrect results.

- A call without an explicit receiver that changes meaning from an instance call to a static call (or vice versa), that was on the stack at the time of reload.

Before:
```
bar() => "before";
class C {
  foo() {
    print(bar());
    reload();
    print(bar());
  }
}
main() {
  new C().foo();
}
```

After:
```
class C {
  bar() => "after";
  foo() {
    print(bar());
    reload();
    print(bar());
  }
}
main() {
  new C().foo();
}
```

Correct result:
```
before
after
```

Actual result:
```
before
before
```

- A super call that was on the stack at the time of reload has a new target.

Before:
```
class A { 
  bar() => "A-bar";
}
class B extends A {}
class C extends B {
  bar() => "C-bar";
  foo() {
    print(super.bar());
    reload();
    print(super.bar());
  }
}
main() {
  new C().foo();
}
```

After:
```
class A {}
class B extends A {
  bar() => "B-bar";
}
class C extends B {
  bar() => "C-bar";
  foo() {
    print(super.bar());
    reload();
    print(super.bar());
  }
}
main() {
  new C().foo();
}
```

Correct result:
```
A-bar
B-bar
```

Actual result:
```
A-bar
A-bar
```

- A static call that was on the stack at the time of reload changes between having a target and having no target.

Before:
```
foo() => "before";
main() {
  print(foo());
  reload();
  print(foo());
}
```

After:
```
main() {
  print(foo());
  reload();
  print(foo());
}
```

Correct result:
```
before
NoSuchMethod: foo
```

Actual result:
```
before
before
```
