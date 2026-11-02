"""
Market Management System
A simple program to manage products, stock, sales and reports for a market shop.
Concepts used: variables, data types, dictionaries, lists, functions,
for loops, while loops, break/continue, file handling, error handling.
"""

import json
from datetime import datetime

DATA_FILE = "market_data.json"

# products: {"rice": {"price": 4500, "stock": 50}, ...}
products = {}
# sales: list of dictionaries, one per sale
sales = []

LOW_STOCK_LEVEL = 5


# ---------- Saving and loading ----------
def save_data():
    with open(DATA_FILE, "w") as f:
        json.dump({"products": products, "sales": sales}, f, indent=4)


def load_data():
    global products, sales
    try:
        with open(DATA_FILE, "r") as f:
            data = json.load(f)
            products = data["products"]
            sales = data["sales"]
        print("Data loaded.")
    except FileNotFoundError:
        print("No saved data found. Starting fresh.")


# ---------- Helpers ----------
def get_number(message, number_type=float):
    """Keep asking until the user types a valid positive number."""
    while True:
        try:
            value = number_type(input(message))
            if value < 0:
                print("Please enter a positive number.")
                continue
            return value
        except ValueError:
            print("Invalid input. Please type a number.")


def format_money(amount):
    """Return an amount as text, e.g. 23500 -> 'UGX 23,500'."""
    return "UGX " + format(int(amount), ",")


def calculate_discount(total):
    """Return the discount for a sale: 5% if total is 50,000 or more, 10% if 100,000 or more."""
    if total >= 100000:
        return total * 0.10
    elif total >= 50000:
        return total * 0.05
    return 0


def show_welcome():
    print("=" * 40)
    print("   WELCOME TO THE MARKET MANAGEMENT SYSTEM")
    print("=" * 40)
    print("Products in shop:", len(products))
    print("Sales recorded:  ", len(sales))


# ---------- Product management ----------
def add_product():
    name = input("Product name: ").strip().lower()
    if name == "":
        print("Name cannot be empty.")
        return
    if name in products:
        print("Product already exists. Use 'Restock' to add more.")
        return
    price = get_number("Price per unit: ")
    stock = get_number("Quantity in stock: ", int)
    products[name] = {"price": price, "stock": stock}
    save_data()
    print(f"{name} added.")


def view_products():
    if not products:
        print("No products yet.")
        return
    print(f"\n{'No.':<5}{'Product':<15}{'Price':<12}{'Stock':<8}")
    print("-" * 40)
    for number, (name, info) in enumerate(products.items(), start=1):
        print(f"{number:<5}{name:<15}{info['price']:<12}{info['stock']:<8}")


def restock_product():
    name = input("Product to restock: ").strip().lower()
    if name not in products:
        print("Product not found.")
        return
    amount = get_number("Quantity to add: ", int)
    products[name]["stock"] += amount
    save_data()
    print(f"{name} now has {products[name]['stock']} in stock.")


def update_price():
    name = input("Product to change price: ").strip().lower()
    if name not in products:
        print("Product not found.")
        return
    products[name]["price"] = get_number("New price: ")
    save_data()
    print("Price updated.")


def delete_product():
    name = input("Product to delete: ").strip().lower()
    if name in products:
        del products[name]
        save_data()
        print(f"{name} deleted.")
    else:
        print("Product not found.")


def search_product():
    keyword = input("Search for: ").strip().lower()
    found = False
    for name, info in products.items():
        if keyword in name:
            print(f"{name} - Price: {info['price']}, Stock: {info['stock']}")
            found = True
    if not found:
        print("No matching product.")


# ---------- Selling ----------
def make_sale():
    if not products:
        print("No products to sell.")
        return

    cart = []
    total = 0

    while True:
        name = input("\nProduct to sell (or 'done' to finish): ").strip().lower()
        if name == "done":
            break
        if name not in products:
            print("Product not found.")
            continue

        available = products[name]["stock"]
        if available == 0:
            print("Out of stock.")
            continue

        qty = get_number(f"Quantity (available {available}): ", int)
        if qty > available:
            print("Not enough stock.")
            continue
        if qty == 0:
            continue

        cost = qty * products[name]["price"]
        products[name]["stock"] -= qty
        cart.append({"product": name, "qty": qty, "cost": cost})
        total += cost
        print(f"Added {qty} x {name} = {cost}")

    if not cart:
        print("Nothing sold.")
        return

    discount = calculate_discount(total)      # function call
    final_total = total - discount
    print_receipt(cart, total, discount, final_total)   # function call

    sales.append({
        "date": datetime.now().strftime("%Y-%m-%d %H:%M"),
        "items": cart,
        "total": final_total,
    })
    save_data()


def print_receipt(cart, total, discount, final_total):
    print("\n" + "=" * 40)
    print("            MARKET RECEIPT")
    print(datetime.now().strftime("          %Y-%m-%d %H:%M"))
    print("=" * 40)
    for item in cart:
        unit_price = item["cost"] / item["qty"]
        print(f"{item['product']:<14}{item['qty']:>3} x {int(unit_price):<7}{format_money(item['cost']):>14}")
    print("-" * 40)
    print(f"{'Subtotal':<26}{format_money(total):>14}")
    if discount > 0:
        print(f"{'Discount':<26}{'-' + format_money(discount):>14}")
    print(f"{'TOTAL':<26}{format_money(final_total):>14}")
    print("=" * 40)


# ---------- Reports ----------
def low_stock_alert():
    print("\nLow stock items:")
    found = False
    for name, info in products.items():
        if info["stock"] <= LOW_STOCK_LEVEL:
            print(f"  - {name}: only {info['stock']} left")
            found = True
    if not found:
        print("  All products are well stocked.")


def sales_report():
    if not sales:
        print("No sales recorded yet.")
        return

    total_income = 0
    product_totals = {}

    for sale in sales:
        total_income += sale["total"]
        for item in sale["items"]:
            name = item["product"]
            product_totals[name] = product_totals.get(name, 0) + item["qty"]

    best = max(product_totals, key=product_totals.get)

    print("\n------ SALES REPORT ------")
    print("Number of sales:", len(sales))
    print("Total income:", format_money(total_income))
    print("Average per sale:", format_money(total_income / len(sales)))
    print("Best selling product:", best, f"({product_totals[best]} units)")
    print("\nUnits sold per product:")
    for name, qty in product_totals.items():
        print(f"  {name}: {qty}")


def stock_value():
    total = 0
    for info in products.values():
        total += info["price"] * info["stock"]
    print("Total value of stock in the shop:", format_money(total))


# ---------- Main menu ----------
def show_menu():
    print("\n===== MARKET MANAGEMENT SYSTEM =====")
    print("1. Add product")
    print("2. View products")
    print("3. Search product")
    print("4. Restock product")
    print("5. Update price")
    print("6. Delete product")
    print("7. Make a sale")
    print("8. Low stock alert")
    print("9. Sales report")
    print("10. Stock value")
    print("0. Exit")


def main():
    load_data()
    show_welcome()          # function call
    actions = {
        "1": add_product,
        "2": view_products,
        "3": search_product,
        "4": restock_product,
        "5": update_price,
        "6": delete_product,
        "7": make_sale,
        "8": low_stock_alert,
        "9": sales_report,
        "10": stock_value,
    }

    while True:
        show_menu()
        choice = input("Choose an option: ").strip()
        if choice == "0":
            save_data()
            print("Data saved. Goodbye!")
            break
        elif choice in actions:
            actions[choice]()
        else:
            print("Invalid choice. Try again.")


if __name__ == "__main__":
    main()
