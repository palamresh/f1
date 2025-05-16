# f1

import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:flutter/material.dart';
import 'package:intl/intl.dart';

class HomeScreen extends StatefulWidget {
  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  double totalTodayCalories = 0;
  double totalRangeCalories = 0;
  List<DocumentSnapshot> filteredEntries = [];

  DateTime? fromDate;
  DateTime? toDate;

  @override
  void initState() {
    super.initState();
    calculateTodayCalories();
  }

  Future<void> calculateTodayCalories() async {
    String today = DateFormat('yyyy-MM-dd').format(DateTime.now());
    QuerySnapshot snapshot = await FirebaseFirestore.instance
        .collection('food_entries')
        .where('date', isEqualTo: today)
        .get();

    double total = 0;
    for (var doc in snapshot.docs) {
      total += doc['totalCalories'];
    }

    setState(() {
      totalTodayCalories = total;
    });
  }

  void showQuantityDialog(DocumentSnapshot foodDoc) {
    TextEditingController quantityController = TextEditingController();

    showDialog(
      context: context,
      builder: (context) {
        return AlertDialog(
          title: Text("Enter quantity (grams)"),
          content: TextField(
            controller: quantityController,
            keyboardType: TextInputType.number,
            decoration: InputDecoration(hintText: "e.g. 150"),
          ),
          actions: [
            TextButton(
              child: Text("Add"),
              onPressed: () async {
                int quantity = int.tryParse(quantityController.text) ?? 0;
                double per100g = foodDoc['caloriesPer100g'].toDouble();
                double totalCalories = (quantity * per100g) / 100;
                String today = DateFormat('yyyy-MM-dd').format(DateTime.now());

                await FirebaseFirestore.instance
                    .collection('food_entries')
                    .add({
                  'foodName': foodDoc['name'],
                  'quantity': quantity,
                  'totalCalories': totalCalories,
                  'date': today,
                });

                Navigator.pop(context);
                calculateTodayCalories(); // refresh total
              },
            )
          ],
        );
      },
    );
  }

  Future<void> pickDate({required bool isFrom}) async {
    DateTime initial = DateTime.now();
    DateTime? picked = await showDatePicker(
      context: context,
      initialDate: initial,
      firstDate: DateTime(2023),
      lastDate: DateTime(2100),
    );

    if (picked != null) {
      setState(() {
        if (isFrom) {
          fromDate = picked;
        } else {
          toDate = picked;
        }
      });
    }
  }

  Future<void> fetchEntriesByDateRange() async {
    if (fromDate == null || toDate == null) return;

    String from = DateFormat('yyyy-MM-dd').format(fromDate!);
    String to = DateFormat('yyyy-MM-dd').format(toDate!);

    QuerySnapshot snapshot = await FirebaseFirestore.instance
        .collection('food_entries')
        .where('date', isGreaterThanOrEqualTo: from)
        .where('date', isLessThanOrEqualTo: to)
        .get();

    double total = 0;
    for (var doc in snapshot.docs) {
      total += doc['totalCalories'];
    }

    setState(() {
      filteredEntries = snapshot.docs;
      totalRangeCalories = total;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Calorie Calculator"),
        centerTitle: true,
      ),
      body: Column(
        children: [
          // 🔹 Food List
          Expanded(
            child: StreamBuilder<QuerySnapshot>(
              stream: FirebaseFirestore.instance
                  .collection('foods')
                  .orderBy('name')
                  .snapshots(),
              builder: (context, snapshot) {
                if (!snapshot.hasData)
                  return Center(child: CircularProgressIndicator());

                final foods = snapshot.data!.docs;

                return ListView.builder(
                  itemCount: foods.length,
                  itemBuilder: (context, index) {
                    var food = foods[index];
                    return ListTile(
                      title: Text(food['name']),
                      subtitle:
                          Text("Per 100g: ${food['caloriesPer100g']} kcal"),
                      trailing: Icon(Icons.add),
                      onTap: () => showQuantityDialog(food),
                    );
                  },
                );
              },
            ),
          ),

          Divider(),

          // 🔸 Date Range Filter
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: Row(
              mainAxisAlignment: MainAxisAlignment.spaceAround,
              children: [
                TextButton(
                  onPressed: () => pickDate(isFrom: true),
                  child: Text(fromDate == null
                      ? "From Date"
                      : DateFormat('yyyy-MM-dd').format(fromDate!)),
                ),
                TextButton(
                  onPressed: () => pickDate(isFrom: false),
                  child: Text(toDate == null
                      ? "To Date"
                      : DateFormat('yyyy-MM-dd').format(toDate!)),
                ),
                ElevatedButton(
                  onPressed: fetchEntriesByDateRange,
                  child: Text("Show"),
                ),
              ],
            ),
          ),

          // 🔸 Filtered Entries List
          if (filteredEntries.isNotEmpty)
            Expanded(
              child: ListView.builder(
                itemCount: filteredEntries.length,
                itemBuilder: (context, index) {
                  var entry = filteredEntries[index];
                  return ListTile(
                    title: Text("${entry['foodName']}"),
                    subtitle: Text(
                        "${entry['quantity']}g - ${entry['totalCalories']} kcal"),
                    trailing: Text(entry['date']),
                  );
                },
              ),
            ),

          // 🔹 Calorie Summary
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: Column(
              children: [
                Text(
                  "Today’s Calories: ${totalTodayCalories.toStringAsFixed(1)} kcal",
                  style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
                ),
                if (fromDate != null && toDate != null)
                  Text(
                    "Filtered Total: ${totalRangeCalories.toStringAsFixed(1)} kcal",
                    style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
                  ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
