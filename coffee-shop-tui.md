import urwid

# Enhanced product list with richer descriptions
products = [
    {
        "id": 1,
        "name": "Single Origin Ethiopian Yirgacheffe",
        "price": 18.99,
        "description": "Light roasted beans with delicate floral notes, bright acidity, and hints of bergamot and jasmine. Perfect for pour-over brewing.",
        "category": "Beans",
        "origin": "Ethiopia",
        "roast": "Light"
    },
    {
        "id": 2,
        "name": "Artisan Dark Roast Blend",
        "price": 16.99,
        "description": "A bold blend of Colombian and Indonesian beans, roasted to perfection with notes of dark chocolate and caramel. Ideal for espresso.",
        "category": "Beans",
        "origin": "Colombia/Indonesia",
        "roast": "Dark"
    },
    {
        "id": 3,
        "name": "Hand-Crafted Ceramic Pour-Over Set",
        "price": 45.00,
        "description": "Elegant ceramic dripper with matching cup. Each piece is uniquely glazed by local artisans. Includes 100 filters.",
        "category": "Equipment",
        "material": "Ceramic",
        "includes": ["Dripper", "Cup", "Filters"]
    },
    {
        "id": 4,
        "name": "Copper Gooseneck Kettle",
        "price": 65.00,
        "description": "Precision-pour copper kettle with thermometer. Perfect flow rate for pour-over brewing. Capacity: 0.9L",
        "category": "Equipment",
        "material": "Copper",
        "capacity": "0.9L"
    }
]

# Initialize an empty cart
cart = []

# Color palette
palette = [
    ('banner', 'black', 'light gray'),
    ('streak', 'black', 'dark red'),
    ('bg', 'black', 'dark blue'),
    ('reversed', 'standout', ''),
    ('button', 'black', 'light gray'),
    ('button_focused', 'white', 'dark blue'),
    ('highlight', 'black', 'brown'),
    ('header', 'white', 'dark blue'),
    ('footer', 'black', 'light gray'),
]

class CoffeeShopUI:
    def __init__(self):
        self.main_loop = None
    
    def create_bordered_box(self, widget):
        return urwid.LineBox(
            urwid.Padding(widget, left=2, right=2),
            title="Artisan Coffee Shop"
        )

    def create_button(self, label, callback):
        button = urwid.Button(label)
        urwid.connect_signal(button, 'click', callback)
        return urwid.AttrMap(button, 'button', focus_map='button_focused')

    def show_about(self, button):
        text = [
            ('header', "\n🌟 Welcome to the Artisan Coffee Shop! 🌟\n\n"),
            "We are passionate about bringing you the finest coffee experiences from around the world. "
            "Our carefully curated selection of beans and equipment represents the pinnacle of coffee craftsmanship.\n\n",
            ('highlight', "🌱 Our Promise:\n"),
            "• Ethically sourced beans\n"
            "• Artisanal roasting\n"
            "• Expert curation\n"
            "• Quality equipment\n\n",
            "Browse our selection and embark on a journey of exceptional coffee moments.",
        ]
        back_button = self.create_button("Back to Main Menu", self.show_main_menu)
        content = urwid.Pile([
            urwid.Text(text),
            urwid.Divider(),
            back_button
        ])
        self.main.original_widget = self.create_bordered_box(urwid.Filler(content))

    def show_shop(self, button):
        body = [
            urwid.Text(('header', "\n📦 Available Products\n")),
            urwid.Divider()
        ]
        
        current_category = None
        for product in products:
            if product['category'] != current_category:
                current_category = product['category']
                body.append(urwid.Text(('highlight', f"\n{current_category}:")))
                body.append(urwid.Divider())
            
            product_widget = urwid.Pile([
                urwid.Text(f"✨ {product['name']} - ${product['price']:.2f}"),
                urwid.Text(f"   {product['description']}", wrap='clip'),
                self.create_button("Add to Cart", lambda b, p=product: self.add_to_cart(b, p)),
                urwid.Divider()
            ])
            body.append(product_widget)
        
        back_button = self.create_button("Back to Main Menu", self.show_main_menu)
        body.append(back_button)
        
        self.main.original_widget = self.create_bordered_box(
            urwid.ListBox(urwid.SimpleFocusListWalker(body))
        )

    def add_to_cart(self, button, product):
        cart.append(product)
        text = urwid.Text([
            ('highlight', "✅ Added to Cart:\n\n"),
            f"{product['name']}\n",
            f"Price: ${product['price']:.2f}\n\n",
            "What would you like to do next?"
        ])
        
        buttons = urwid.Pile([
            self.create_button("Continue Shopping", self.show_shop),
            self.create_button("View Cart", self.show_cart),
            self.create_button("Back to Main Menu", self.show_main_menu)
        ])
        
        content = urwid.Pile([text, urwid.Divider(), buttons])
        self.main.original_widget = self.create_bordered_box(urwid.Filler(content))

    def show_cart(self, button):
        if not cart:
            content = urwid.Pile([
                urwid.Text(('header', "🛒 Your Cart\n")),
                urwid.Text("Your cart is empty."),
                urwid.Divider(),
                self.create_button("Browse Shop", self.show_shop),
                self.create_button("Back to Main Menu", self.show_main_menu)
            ])
            self.main.original_widget = self.create_bordered_box(urwid.Filler(content))
            return

        body = [urwid.Text(('header', "🛒 Your Cart\n")), urwid.Divider()]
        total = 0
        
        for item in cart:
            item_widget = urwid.Columns([
                ('weight', 8, urwid.Text(item['name'])),
                ('weight', 2, urwid.Text(f"${item['price']:.2f}"))
            ])
            body.append(item_widget)
            total += item['price']
        
        body.extend([
            urwid.Divider(),
            urwid.Text(('highlight', f"Total: ${total:.2f}")),
            urwid.Divider(),
            self.create_button("Proceed to Checkout", self.show_checkout),
            self.create_button("Continue Shopping", self.show_shop),
            self.create_button("Back to Main Menu", self.show_main_menu)
        ])
        
        self.main.original_widget = self.create_bordered_box(
            urwid.ListBox(urwid.SimpleFocusListWalker(body))
        )

    def show_checkout(self, button):
        if not cart:
            return self.show_cart(button)
        
        total = sum(item['price'] for item in cart)
        content = urwid.Pile([
            urwid.Text(('header', "💳 Checkout\n")),
            urwid.Text(f"Order Total: ${total:.2f}\n"),
            urwid.Text("Please select your payment method:"),
            urwid.Divider(),
            self.create_button("Credit Card", self.process_payment),
            self.create_button("PayPal", self.process_payment),
            urwid.Divider(),
            self.create_button("Back to Cart", self.show_cart),
            self.create_button("Back to Main Menu", self.show_main_menu)
        ])
        
        self.main.original_widget = self.create_bordered_box(urwid.Filler(content))

    def process_payment(self, button):
        content = urwid.Pile([
            urwid.Text(('highlight', "🎉 Thank you for your order!\n")),
            urwid.Text("Your order has been processed successfully.\n"),
            urwid.Text("You will receive a confirmation email shortly.\n"),
            urwid.Divider(),
            self.create_button("Back to Main Menu", self.show_main_menu)
        ])
        cart.clear()
        self.main.original_widget = self.create_bordered_box(urwid.Filler(content))

    def exit_program(self, button):
        raise urwid.ExitMainLoop()

    def show_main_menu(self, button=None):
        menu_items = [
            ("📖 About", self.show_about),
            ("🛍️  Shop", self.show_shop),
            ("🛒 Cart", self.show_cart),
            ("💳 Checkout", self.show_checkout),
            ("❌ Exit", self.exit_program)
        ]
        
        body = [
            urwid.Text(('header', "\n☕ Welcome to Artisan Coffee Shop ☕\n")),
            urwid.Divider()
        ]
        
        for name, callback in menu_items:
            body.append(self.create_button(name, callback))
        
        menu = urwid.ListBox(urwid.SimpleFocusListWalker(body))
        self.main.original_widget = self.create_bordered_box(menu)

    def run(self):
        # Create main widget
        self.main = urwid.Padding(urwid.Filler(urwid.Text("")), left=2, right=2)
        
        # Show the main menu
        self.show_main_menu()
        
        # Create the overlay
        top = urwid.Overlay(
            self.main, urwid.SolidFill(" "),
            align="center", width=('relative', 80),
            valign="middle", height=('relative', 80),
            min_width=40, min_height=20
        )
        
        # Start the main loop
        self.main_loop = urwid.MainLoop(top, palette=palette)
        self.main_loop.run()

if __name__ == "__main__":
    app = CoffeeShopUI()
    app.run()
