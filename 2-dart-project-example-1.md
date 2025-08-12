Let’s build a small, practical **Contact Book CLI** in Dart that demonstrates classes, collections, null-safety, and file I/O.
I’ll give you a single-file, ready-to-run program plus a step-by-step explanation of the important parts.

---

# Full code — `contact_book.dart`

Save the file as `contact_book.dart` and run it with `dart run contact_book.dart`.

```dart
// contact_book.dart
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

/// ContactBook handles in-memory list + file persistence.
class ContactBook {
  final String filePath;
  final List<Contact> _contacts = [];

  ContactBook({this.filePath = 'contacts.json'});

  /// Load contacts from file; if file missing or invalid, start empty.
  Future<void> load() async {
    final file = File(filePath);
    if (!await file.exists()) {
      return;
    }
    try {
      final content = await file.readAsString();
      final data = jsonDecode(content) as List<dynamic>;
      _contacts.clear();
      for (final e in data) {
        _contacts.add(Contact.fromMap(e as Map<String, dynamic>));
      }
    } catch (e) {
      print('Warning: Could not load contacts from $filePath: $e');
      _contacts.clear();
    }
  }

  /// Save current contacts list to file (overwrites).
  Future<void> save() async {
    final file = File(filePath);
    final json = jsonEncode(_contacts.map((c) => c.toMap()).toList());
    await file.writeAsString(json);
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
}

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

String newId() => DateTime.now().microsecondsSinceEpoch.toString();

/// CLI flows
Future<void> addFlow(ContactBook book) async {
  print('\n--- Add Contact ---');
  final name = promptNonEmpty('Name: ');
  final phone = promptNonEmpty('Phone: ');
  final email = promptNonEmpty('Email: ');
  final contact = Contact(id: newId(), name: name, phone: phone, email: email);
  book.addContact(contact);
  await book.save();
  print('Contact added.\n');
}

void listFlow(ContactBook book) {
  print('\n--- All Contacts ---');
  final all = book.listAll();
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
  for (final r in results) {
    print(r);
    print('------------------');
  }
  print('');
}

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
  final name = prompt('Name [${c.name}]: ').trim();
  final phone = prompt('Phone [${c.phone}]: ').trim();
  final email = prompt('Email [${c.email}]: ').trim();

  if (name.isNotEmpty) c.name = name;
  if (phone.isNotEmpty) c.phone = phone;
  if (email.isNotEmpty) c.email = email;

  await book.save();
  print('Contact updated.\n');
}

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
  final conf = prompt('Delete ${c.name}? (y/N): ').toLowerCase();
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

# Step-by-step explanation (what each piece does)

1. **Model (`Contact`)**

   * Small class with `id`, `name`, `phone`, `email`.
   * `fromMap` / `toMap` convert between `Contact` and `Map` for JSON serialization.
   * `toString()` for pretty printing.

2. **Data store (`ContactBook`)**

   * Keeps an in-memory `List<Contact>`.
   * `load()` reads `contacts.json`, decodes JSON into `Contact` objects.
   * `save()` writes the contact list back to `contacts.json` (JSON array of maps).
   * `search`, `getById`, `delete`, `addContact` provide in-memory manipulations.

3. **CLI helpers**

   * `prompt()` wraps `stdin.readLineSync()` to avoid dealing with nulls.
   * `promptNonEmpty()` forces the user to give a non-empty answer.

4. **Flows**

   * `addFlow`, `listFlow`, `searchFlow`, `updateFlow`, `deleteFlow` implement user interactions and call `book.save()` after changes.

5. **Main loop**

   * Loads contacts, shows a basic menu, and calls the appropriate flow until the user exits.

6. **Persistence**

   * Contacts are saved to `contacts.json` in the directory you run the script from.
   * Example file contents after adding one contact:

   ```json
   [
     {
       "id": "1700000000000000",
       "name": "Asha",
       "phone": "+91-9999999999",
       "email": "asha@example.com"
     }
   ]
   ```

---

# How to run it

1. Put the code into `contact_book.dart`.
2. From a terminal in the same folder:

   * `dart run contact_book.dart`
3. The app will create `contacts.json` when you add your first contact.

(If you prefer to create a full package, run `dart create -t console-simple contact_book`, replace `bin/contact_book.dart` with this code and run `dart run`.)

---

