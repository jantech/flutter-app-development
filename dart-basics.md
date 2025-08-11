# **Dart** (for beginners) 🚀

Nice — Dart is a great choice (especially if you later want to build Flutter apps). 

---

## 1) Quick orientation — what Dart is good for

* Frontend mobile & desktop apps (via **Flutter**)
* Command-line tools, scripts, and servers
* Fast dev cycle, sound **null safety**, and strong tooling

---

## 2) Install & verify the SDK

* Check installation with:

```bash
dart --version
```

* Create a quick project:

```bash
dart create -t console-simple my_app
cd my_app
dart run
```

(If `dart` isn’t found, follow official install instructions for your OS)

---

## 3) Your first program — Hello World

```dart
void main() {
  print('Hello, Dart!');
}
```

Save as `bin/main.dart` and run:

```bash
dart run bin/main.dart
```

---

## 4) Basic syntax & types (copyable)

```dart
void main() {
  int i = 5;
  double d = 3.14;
  String name = 'Asha';
  bool flag = true;

  var inferred = 'I am a string';   // type inferred as String
  dynamic anything = 42;             // avoids static type checks

  final city = 'Pune';               // runtime constant
  const pi = 3.14159;                // compile-time constant

  print('$name from $city, pi=$pi');
}
```

---

## 5) Null safety (essential)

```dart
String? maybeName;               // nullable
print(maybeName?.length ?? 0);   // safe access with null-aware operators

late String lazyInit;            // promise to initialize later
lazyInit = 'Now initialized';
print(lazyInit);
```

Dart’s sound null safety prevents many NPEs — learn `?`, `!`, `??`, `?.`, `late`.

---

## 6) Control flow & collections

```dart
// if / for
for (var i = 0; i < 3; i++) print(i);

// Collections
List<int> nums = [1, 2, 3];
Map<String, int> ages = {'Ana': 30, 'Ben': 28};
Set<String> tags = {'dart', 'flutter'};

// Useful operators
var copy = [...nums];         // spread
var maybe = ages['Ana'] ?? 0; // null-coalescing
```

---

## 7) Functions (positional, named, default)

```dart
int add(int a, int b) => a + b;

String greet(String name, {int age = 0}) {
  return 'Hi $name, age $age';
}

void main() {
  print(add(2,3));
  print(greet('Riya', age: 27));
}
```

---

## 8) Classes & OOP

```dart
class Person {
  String name;
  int age;

  Person(this.name, this.age);

  // named constructor
  Person.fromJson(Map<String, dynamic> json)
      : name = json['name'] as String,
        age = json['age'] as int;

  Map<String, dynamic> toJson() => {'name': name, 'age': age};

  void sayHello() => print('Hi, I am $name');
}

void main() {
  var p = Person('Maya', 24);
  p.sayHello();
}
```

Also learn: `extends`, `implements`, `mixins`, `abstract` classes, `get`/`set`, and `factory` constructors.

---

## 9) Asynchronous programming (Future & Stream)

```dart
Future<String> fetchUser() async {
  await Future.delayed(Duration(seconds: 1));
  return 'Sam';
}

void main() async {
  try {
    final user = await fetchUser();
    print(user);
  } catch (e) {
    print('Error: $e');
  }
}

// Stream example
Stream<int> counter() async* {
  for (var i = 1; i <= 3; i++) {
    await Future.delayed(Duration(milliseconds: 300));
    yield i;
  }
}

void useStream() async {
  await for (final n in counter()) {
    print(n);
  }
}
```

---

* Useful commands:

  * `dart pub get` — fetch packages
  * `dart run` — run scripts
  * `dart format .` — format code
  * `dart analyze` — static analyzer
  * `dart test` — run tests

---

## 10) Tips & best practices

* Prefer `final` over `var` when value won’t change.
* Use named parameters for clearer APIs.
* Keep functions small and testable.
* Use `dart format` and `dart analyze` frequently.
* Try DartPad (online REPL) to experiment quickly. https://dartpad.dev/

---
