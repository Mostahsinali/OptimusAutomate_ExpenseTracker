# OptimusAutomate_ExpenseTracker
"""
TASK 1: Expense Tracker CLI App
Optimus Automate — Python Programming Internship
"""

import json
import os
import sqlite3
from datetime import datetime

DB_FILE = "expenses.db"

# ─── DATABASE SETUP ────────────────────────────────────────────────────────────

def init_db():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("""
        CREATE TABLE IF NOT EXISTS expenses (
            id       INTEGER PRIMARY KEY AUTOINCREMENT,
            date     TEXT    NOT NULL,
            category TEXT    NOT NULL,
            amount   REAL    NOT NULL,
            note     TEXT
        )
    """)
    conn.commit()
    conn.close()

def get_conn():
    return sqlite3.connect(DB_FILE)

# ─── CORE FUNCTIONS ────────────────────────────────────────────────────────────

CATEGORIES = [
    "Food", "Transport", "Utilities", "Entertainment",
    "Health", "Shopping", "Education", "Other"
]

def add_expense():
    print("\n── Add Expense ──────────────────────")
    
    # Date
    date_input = input("Date (YYYY-MM-DD) [leave blank for today]: ").strip()
    if not date_input:
        date_input = datetime.now().strftime("%Y-%m-%d")
    else:
        try:
            datetime.strptime(date_input, "%Y-%m-%d")
        except ValueError:
            print("❌  Invalid date format. Use YYYY-MM-DD.")
            return

    # Category
    print("\nCategories:")
    for i, cat in enumerate(CATEGORIES, 1):
        print(f"  {i}. {cat}")
    cat_input = input("Choose category number: ").strip()
    if not cat_input.isdigit() or not (1 <= int(cat_input) <= len(CATEGORIES)):
        print("❌  Invalid category.")
        return
    category = CATEGORIES[int(cat_input) - 1]

    # Amount
    amount_input = input("Amount (PKR): ").strip()
    try:
        amount = float(amount_input)
        if amount <= 0:
            raise ValueError
    except ValueError:
        print("❌  Amount must be a positive number.")
        return

    # Note
    note = input("Note (optional): ").strip()

    conn = get_conn()
    conn.execute(
        "INSERT INTO expenses (date, category, amount, note) VALUES (?, ?, ?, ?)",
        (date_input, category, amount, note)
    )
    conn.commit()
    conn.close()
    print(f"✅  Expense of PKR {amount:.2f} added under '{category}'.")


def view_expenses():
    print("\n── All Expenses ─────────────────────")
    conn = get_conn()
    rows = conn.execute(
        "SELECT id, date, category, amount, note FROM expenses ORDER BY date DESC"
    ).fetchall()
    conn.close()

    if not rows:
        print("No expenses found.")
        return

    print(f"{'ID':<5} {'Date':<12} {'Category':<15} {'Amount (PKR)':<14} Note")
    print("─" * 65)
    for row in rows:
        note = row[4] if row[4] else "—"
        print(f"{row[0]:<5} {row[1]:<12} {row[2]:<15} {row[3]:<14.2f} {note}")


def delete_expense():
    print("\n── Delete Expense ───────────────────")
    view_expenses()
    id_input = input("\nEnter ID to delete (or 0 to cancel): ").strip()
    if not id_input.isdigit():
        print("❌  Invalid ID.")
        return
    exp_id = int(id_input)
    if exp_id == 0:
        return

    conn = get_conn()
    row = conn.execute("SELECT id FROM expenses WHERE id = ?", (exp_id,)).fetchone()
    if not row:
        print("❌  No expense found with that ID.")
        conn.close()
        return
    conn.execute("DELETE FROM expenses WHERE id = ?", (exp_id,))
    conn.commit()
    conn.close()
    print(f"✅  Expense #{exp_id} deleted.")


def monthly_summary():
    print("\n── Monthly Summary ──────────────────")
    month_input = input("Enter month (YYYY-MM) [leave blank for current month]: ").strip()
    if not month_input:
        month_input = datetime.now().strftime("%Y-%m")
    else:
        try:
            datetime.strptime(month_input, "%Y-%m")
        except ValueError:
            print("❌  Invalid format. Use YYYY-MM.")
            return

    conn = get_conn()
    rows = conn.execute(
        """
        SELECT category, SUM(amount)
        FROM expenses
        WHERE strftime('%Y-%m', date) = ?
        GROUP BY category
        ORDER BY SUM(amount) DESC
        """,
        (month_input,)
    ).fetchall()
    total = conn.execute(
        "SELECT SUM(amount) FROM expenses WHERE strftime('%Y-%m', date) = ?",
        (month_input,)
    ).fetchone()[0] or 0
    conn.close()

    print(f"\nSummary for {month_input}")
    print("─" * 35)
    if not rows:
        print("No expenses this month.")
        return
    for cat, amt in rows:
        bar = "█" * int((amt / total) * 20)
        print(f"  {cat:<15} PKR {amt:>8.2f}  {bar}")
    print("─" * 35)
    print(f"  {'TOTAL':<15} PKR {total:>8.2f}")


def export_to_json():
    conn = get_conn()
    rows = conn.execute(
        "SELECT id, date, category, amount, note FROM expenses ORDER BY date"
    ).fetchall()
    conn.close()

    data = [
        {"id": r[0], "date": r[1], "category": r[2], "amount": r[3], "note": r[4]}
        for r in rows
    ]
    with open("expenses_export.json", "w") as f:
        json.dump(data, f, indent=2)
    print(f"✅  {len(data)} expenses exported to expenses_export.json")


# ─── MENU ──────────────────────────────────────────────────────────────────────

def main():
    init_db()
    print("\n╔══════════════════════════════╗")
    print("║    💰 Expense Tracker CLI    ║")
    print("╚══════════════════════════════╝")

    menu = {
        "1": ("Add Expense",       add_expense),
        "2": ("View All Expenses", view_expenses),
        "3": ("Delete Expense",    delete_expense),
        "4": ("Monthly Summary",   monthly_summary),
        "5": ("Export to JSON",    export_to_json),
        "0": ("Exit",              None),
    }

    while True:
        print("\n── Menu ─────────────────────────────")
        for key, (label, _) in menu.items():
            print(f"  {key}. {label}")
        choice = input("\nChoose option: ").strip()

        if choice == "0":
            print("Goodbye! 👋")
            break
        elif choice in menu:
            _, func = menu[choice]
            func()
        else:
            print("❌  Invalid option. Try again.")


if __name__ == "__main__":
    main()
