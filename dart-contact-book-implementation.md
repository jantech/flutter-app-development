# Contact Manager- Dart Project

---

## **User Story**

**As** a user who wants to store personal or work contacts,

**I want** a simple command-line application where I can add, view, search, update, and delete contacts,

**So that** I can quickly manage my contact list without needing a complex GUI or internet connection, and have the data persist between sessions.

---

## **Acceptance Criteria**

### **1. Contact Management**

* **Add Contact**

  * Given the app is running,
    When I choose “Add contact” and enter valid name, phone, and email,
    Then the contact is stored in memory and saved to `contacts.json`.
  * **Validation:**

    * Name cannot be empty.
    * Phone must contain at least 7 digits (allowing spaces, +, -, and parentheses).
    * Email must be in a valid format (`name@example.com`).
* **List Contacts**

  * Given the app has one or more contacts,
    When I choose “List contacts”,
    Then I see all contacts printed with their ID, name, phone, and email.
  * If no contacts exist, display “No contacts yet.”
* **Search Contacts**

  * Given the app has contacts,
    When I choose “Search contacts” and enter a search term,
    Then I see a list of contacts where the name, phone, or email contains that term (case-insensitive).
  * If no matches are found, display “No matches found.”
* **Update Contact**

  * Given I know a contact’s ID,
    When I choose “Update contact” and enter that ID,
    Then I can change the name, phone, and/or email (leaving blank to keep the old value).
  * Validation rules for phone/email still apply.
* **Delete Contact**

  * Given I know a contact’s ID,
    When I choose “Delete contact” and confirm with “y” or “yes”,
    Then that contact is removed from the list and the change is saved.

---

### **2. Data Persistence**

* **Load on Start**

  * Given `contacts.json` exists and contains valid JSON,
    When I start the application,
    Then all contacts in the file are loaded into memory.
  * If the file is missing or invalid, the app starts with an empty list.
* **Save on Change**

  * Given I add, update, or delete a contact,
    When the operation completes successfully,
    Then the updated list is saved to `contacts.json`.

---

### **3. CLI Interaction**

* The application displays a main menu with options:

  ```
  1) List contacts
  2) Add contact
  3) Search contacts
  4) Update contact
  5) Delete contact
  6) Exit
  ```
* The application continues showing the menu until the user selects “Exit” or types “exit”.
* The app gracefully handles:

  * Invalid menu choices.
  * Empty input when a non-empty value is required.
  * Invalid ID lookups.

---

### **4. Non-Functional Requirements**

* Must run with `dart run` without additional dependencies.
* Must store data in a human-readable JSON file.
* Code should be structured into:

  * `lib/contact_book.dart` (business logic)
  * `bin/main.dart` (CLI)
* Must handle file read/write errors without crashing, showing a warning instead.

---

# User Stories & Acceptance Criteria (with Code)

## Story 1 — Persist contacts to disk

**As a** user
**I want** my contacts stored in a JSON file
**So that** they’re available the next time I run the app

### Acceptance Criteria

1. If `contacts.json` doesn’t exist, the app starts with an empty book without crashing.
2. If `contacts.json` is invalid JSON, the app warns and starts with an empty book.
3. Saving writes **pretty-printed JSON**.
4. When saving and a file already exists, create a **`.bak` backup** first.

### Code (implemented in `contact_book.dart` → `load()` & `save()`)

#### Prepare the Contact Model
```dart
// contact_model.dart
import 'dart:convert';
import 'dart:io';

/// Simple Contact model
class Contact {
  final String id;
  String name;
  String phone;
  String email;

  Contact({
    required this.id,
    required this.name,
    required this.phone,
    required this.email,
  });

  factory Contact.fromMap(Map<String, dynamic> m) {
    return Contact(
      id: m['id'] as String,
      name: (m['name'] as String?) ?? '',
      phone: (m['phone'] as String?) ?? '',
      email: (m['email'] as String?) ?? '',
    );
  }

  Map<String, dynamic> toMap() => {
        'id': id,
        'name': name,
        'phone': phone,
        'email': email,
      };

  @override
  String toString() {
    return '$name (id: $id)\n  Phone: $phone\n  Email: $email';
  }
}
```
#### Prepare the `contact_book` Library with `Save` and `Load` contacts from json file
```dart
// contact_book.dart
/// ContactBook handles in-memory list + file persistence.
class ContactBook {
  final String filePath;
  final List<Contact> _contacts = [];

  ContactBook({this.filePath = 'contacts.json'});

  /// Load contacts from file; if file missing or invalid, start empty.
  Future<void> load() async {
    final file = File(filePath);
    if (!await file.exists()) return;

    try {
      final content = await file.readAsString();
      final data = jsonDecode(content) as List<dynamic>;
      _contacts
        ..clear()
        ..addAll(data.map((e) => Contact.fromMap(e as Map<String, dynamic>)));
    } catch (e) {
      stderr.writeln('Warning: Could not load contacts from $filePath: $e');
      _contacts.clear();
    }
  }

  /// Save current contacts list to file (overwrites).
  /// - pretty-printed JSON
  /// - creates .bak backup if file already exists
  Future<void> save() async {
    final file = File(filePath);

    // Backup
    if (await file.exists()) {
      final bak = File('$filePath.bak');
      await file.copy(bak.path);
    }

    // Pretty JSON
    final encoder = const JsonEncoder.withIndent('  ');
    final jsonString = encoder.convert(_contacts.map((c) => c.toMap()).toList());
    await file.writeAsString(jsonString);
  }

  void addContact(Contact c) => _contacts.add(c);

  List<Contact> listAll() => List.unmodifiable(_contacts);

  List<Contact> search(String q) {
    final query = q.toLowerCase();
    return _contacts.where((c) {
      return c.name.toLowerCase().contains(query) ||
          c.phone.toLowerCase().contains(query) ||
          c.email.toLowerCase().contains(query);
    }).toList();
  }

  Contact? getById(String id) {
    try {
      return _contacts.firstWhere((c) => c.id == id);
    } catch (_) {
      return null;
    }
  }

  bool delete(String id) {
    final before = _contacts.length;
    _contacts.removeWhere((c) => c.id == id);
    return _contacts.length < before;
  }

  /// For listing: natural sort by name (case-insensitive)
  List<Contact> listSortedByName() {
    final copy = [..._contacts];
    copy.sort((a, b) => a.name.toLowerCase().compareTo(b.name.toLowerCase()));
    return copy;
  }
}
```

---

## Story 2 — Add a contact with validation

**As a** user
**I want** to add a contact
**So that** I can store their name, phone, and email

### Acceptance Criteria

1. The app prompts for **Name**, **Phone**, and **Email**.
2. Name and Phone are **required**; Email is optional but, if provided, must look like a valid email.
3. Phone must be digits with optional `+`, spaces, dashes, parentheses.
4. Each new contact gets a **unique ID**.
5. After adding, the change is **saved** to disk.

### Code (`main.dart` → `addFlow`, validators, id)

```dart
// main.dart
import 'dart:io';
import 'dart:math';
import 'contact_book.dart';

/// Helper: print and read a line (handles nullable readLineSync)
String prompt(String message) {
  stdout.write(message);
  return stdin.readLineSync() ?? '';
}

/// Helper: read non-empty input (loops until non-empty)
String promptNonEmpty(String message) {
  while (true) {
    final s = prompt(message).trim();
    if (s.isNotEmpty) return s;
    print('Please enter a value.');
  }
}

/// Simple random-ish UUID (no external packages)
String newId() {
  final r = Random.secure();
  String fourHex() => r.nextInt(0x10000).toRadixString(16).padLeft(4, '0');
  return '${fourHex()}${fourHex()}-${fourHex()}-${fourHex()}-${fourHex()}-${fourHex()}${fourHex()}${fourHex()}';
}

bool isValidEmail(String s) {
  if (s.isEmpty) return true; // email optional
  // simple, pragmatic pattern (not fully RFC)
  final re = RegExp(r'^[^\s@]+@[^\s@]+\.[^\s@]+$');
  return re.hasMatch(s);
}

bool isValidPhone(String s) {
  // allow +, spaces, -, (), and digits; must contain at least 7 digits
  final allowed = RegExp(r'^[0-9+\-\s()]+$');
  if (!allowed.hasMatch(s)) return false;
  final digits = RegExp(r'\d').allMatches(s).length;
  return digits >= 7;
}

/// CLI flows
Future<void> addFlow(ContactBook book) async {
  print('\n--- Add Contact ---');

  final name = promptNonEmpty('Name: ');

  String phone;
  while (true) {
    phone = promptNonEmpty('Phone: ');
    if (isValidPhone(phone)) break;
    print('Invalid phone. Use digits and (+ - ( ) space). Must have ≥ 7 digits.');
  }

  String email;
  while (true) {
    email = prompt('Email (optional): ').trim();
    if (isValidEmail(email)) break;
    print('Invalid email format. Try again.');
  }

  final contact = Contact(id: newId(), name: name, phone: phone, email: email);
  book.addContact(contact);
  await book.save();
  print('Contact added.\n');
}
```

---

## Story 3 — List contacts

**As a** user
**I want** to list all contacts
**So that** I can review what’s stored

### Acceptance Criteria

1. If there are no contacts, show “No contacts yet.”
2. Otherwise, show all contacts **sorted by name** (case-insensitive).
3. A separator appears between contacts for readability.

### Code (`main.dart` → `listFlow`)

```dart
void listFlow(ContactBook book) {
  print('\n--- All Contacts ---');
  final all = book.listSortedByName();
  if (all.isEmpty) {
    print('No contacts yet.\n');
    return;
  }
  for (final c in all) {
    print(c);
    print('------------------');
  }
  print('');
}
```

---

## Story 4 — Search contacts

**As a** user
**I want** to search by name, phone, or email
**So that** I can quickly find someone

### Acceptance Criteria

1. Prompt for a search string; empty string cancels search.
2. Search is **case-insensitive** across name, phone, and email.
3. Display the **number of matches** before the results.

### Code (`main.dart` → `searchFlow`)

```dart
void searchFlow(ContactBook book) {
  print('\n--- Search Contacts ---');
  final q = prompt('Enter name, phone, or email to search: ').trim();
  if (q.isEmpty) {
    print('Empty search.\n');
    return;
  }
  final results = book.search(q);
  if (results.isEmpty) {
    print('No matches found.\n');
    return;
  }
  print('Matches: ${results.length}\n');
  for (final r in results) {
    print(r);
    print('------------------');
  }
  print('');
}
```

---

## Story 5 — Update a contact

**As a** user
**I want** to update an existing contact
**So that** I can fix mistakes or add info

### Acceptance Criteria

1. Prompt for the **contact ID**.
2. If ID not found, show “Contact not found.”
3. Show current values; **press Enter keeps** current.
4. Validate phone/email using the same rules as when adding.
5. Save changes to disk.

### Code (`main.dart` → `updateFlow`)

```dart
Future<void> updateFlow(ContactBook book) async {
  print('\n--- Update Contact ---');
  final id = prompt('Enter contact id to update: ').trim();
  if (id.isEmpty) {
    print('No id entered.\n');
    return;
  }
  final c = book.getById(id);
  if (c == null) {
    print('Contact not found.\n');
    return;
  }
  print('Press Enter to keep the current value.');

  final nameIn = prompt('Name [${c.name}]: ').trim();
  String phoneIn = prompt('Phone [${c.phone}]: ').trim();
  String emailIn = prompt('Email [${c.email}]: ').trim();

  if (nameIn.isNotEmpty) c.name = nameIn;

  if (phoneIn.isNotEmpty) {
    while (!isValidPhone(phoneIn)) {
      print('Invalid phone. Use digits and (+ - ( ) space). Must have ≥ 7 digits.');
      phoneIn = prompt('Phone [${c.phone}]: ').trim();
      if (phoneIn.isEmpty) break; // keep old
    }
    if (phoneIn.isNotEmpty) c.phone = phoneIn;
  }

  if (emailIn.isNotEmpty) {
    while (!isValidEmail(emailIn)) {
      print('Invalid email format. Try again.');
      emailIn = prompt('Email [${c.email}]: ').trim();
      if (emailIn.isEmpty) break; // keep old
    }
    if (emailIn.isNotEmpty) c.email = emailIn;
  }

  await book.save();
  print('Contact updated.\n');
}
```

---

## Story 6 — Delete a contact

**As a** user
**I want** to delete a contact
**So that** I can remove old or duplicate entries

### Acceptance Criteria

1. Prompt for the **contact ID**.
2. If not found, show “Contact not found.”
3. Ask for confirmation `y/yes` to proceed.
4. On success, save to disk and show “Deleted.”

### Code (`main.dart` → `deleteFlow`)

```dart
Future<void> deleteFlow(ContactBook book) async {
  print('\n--- Delete Contact ---');
  final id = prompt('Enter contact id to delete: ').trim();
  if (id.isEmpty) {
    print('No id entered.\n');
    return;
  }
  final c = book.getById(id);
  if (c == null) {
    print('Contact not found.\n');
    return;
  }
  final conf = prompt('Delete ${c.name}? (y/N): ').toLowerCase().trim();
  if (conf == 'y' || conf == 'yes') {
    final ok = book.delete(id);
    if (ok) {
      await book.save();
      print('Deleted.\n');
    } else {
      print('Could not delete.\n');
    }
  } else {
    print('Cancelled.\n');
  }
}
```

---

## Story 7 — Navigate via a robust CLI menu

**As a** user
**I want** a simple menu
**So that** I can perform actions without memorizing commands

### Acceptance Criteria

1. Show a numbered menu with: List, Add, Search, Update, Delete, Exit.
2. Invalid options show a clear message and re-display menu.
3. On Exit, show “Goodbye!” and terminate.

### Code (`main.dart` → `main()`)

```dart
Future<void> main() async {
  final book = ContactBook();
  await book.load();

  print('=== Simple Contact Book ===');
  while (true) {
    print('''
Choose an action:
  1) List contacts
  2) Add contact
  3) Search contacts
  4) Update contact
  5) Delete contact
  6) Exit
''');
    final choice = prompt('> ').trim();
    switch (choice) {
      case '1':
        listFlow(book);
        break;
      case '2':
        await addFlow(book);
        break;
      case '3':
        searchFlow(book);
        break;
      case '4':
        await updateFlow(book);
        break;
      case '5':
        await deleteFlow(book);
        break;
      case '6':
      case 'exit':
        print('Goodbye!');
        return;
      default:
        print('Unknown option. Try again.\n');
    }
  }
}
```

---

# How to run

1. Save the two files:

   * `contact_book.dart` (first big block)
   * `main.dart` (rest of blocks combined)
2. Run:

   ```bash
   dart run main.dart
   ```

---
