import urwid

# Sample product list
products = [
    {"id": 1, "name": "Coffee Bag", "price": 15.00, "description": "A bag of premium coffee beans."},
    {"id": 2, "name": "Espresso Cup", "price": 5.00, "description": "A ceramic espresso cup."},
]

# Initialize an empty cart
cart = []

def menu(title, options):
    body = [urwid.Text(title), urwid.Divider()]
    for name, callback in options:
        button = urwid.Button(name)
        urwid.connect_signal(button, 'click', callback)
        body.append(urwid.AttrMap(button, None, focus_map='reversed'))
    return urwid.ListBox(urwid.SimpleFocusListWalker(body))

def show_about(button):
    text = "Welcome to the Coffee Shop CLI!\n\nThis is a TUI-based shop where you can buy coffee and related products."
    main.original_widget = urwid.Padding(urwid.Text(text), left=2, right=2)

def show_shop(button):
    body = [urwid.Text("Available Products:"), urwid.Divider()]
    for product in products:
        button = urwid.Button(f"{product['name']} - ${product['price']:.2f}")
        urwid.connect_signal(button, 'click', add_to_cart, product)
        body.append(urwid.AttrMap(button, None, focus_map='reversed'))
    main.original_widget = urwid.ListBox(urwid.SimpleFocusListWalker(body))

def add_to_cart(button, product):
    cart.append(product)
    main.original_widget = urwid.Text(f"Added {product['name']} to the cart.")

def show_cart(button):
    if not cart:
        main.original_widget = urwid.Text("Your cart is empty.")
        return
    body = [urwid.Text("Your Cart:"), urwid.Divider()]
    total = 0
    for item in cart:
        body.append(urwid.Text(f"{item['name']} - ${item['price']:.2f}"))
        total += item['price']
    body.append(urwid.Divider())
    body.append(urwid.Text(f"Total: ${total:.2f}"))
    checkout_button = urwid.Button("Proceed to Checkout")
    urwid.connect_signal(checkout_button, 'click', show_checkout)
    body.append(urwid.AttrMap(checkout_button, None, focus_map='reversed'))
    main.original_widget = urwid.ListBox(urwid.SimpleFocusListWalker(body))

def show_checkout(button):
    main.original_widget = urwid.Text(
        "Checkout page\n\nHere, you could integrate with Stripe for payment."
    )

def exit_program(button):
    raise urwid.ExitMainLoop()

menu_top = menu("Coffee Shop CLI", [
    ("About", show_about),
    ("Shop", show_shop),
    ("Cart", show_cart),
    ("Checkout", show_checkout),
    ("Quit", exit_program),
])

main = urwid.Padding(menu_top, left=2, right=2)
top = urwid.Overlay(main, urwid.SolidFill("\N{MEDIUM SHADE}"),
                    align="center", width=("relative", 60),
                    valign="middle", height=("relative", 60),
                    min_width=20, min_height=9)

if __name__ == "__main__":
    urwid.MainLoop(top, palette=[("reversed", "standout", "")]).run()

