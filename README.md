# OOPS-mini-project
#include <algorithm>
#include <cctype>
#include <iomanip>
#include <iostream>
#include <limits>
#include <map>
#include <memory>
#include <sstream>
#include <string>
#include <vector>

using namespace std;

// Abstract base class representing a general record.
class Record {
private:
    int id;
    string title;
    double amount;

public:
    Record(int recordId, const string& recordTitle, double recordAmount)
        : id(recordId), title(recordTitle), amount(recordAmount) {}

    virtual ~Record() {}
    virtual void display() const = 0;

    int getId() const { return id; }
    string getTitle() const { return title; }
    double getAmount() const { return amount; }

protected:
    void setTitle(const string& newTitle) { title = newTitle; }
    void setAmount(double newAmount) { amount = newAmount; }
};

// Transaction inherits from Record and adds expense-specific details.
class Transaction : public Record {
private:
    string category;
    string date;
    string notes;

public:
    Transaction(int id, const string& title, double amount,
                const string& category, const string& date,
                const string& notes = "")
        : Record(id, title, amount), category(category), date(date), notes(notes) {}

    void display() const override {
        cout << left << setw(5) << getId()
             << setw(24) << getTitle()
             << setw(18) << category
             << setw(14) << date
             << right << setw(12) << fixed << setprecision(2) << getAmount()
             << "\n";
    }

    string getCategory() const { return category; }
    string getDate() const { return date; }
    string getNotes() const { return notes; }

    void update(const string& title, double amount, const string& newCategory,
                const string& newDate, const string& newNotes) {
        setTitle(title);
        setAmount(amount);
        category = newCategory;
        date = newDate;
        notes = newNotes;
    }
};

// Abstract base class for reports.
class Report {
public:
    virtual ~Report() {}
    virtual void generate(const vector<Transaction>& transactions,
                          double budget) const = 0;
};

class SummaryReport : public Report {
public:
    void generate(const vector<Transaction>& transactions,
                  double budget) const override {
        double spent = 0.0;
        for (const auto& transaction : transactions) {
            spent += transaction.getAmount();
        }
        const double remaining = budget - spent;
        const double percentage = budget > 0 ? (spent / budget) * 100.0 : 0.0;

        cout << "\n========== MONTHLY SUMMARY ==========\n";
        cout << "Monthly budget : Rs. " << fixed << setprecision(2) << budget << "\n";
        cout << "Total spent    : Rs. " << spent << "\n";
        cout << "Remaining      : Rs. " << remaining << "\n";
        cout << "Budget used    : " << setprecision(1) << percentage << "%\n";
        cout << "Transactions   : " << transactions.size() << "\n";
        if (remaining >= 0) {
            cout << "Status         : Within budget\n";
        } else {
            cout << "Status         : Over budget by Rs. " << fixed << setprecision(2)
                 << -remaining << "\n";
        }
    }
};

class CategoryReport : public Report {
public:
    void generate(const vector<Transaction>& transactions,
                  double /*budget*/) const override {
        map<string, double> totals;
        for (const auto& transaction : transactions) {
            totals[transaction.getCategory()] += transaction.getAmount();
        }

        cout << "\n========== CATEGORY BREAKDOWN =======\n";
        if (totals.empty()) {
            cout << "No transactions available.\n";
            return;
        }

        double highestAmount = 0.0;
        string highestCategory;
        for (const auto& item : totals) {
            if (item.second > highestAmount) {
                highestAmount = item.second;
                highestCategory = item.first;
            }
        }

        for (const auto& item : totals) {
            int barLength = highestAmount > 0
                ? static_cast<int>((item.second / highestAmount) * 30)
                : 0;
            cout << left << setw(18) << item.first << " Rs. "
                 << right << setw(9) << fixed << setprecision(2) << item.second
                 << "  " << string(barLength, '#') << "\n";
        }
        cout << "Highest spending category: " << highestCategory
             << " (Rs. " << fixed << setprecision(2) << highestAmount << ")\n";
    }
};

class ExpenseTracker {
private:
    vector<Transaction> transactions;
    double monthlyBudget;
    int nextId;

    static string toLower(string text) {
        transform(text.begin(), text.end(), text.begin(),
                  [](unsigned char ch) { return static_cast<char>(tolower(ch)); });
        return text;
    }

    static void printTableHeader() {
        cout << "\n" << left << setw(5) << "ID"
             << setw(24) << "TITLE"
             << setw(18) << "CATEGORY"
             << setw(14) << "DATE"
             << right << setw(12) << "AMOUNT (Rs.)" << "\n";
        cout << string(73, '-') << "\n";
    }

    Transaction* findById(int id) {
        for (auto& transaction : transactions) {
            if (transaction.getId() == id) return &transaction;
        }
        return nullptr;
    }

public:
    explicit ExpenseTracker(double budget) : monthlyBudget(budget), nextId(1) {}

    double total() const {
        double sum = 0.0;
        for (const auto& transaction : transactions) sum += transaction.getAmount();
        return sum;
    }

    void add(const string& title, double amount, const string& category,
             const string& date, const string& notes = "") {
        transactions.emplace_back(nextId++, title, amount, category, date, notes);
    }

    bool edit(int id, const string& title, double amount, const string& category,
              const string& date, const string& notes) {
        Transaction* transaction = findById(id);
        if (!transaction) return false;
        transaction->update(title, amount, category, date, notes);
        return true;
    }

    bool remove(int id) {
        auto it = find_if(transactions.begin(), transactions.end(),
                          [id](const Transaction& transaction) {
                              return transaction.getId() == id;
                          });
        if (it == transactions.end()) return false;
        transactions.erase(it);
        return true;
    }

    void showAll() const {
        if (transactions.empty()) {
            cout << "\nNo transactions found.\n";
            return;
        }
        printTableHeader();
        for (const auto& transaction : transactions) transaction.display();
        cout << string(73, '-') << "\n";
        cout << "Total: Rs. " << fixed << setprecision(2) << total() << "\n";
    }

    void search(const string& query) const {
        const string needle = toLower(query);
        bool found = false;
        printTableHeader();
        for (const auto& transaction : transactions) {
            if (toLower(transaction.getTitle()).find(needle) != string::npos) {
                transaction.display();
                found = true;
            }
        }
        if (!found) cout << "No transactions found.\n";
    }

    void filterCategory(const string& category) const {
        const string wanted = toLower(category);
        bool found = false;
        printTableHeader();
        for (const auto& transaction : transactions) {
            if (toLower(transaction.getCategory()) == wanted) {
                transaction.display();
                found = true;
            }
        }
        if (!found) cout << "No transactions found in this category.\n";
    }

    void dashboard() const {
        cout << "\n============= PENNYWISE DASHBOARD =============\n";
        cout << "Monthly budget : Rs. " << fixed << setprecision(2) << monthlyBudget << "\n";
        cout << "Spent          : Rs. " << total() << "\n";
        cout << "Remaining      : Rs. " << monthlyBudget - total() << "\n";
        cout << "Transactions   : " << transactions.size() << "\n";
        const double used = monthlyBudget > 0 ? (total() / monthlyBudget) * 100.0 : 0.0;
        cout << "Budget used    : " << setprecision(1) << used << "%\n";
        cout << "Insight        : You are " << (total() <= monthlyBudget ? "within" : "over")
             << " budget.\n";

        map<string, double> categoryTotals;
        for (const auto& transaction : transactions) {
            categoryTotals[transaction.getCategory()] += transaction.getAmount();
        }
        if (!categoryTotals.empty()) {
            auto highest = max_element(categoryTotals.begin(), categoryTotals.end(),
                [](const pair<string, double>& a, const pair<string, double>& b) {
                    return a.second < b.second;
                });
            cout << "Highest category: " << highest->first << " (Rs. "
                 << fixed << setprecision(2) << highest->second << ")\n";
        }
    }

    void reports() const {
        SummaryReport summary;
        CategoryReport category;
        vector<unique_ptr<Report> > reports;
        reports.emplace_back(new SummaryReport(summary));
        reports.emplace_back(new CategoryReport(category));
        for (const auto& report : reports) report->generate(transactions, monthlyBudget);
    }

    void loadDemoData() {
        add("Groceries", 850, "Food", "2026-11-01", "Monthly groceries");
        add("Bus Pass", 120, "Transport", "2026-11-02", "Local bus pass");
        add("Coffee", 180, "Food", "2026-11-03", "Coffee with friends");
        add("Mobile Recharge", 299, "Bills", "2026-11-04", "Phone recharge");
        add("Textbooks", 650, "Education", "2026-11-05", "Course textbooks");
        add("Lunch", 240, "Food", "2026-11-06", "Lunch");
        add("Movie Tickets", 450, "Entertainment", "2026-11-07", "Weekend movie");
        add("Electricity Bill", 1200, "Bills", "2026-11-08", "Electricity bill");
        add("Cab Ride", 320, "Transport", "2026-11-09", "Ride to campus");
        add("Groceries", 1100, "Food", "2026-11-10", "Weekly groceries");
        add("Coffee", 160, "Food", "2026-11-11", "Coffee");
        add("Medicine", 375, "Health", "2026-11-12", "Medicine");
    }
};

static string readLine(const string& prompt) {
    cout << prompt;
    string value;
    getline(cin, value);
    return value;
}

static double readAmount(const string& prompt) {
    while (true) {
        string input = readLine(prompt);
        stringstream stream(input);
        double amount;
        char extra;
        if ((stream >> amount) && !(stream >> extra) && amount > 0) return amount;
        cout << "Please enter a valid amount greater than zero.\n";
    }
}

static int readInteger(const string& prompt) {
    while (true) {
        string input = readLine(prompt);
        stringstream stream(input);
        int value;
        char extra;
        if ((stream >> value) && !(stream >> extra)) return value;
        cout << "Please enter a valid whole number.\n";
    }
}

static void printMenu() {
    cout << "\n=============== PENNYWISE ===============\n"
         << "1. Dashboard\n"
         << "2. View all transactions\n"
         << "3. Add expense\n"
         << "4. Edit expense\n"
         << "5. Delete expense\n"
         << "6. Search by title\n"
         << "7. Filter by category\n"
         << "8. Reports\n"
         << "0. Exit\n"
         << "==========================================\n";
}

int main() {
    ExpenseTracker tracker(10000.0);
    tracker.loadDemoData();

    cout << "Welcome to PennyWise - Personal Expense Tracker\n"
         << "Demo data for November 2026 has been loaded.\n";

    bool running = true;
    while (running) {
        printMenu();
        int choice = readInteger("Enter your choice: ");
        switch (choice) {
        case 1:
            tracker.dashboard();
            break;
        case 2:
            tracker.showAll();
            break;
        case 3: {
            string title = readLine("Expense title: ");
            if (title.empty()) { cout << "Title cannot be empty.\n"; break; }
            double amount = readAmount("Amount (Rs.): ");
            string category = readLine("Category: ");
            string date = readLine("Date (YYYY-MM-DD): ");
            string notes = readLine("Notes (optional): ");
            if (category.empty() || date.empty()) {
                cout << "Category and date cannot be empty; expense was not added.\n";
                break;
            }
            tracker.add(title, amount, category, date, notes);
            cout << "Expense added successfully.\n";
            break;
        }
        case 4: {
            tracker.showAll();
            int id = readInteger("Enter the ID to edit: ");
            string title = readLine("New title: ");
            if (title.empty()) { cout << "Title cannot be empty.\n"; break; }
            double amount = readAmount("New amount (Rs.): ");
            string category = readLine("New category: ");
            string date = readLine("New date (YYYY-MM-DD): ");
            string notes = readLine("New notes (optional): ");
            if (tracker.edit(id, title, amount, category, date, notes))
                cout << "Expense updated successfully.\n";
            else
                cout << "No transaction exists with that ID.\n";
            break;
        }
        case 5: {
            tracker.showAll();
            int id = readInteger("Enter the ID to delete: ");
            string confirm = readLine("Are you sure you want to delete it? (y/n): ");
            if (confirm == "y" || confirm == "Y") {
                if (tracker.remove(id)) cout << "Expense deleted successfully.\n";
                else cout << "No transaction exists with that ID.\n";
            } else {
                cout << "Delete cancelled.\n";
            }
            break;
        }
        case 6: {
            string query = readLine("Enter title or part of title to search: ");
            tracker.search(query);
            break;
        }
        case 7: {
            string category = readLine("Enter category (e.g. Food): ");
            tracker.filterCategory(category);
            break;
        }
        case 8:
            tracker.reports();
            break;
        case 0:
            running = false;
            cout << "Thank you for using PennyWise. Goodbye!\n";
            break;
        default:
            cout << "Invalid choice. Please select an option from the menu.\n";
        }
    }
    return 0;
}
