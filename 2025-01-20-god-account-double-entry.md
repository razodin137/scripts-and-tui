import csv
from datetime import datetime
import requests

# Function to calculate the current price in THB
def convert_to_thb(amount, currency="USD"):
    try:
        response = requests.get(f"https://api.exchangerate-api.com/v4/latest/{currency}")
        data = response.json()
        rate = data["rates"].get("THB", None)
        if rate:
            return round(amount * rate, 2)
        else:
            return "Rate Unavailable"
    except Exception as e:
        return f"Error: {str(e)}"

# Function to add a double-entry record to the CSV
def add_double_entry_to_csv(filename, amount, category, account, description, scriptural_basis, currency="USD"):
    date = datetime.now().strftime("%Y-%m-%d")
    ten_percent = round(amount * 0.1, 2)
    thb_value = convert_to_thb(amount, currency)
    
    # Prepare double-entry rows
    debit_entry = {
        "Date": date,
        "Amount": amount,
        "Account": category,  # Debit side: Category as the account (e.g., "Donation")
        "Debit/Credit": "Debit",
        "10% Calculation": ten_percent,
        "THB Value": thb_value,
        "Description": description,
        "Scriptural Basis": scriptural_basis
    }
    credit_entry = {
        "Date": date,
        "Amount": amount,
        "Account": account,  # Credit side: Source account (e.g., "Bank")
        "Debit/Credit": "Credit",
        "10% Calculation": ten_percent,
        "THB Value": thb_value,
        "Description": description,
        "Scriptural Basis": scriptural_basis
    }
    
    # Write to CSV
    try:
        with open(filename, mode='a', newline='', encoding='utf-8') as file:
            fieldnames = ["Date", "Amount", "Account", "Debit/Credit", "10% Calculation", "THB Value", "Description", "Scriptural Basis"]
            writer = csv.DictWriter(file, fieldnames=fieldnames)
            
            # Add header only if the file is new
            if file.tell() == 0:
                writer.writeheader()
            
            # Write both debit and credit entries
            writer.writerow(debit_entry)
            writer.writerow(credit_entry)
        
        print("Transaction recorded successfully!")
    except Exception as e:
        print(f"Error writing to CSV: {str(e)}")

# Example Usage
if __name__ == "__main__":
    filename = "double_entry_bookkeeping.csv"
    amount = float(input("Enter the amount: "))
    category = input("Enter the category (e.g., Donation, Business, etc.): ")
    account = input("Enter the account (e.g., Bank, Cash, etc.): ")
    description = input("Enter the description: ")
    scriptural_basis = input("Enter the scriptural basis: ")
    
    add_double_entry_to_csv(filename, amount, category, account, description, scriptural_basis)

