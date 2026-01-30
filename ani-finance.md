import sqlite3
import os

# Get the path to the desktop
desktop_path = os.path.expanduser("~/Desktop")
db_path = os.path.join(desktop_path, "finance_tracker.db")

# Connect to SQLite database (it will create the database if it doesn't exist)
conn = sqlite3.connect(db_path)
cursor = conn.cursor()

# Create tables for income, expenses, and savings goals
def create_tables():
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS income (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            amount REAL,
            category TEXT,
            description TEXT,
            date TEXT
        )
    ''')

    cursor.execute('''
        CREATE TABLE IF NOT EXISTS expenses (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            amount REAL,
            category TEXT,
            description TEXT,
            date TEXT
        )
    ''')

    cursor.execute('''
        CREATE TABLE IF NOT EXISTS savings_goals (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            goal_name TEXT,
            target_amount REAL,
            current_amount REAL,
            deadline TEXT
        )
    ''')

    conn.commit()

# Function to insert income data
def add_income():
    print("\n--- Add Income ---")
    amount = float(input("Amount: $"))
    category = input("Category (e.g., Allowance, Gift, etc.): ")
    description = input("Description (e.g., From parents, Birthday gift, etc.): ")
    date = input("Date (YYYY-MM-DD): ")

    cursor.execute('''
        INSERT INTO income (amount, category, description, date)
        VALUES (?, ?, ?, ?)
    ''', (amount, category, description, date))

    conn.commit()
    print("Income added successfully!")

# Function to insert expense data
def add_expense():
    print("\n--- Add Expense ---")
    amount = float(input("Amount: $"))
    category = input("Category (e.g., Snacks, Entertainment, etc.): ")
    description = input("Description (e.g., Chips, Movie, etc.): ")
    date = input("Date (YYYY-MM-DD): ")

    cursor.execute('''
        INSERT INTO expenses (amount, category, description, date)
        VALUES (?, ?, ?, ?)
    ''', (amount, category, description, date))

    conn.commit()
    print("Expense added successfully!")

# Function to insert savings goal data
def add_savings_goal():
    print("\n--- Add Savings Goal ---")
    goal_name = input("Goal Name (e.g., Buy a New Toy): ")
    target_amount = float(input("Target Amount: $"))
    current_amount = float(input("Current Amount Saved: $"))
    deadline = input("Deadline (YYYY-MM-DD): ")

    cursor.execute('''
        INSERT INTO savings_goals (goal_name, target_amount, current_amount, deadline)
        VALUES (?, ?, ?, ?)
    ''', (goal_name, target_amount, current_amount, deadline))

    conn.commit()
    print("Savings goal added successfully!")

# Function to view summary
def view_summary():
    print("\n--- View Summary ---")
    
    # Display Income
    cursor.execute("SELECT * FROM income")
    print("\nIncome Records:")
    for row in cursor.fetchall():
        print(f"ID: {row[0]}, Amount: ${row[1]}, Category: {row[2]}, Date: {row[4]}, Description: {row[3]}")
    
    # Display Expenses
    cursor.execute("SELECT * FROM expenses")
    print("\nExpense Records:")
    for row in cursor.fetchall():
        print(f"ID: {row[0]}, Amount: ${row[1]}, Category: {row[2]}, Date: {row[4]}, Description: {row[3]}")
    
    # Display Savings Goals
    cursor.execute("SELECT * FROM savings_goals")
    print("\nSavings Goals:")
    for row in cursor.fetchall():
        print(f"ID: {row[0]}, Goal: {row[1]}, Target: ${row[2]}, Saved: ${row[3]}, Deadline: {row[4]}")

# Main function to run the program
def main():
    create_tables()

    while True:
        print("\n--- Welcome to the Finance Tracker ---")
        print("1. Add Income")
        print("2. Add Expense")
        print("3. Add Savings Goal")
        print("4. View Summary")
        print("5. Exit")

        choice = input("Select an option (1-5): ")

        if choice == "1":
            add_income()
        elif choice == "2":
            add_expense()
        elif choice == "3":
            add_savings_goal()
        elif choice == "4":
            view_summary()
        elif choice == "5":
            print("Goodbye!")
            break
        else:
            print("Invalid choice, please try again.")

# Run the program
if __name__ == "__main__":
    main()

# Close the connection
conn.close()

