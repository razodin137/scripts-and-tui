import urwid
import random

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

# Simplified color palette
palette = [
    ('button_focused', 'white', 'dark blue'),
    ('highlight', 'black', 'brown'),
    ('easter_egg', 'light magenta', 'black'),
]

class CoffeeShopUI:
    def __init__(self):
        self.main_loop = None
        self.konami_code = ['up', 'up', 'down', 'down', 'left', 'right', 'left', 'right', 'b', 'a']
        self.current_input = []
        
    def handle_input(self, key):
        if key in ('up', 'down', 'left', 'right', 'b', 'a'):
            self.current_input.append(key)
            if len(self.current_input) > len(self.konami_code):
                self.current_input.pop(0)
            if self.current_input == self.konami_code:
                self.show_easter_egg(None)
        return key
    
    def show_easter_egg(self, button):
        ascii_art = """
        ☕️ SECRET MENU UNLOCKED! ☕️
             
             )  (
            (   ) )
             ) ( (
           _______)_
        .-'---------|  
       ( C|/\\/\\/\\/\\/|
        '-./\\/\\/\\/\\/|
          '_________'
           '-------'
        """
        secret_products = [
            "🌟 Rainbow Sparkle Latte - $4.20",
            "🌈 Unicorn Bean Blend - $42.00",
            "🎮 Gamer Fuel Cold Brew - $13.37"
        ]
        
        content = urwid.Pile([
            urwid.Text(('easter_egg', ascii_art)),
            urwid.Divider(),
            urwid.Text(('easter_egg', "\nCongratulations! You've discovered our secret menu:\n")),
            *[urwid.Text(('easter_egg', item)) for item in secret_products],
            urwid.Divider(),
            self.create_centered_button("Back to Reality", self.show_main_menu)
        ])
        
        self.main.original_widget = urwid.Filler(content, 'middle')

    def create_centered_button(self, label, callback):
        button = urwid.Button(label)
        urwid.connect_signal(button, 'click', callback)
        return urwid.Padding(
            urwid.AttrMap(button, None, focus_map='button_focused'),
            align='center',
            width=('relative', 80)
        )

    def create_centered_text(self, text, attr=None):
        return urwid.Padding(
            urwid.Text(text, align='center'),
            align='center',
            width=('relative', 80)
        )

    def show_about(self, button):
        text = [
            "☕️ Welcome to the Artisan Coffee Shop! ☕️\n\n",
            "We are passionate about bringing you the finest coffee experiences\n"
            "from around the world.\n\n",
            "🌱 Our Promise:\n",
            "• Ethically sourced beans\n"
            "• Artisanal roasting\n"
            "• Expert curation\n"
            "• Quality equipment\n\n",
            "Browse our selection and embark on a journey of exceptional coffee moments.",
        ]
        
        content = urwid.Pile([
            self.create_centered_text(line) for line in text
        ] + [
            urwid.Divider(),
            self.create_centered_button("Back to Main Menu", self.show_main_menu)
        ])
        
        self.main.original_widget = urwid.Filler(content, 'middle')

    def show_shop(self, button):
        items = [self.create_centered_text("📦 Available Products\n")]
        
        current_category = None
        for product in products:
            if product['category'] != current_category:
                current_category = product['category']
                items.extend([
                    urwid.Divider(),
                    self.create_centered_text(f"\n{current_category}:")
                ])
            
            items.extend([
                self.create_centered_text(f"✨ {product['name']} - ${product['price']:.2f}"),
                self.create_centered_text(f"{product['description']}", wrap='clip'),
                self.create_centered_button("Add to Cart", lambda b, p=product: self.add_to_cart(b, p)),
                urwid.Divider()
            ])
        
        items.append(self.create_centered_button("Back to Main Menu", self.show_main_menu))
        
        self.main.original_widget = urwid.ListBox(urwid.SimpleFocusListWalker(items))

    def add_to_cart(self, button, product):
        cart.append(product)
        
        content = urwid.Pile([
            self.create_centered_text("✅ Added to Cart:\n"),
            self.create_centered_text(f"{product['name']}"),
            self.create_centered_text(f"Price: ${product['price']:.2f}\n"),
            self.create_centered_text("What would you like to do next?\n"),
            urwid.Divider(),
            self.create_centered_button("Continue Shopping", self.show_shop),
            self.create_centered_button("View Cart", self.show_cart),
            self.create_centered_button("Back to Main Menu", self.show_main_menu)
        ])
        
        self.main.original_widget = urwid.Filler(content, 'middle')

    def show_cart(self, button):
        if not cart:
            content = urwid.Pile([
                self.create_centered_text("🛒 Your Cart\n"),
                self.create_centered_text("Your cart is empty."),
                urwid.Divider(),
                self.create_centered_button("Browse Shop", self.show_shop),
                self.create_centered_button("Back to Main Menu", self.show_main_menu)
            ])
            self.main.original_widget = urwid.Filler(content, 'middle')
            return

        items = [self.create_centered_text("🛒 Your Cart\n")]
        total = 0
        
        for item in cart:
            items.extend([
                self.create_centered_text(f"{item['name']} - ${item['price']:.2f}")
            ])
            total += item['price']
        
        items.extend([
            urwid.Divider(),
            self.create_centered_text(f"Total: ${total:.2f}"),
            urwid.Divider(),
            self.create_centered_button("Proceed to Checkout", self.show_checkout),
            self.create_centered_button("Continue Shopping", self.show_shop),
            self.create_centered_button("Back to Main Menu", self.show_main_menu)
        ])
        
        self.main.original_widget = urwid.ListBox(urwid.SimpleFocusListWalker(items))

    def show_checkout(self, button):
        if not cart:
            return self.show_cart(button)
        
        total = sum(item['price'] for item in cart)
        content = urwid.Pile([
            self.create_centered_text("💳 Checkout\n"),
            self.create_centered_text(f"Order Total: ${total:.2f}\n"),
            self.create_centered_text("Please select your payment method:"),
            urwid.Divider(),
            self.create_centered_button("Credit Card", self.process_payment),
            self.create_centered_button("PayPal", self.process_payment),
            urwid.Divider(),
            self.create_centered_button("Back to Cart", self.show_cart),
            self.create_centered_button("Back to Main Menu", self.show_main_menu)
        ])
        
        self.main.original_widget = urwid.Filler(content, 'middle')

    def process_payment(self, button):
        content = urwid.Pile([
            self.create_centered_text("🎉 Thank you for your order!\n"),
            self.create_centered_text("Your order has been processed successfully.\n"),
            self.create_centered_text("You will receive a confirmation email shortly.\n"),
            urwid.Divider(),
            self.create_centered_button("Back to Main Menu", self.show_main_menu)
        ])
        cart.clear()
        self.main.original_widget = urwid.Filler(content, 'middle')

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
        
        content = [
            self.create_centered_text("☕ Welcome to Artisan Coffee Shop ☕\n"),
            urwid.Divider()
        ]
        
        for name, callback in menu_items:
            content.append(self.create_centered_button(name, callback))
        
        self.main.original_widget = urwid.Filler(urwid.Pile(content), 'middle')

    def run(self):
        # Create main widget - using WidgetPlaceholder instead of Widget
        self.main = urwid.WidgetPlaceholder(urwid.SolidFill(" "))
        
        # Show the main menu
        self.show_main_menu()
        
        # Create the overlay
        top = urwid.Overlay(
            self.main, urwid.SolidFill(" "),
            align="center", width=('relative', 60),
            valign="middle", height=('relative', 60),
            min_width=40, min_height=20
        )
        
        # Start the main loop with input handling
        self.main_loop = urwid.MainLoop(
            top,
            palette=palette,
            unhandled_input=self.handle_input
        )
        self.main_loop.run()

if __name__ == "__main__":
    app = CoffeeShopUI()
    app.run()
