# Flutter Contact Book — Full Project

This document contains a ready-to-run Flutter project that converts the CLI Contact Book into a simple Flutter app with UI, persistence, and basic CRUD (Create, Read, Update, Delete) + search. Copy the files into a Flutter project (steps below) and run.

---

=== FILE: pubspec.yaml ===

```yaml
name: contact_book_flutter
description: A simple contact book app (learning project)
publish_to: "none"
version: 0.0.1

environment:
  sdk: ">=2.17.0 <4.0.0"

dependencies:
  flutter:
    sdk: flutter
  provider: ^6.1.5
  path_provider: ^2.1.5

  # The following adds the Cupertino Icons font to your application.
  cupertino_icons: ^1.0.2

dev_dependencies:
  flutter_test:
    sdk: flutter

flutter:
  uses-material-design: true
```

---

=== FILE: lib/models/contact.dart ===

```dart
// lib/models/contact.dart

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

  factory Contact.fromJson(Map<String, dynamic> json) => Contact(
        id: json['id'] as String,
        name: (json['name'] as String?) ?? '',
        phone: (json['phone'] as String?) ?? '',
        email: (json['email'] as String?) ?? '',
      );

  Map<String, dynamic> toJson() => {
        'id': id,
        'name': name,
        'phone': phone,
        'email': email,
      };

  String initials() {
    final parts = name.trim().split(RegExp('\\s+'));
    if (parts.isEmpty) return '?';
    if (parts.length == 1) return parts.first.substring(0, 1).toUpperCase();
    return (parts[0][0] + parts[1][0]).toUpperCase();
  }
}
```

---

=== FILE: lib/services/contact_storage.dart ===

```dart
// lib/services/contact_storage.dart

import 'dart:convert';
import 'dart:io';

import 'package:path_provider/path_provider.dart';

import '../models/contact.dart';

class ContactStorage {
  final String fileName;

  ContactStorage({this.fileName = 'contacts.json'});

  Future<File> _localFile() async {
    final dir = await getApplicationDocumentsDirectory();
    return File('${dir.path}/$fileName');
  }

  Future<List<Contact>> loadContacts() async {
    try {
      final file = await _localFile();
      if (!await file.exists()) return [];
      final jsonStr = await file.readAsString();
      final List<dynamic> list = jsonDecode(jsonStr) as List<dynamic>;
      return list.map((e) => Contact.fromJson(e as Map<String, dynamic>)).toList();
    } catch (e) {
      // On any error, return empty list so the app can still run.
      return [];
    }
  }

  Future<void> saveContacts(List<Contact> contacts) async {
    final file = await _localFile();
    final jsonStr = jsonEncode(contacts.map((c) => c.toJson()).toList());
    await file.writeAsString(jsonStr);
  }
}
```

---

=== FILE: lib/models/contact_book_model.dart ===

```dart
// lib/models/contact_book_model.dart

import 'package:flutter/foundation.dart';
import '../models/contact.dart';
import '../services/contact_storage.dart';

class ContactBookModel extends ChangeNotifier {
  final ContactStorage storage;
  final List<Contact> _contacts = [];

  ContactBookModel({required this.storage});

  List<Contact> get contacts => List.unmodifiable(_contacts);

  Future<void> load() async {
    _contacts.clear();
    final loaded = await storage.loadContacts();
    _contacts.addAll(loaded);
    notifyListeners();
  }

  Future<void> add(Contact contact) async {
    _contacts.add(contact);
    await storage.saveContacts(_contacts);
    notifyListeners();
  }

  Future<void> update(Contact updated) async {
    final idx = _contacts.indexWhere((c) => c.id == updated.id);
    if (idx != -1) {
      _contacts[idx] = updated;
      await storage.saveContacts(_contacts);
      notifyListeners();
    }
  }

  Future<void> delete(String id) async {
    _contacts.removeWhere((c) => c.id == id);
    await storage.saveContacts(_contacts);
    notifyListeners();
  }

  List<Contact> search(String q) {
    final query = q.trim().toLowerCase();
    if (query.isEmpty) return contacts;
    return _contacts.where((c) {
      return c.name.toLowerCase().contains(query) ||
          c.phone.toLowerCase().contains(query) ||
          c.email.toLowerCase().contains(query);
    }).toList();
  }
}
```

---

=== FILE: lib/screens/contact_form.dart ===

```dart
// lib/screens/contact_form.dart

import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

import '../models/contact.dart';
import '../models/contact_book_model.dart';

class ContactFormPage extends StatefulWidget {
  final Contact? contact; // if null -> create

  const ContactFormPage({Key? key, this.contact}) : super(key: key);

  @override
  State<ContactFormPage> createState() => _ContactFormPageState();
}

class _ContactFormPageState extends State<ContactFormPage> {
  final _formKey = GlobalKey<FormState>();
  late TextEditingController _nameCtrl;
  late TextEditingController _phoneCtrl;
  late TextEditingController _emailCtrl;

  @override
  void initState() {
    super.initState();
    _nameCtrl = TextEditingController(text: widget.contact?.name ?? '');
    _phoneCtrl = TextEditingController(text: widget.contact?.phone ?? '');
    _emailCtrl = TextEditingController(text: widget.contact?.email ?? '');
  }

  @override
  void dispose() {
    _nameCtrl.dispose();
    _phoneCtrl.dispose();
    _emailCtrl.dispose();
    super.dispose();
  }

  void _save() async {
    if (!_formKey.currentState!.validate()) return;
    final model = context.read<ContactBookModel>();
    if (widget.contact == null) {
      final newContact = Contact(
        id: DateTime.now().microsecondsSinceEpoch.toString(),
        name: _nameCtrl.text.trim(),
        phone: _phoneCtrl.text.trim(),
        email: _emailCtrl.text.trim(),
      );
      await model.add(newContact);
    } else {
      final updated = Contact(
        id: widget.contact!.id,
        name: _nameCtrl.text.trim(),
        phone: _phoneCtrl.text.trim(),
        email: _emailCtrl.text.trim(),
      );
      await model.update(updated);
    }
    if (mounted) Navigator.of(context).pop();
  }

  @override
  Widget build(BuildContext context) {
    final isEditing = widget.contact != null;
    return Scaffold(
      appBar: AppBar(title: Text(isEditing ? 'Edit Contact' : 'Add Contact')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Form(
          key: _formKey,
          child: Column(
            children: [
              TextFormField(
                controller: _nameCtrl,
                decoration: const InputDecoration(labelText: 'Name'),
                validator: (v) => (v ?? '').trim().isEmpty ? 'Enter a name' : null,
              ),
              const SizedBox(height: 12),
              TextFormField(
                controller: _phoneCtrl,
                decoration: const InputDecoration(labelText: 'Phone'),
                keyboardType: TextInputType.phone,
                validator: (v) => (v ?? '').trim().isEmpty ? 'Enter a phone' : null,
              ),
              const SizedBox(height: 12),
              TextFormField(
                controller: _emailCtrl,
                decoration: const InputDecoration(labelText: 'Email'),
                keyboardType: TextInputType.emailAddress,
                validator: (v) {
                  final s = (v ?? '').trim();
                  if (s.isEmpty) return 'Enter an email';
                  final valid = RegExp(r"^[^@\s]+@[^@\s]+\.[^@\s]+$");
                  if (!valid.hasMatch(s)) return 'Enter a valid email';
                  return null;
                },
              ),
              const SizedBox(height: 24),
              ElevatedButton.icon(
                onPressed: _save,
                icon: const Icon(Icons.save),
                label: Text(isEditing ? 'Save' : 'Add'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

=== FILE: lib/screens/home_screen.dart ===

```dart
// lib/screens/home_screen.dart

import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

import '../models/contact.dart';
import '../models/contact_book_model.dart';
import 'contact_form.dart';

class HomeScreen extends StatefulWidget {
  const HomeScreen({Key? key}) : super(key: key);

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  String _query = '';
  final _searchCtrl = TextEditingController();

  @override
  void dispose() {
    _searchCtrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final model = context.watch<ContactBookModel>();
    final List<Contact> results = model.search(_query);

    return Scaffold(
      appBar: AppBar(title: const Text('Contact Book')),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: TextField(
              controller: _searchCtrl,
              decoration: const InputDecoration(
                labelText: 'Search',
                prefixIcon: Icon(Icons.search),
                border: OutlineInputBorder(),
              ),
              onChanged: (v) => setState(() => _query = v),
            ),
          ),
          Expanded(
            child: results.isEmpty
                ? const Center(child: Text('No contacts'))
                : ListView.separated(
                    itemCount: results.length,
                    separatorBuilder: (_, __) => const Divider(height: 0),
                    itemBuilder: (context, index) {
                      final c = results[index];
                      return ListTile(
                        leading: CircleAvatar(child: Text(c.initials())),
                        title: Text(c.name),
                        subtitle: Text('${c.phone}\n${c.email}'),
                        isThreeLine: true,
                        trailing: PopupMenuButton<String>(
                          onSelected: (v) async {
                            if (v == 'edit') {
                              await Navigator.of(context).push(MaterialPageRoute(
                                  builder: (_) => ContactFormPage(contact: c)));
                            } else if (v == 'delete') {
                              final ok = await showDialog<bool>(
                                context: context,
                                builder: (ctx) => AlertDialog(
                                  title: const Text('Delete contact'),
                                  content: Text('Delete ${c.name}?'),
                                  actions: [
                                    TextButton(onPressed: () => Navigator.of(ctx).pop(false), child: const Text('Cancel')),
                                    TextButton(onPressed: () => Navigator.of(ctx).pop(true), child: const Text('Delete')),
                                  ],
                                ),
                              );
                              if (ok == true) await context.read<ContactBookModel>().delete(c.id);
                            }
                          },
                          itemBuilder: (_) => [
                            const PopupMenuItem(value: 'edit', child: Text('Edit')),
                            const PopupMenuItem(value: 'delete', child: Text('Delete')),
                          ],
                        ),
                        onTap: () {
                          // Quick edit on tap
                          Navigator.of(context).push(MaterialPageRoute(builder: (_) => ContactFormPage(contact: c)));
                        },
                      );
                    },
                  ),
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () async {
          await Navigator.of(context).push(MaterialPageRoute(builder: (_) => const ContactFormPage()));
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

---

=== FILE: lib/main.dart ===

```dart
// lib/main.dart

import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

import 'models/contact_book_model.dart';
import 'services/contact_storage.dart';
import 'screens/home_screen.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  final storage = ContactStorage();
  final model = ContactBookModel(storage: storage);
  await model.load(); // load persisted contacts before showing UI

  runApp(MyApp(model: model));
}

class MyApp extends StatelessWidget {
  final ContactBookModel model;
  const MyApp({Key? key, required this.model}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider<ContactBookModel>.value(
      value: model,
      child: MaterialApp(
        title: 'Contact Book',
        theme: ThemeData(primarySwatch: Colors.blue),
        home: const HomeScreen(),
      ),
    );
  }
}
```

---

# How to use

1. Create a Flutter project:

```bash
flutter create contact_book_flutter
cd contact_book_flutter
```

2. Replace the generated `pubspec.yaml` and `lib/` contents with the files above (or copy/paste each file into the corresponding path).
3. Run:

```bash
flutter pub get
flutter run
```

4. The app will persist contacts in a local file inside the app documents directory (platform-specific). When you add, edit, or delete contacts, they are saved.

---
