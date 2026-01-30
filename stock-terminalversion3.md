import time
import random
import curses
from datetime import datetime, timedelta

class Stock:
    def __init__(self, symbol, price, history=None):
        self.symbol = symbol
        self.price = price
        self.prev_price = price
        self.history = {
            '1D': history or [price] * 24,
            '7D': history or [price] * 7,
            '30D': history or [price] * 30,
            '1Y': history or [price] * 12
        }
        
    def update(self):
        self.prev_price = self.price
        change = random.gauss(0, self.price * 0.002)
        self.price = max(0.01, self.price + change)
        
        self.history['1D'].append(self.price)
        if len(self.history['1D']) > 24:
            self.history['1D'].pop(0)
            
        if random.random() < 0.1:
            for timeframe in ['7D', '30D', '1Y']:
                self.history[timeframe].append(self.price)
                max_length = {'7D': 7, '30D': 30, '1Y': 12}[timeframe]
                if len(self.history[timeframe]) > max_length:
                    self.history[timeframe].pop(0)

def generate_chart(values, width=50, height=8):
    if not values:
        return []
    
    min_val = min(values)
    max_val = max(values)
    value_range = max_val - min_val if max_val != min_val else 1
    
    chart = []
    for y in range(height-1, -1, -1):
        row = []
        for x in range(min(width, len(values))):
            normalized_val = (values[x] - min_val) / value_range
            if normalized_val * height >= y:
                row.append('█')
            else:
                row.append(' ')
        chart.append(''.join(row))
    return chart

def safe_addstr(stdscr, y, x, string):
    """Safely add a string to the screen, truncating if necessary."""
    height, width = stdscr.getmaxyx()
    if y < height:
        if x < width:
            try:
                # Truncate the string if it would exceed screen width
                max_length = width - x
                truncated_string = string[:max_length]
                stdscr.addstr(y, x, truncated_string)
            except curses.error:
                pass

def draw_ui(stdscr, stocks, portfolio_value, selected_stock_idx, selected_timeframe, input_mode, input_buffer):
    stdscr.clear()
    height, width = stdscr.getmaxyx()
    
    # Ensure minimum terminal size
    if height < 20 or width < 80:
        safe_addstr(stdscr, 0, 0, "Terminal too small. Please resize to at least 80x20")
        stdscr.refresh()
        return
    
    # Calculate dimensions
    chart_height = min(8, height - 12)  # Adjust chart height based on terminal size
    
    # Header
    safe_addstr(stdscr, 0, 0, "╔" + "═" * (width-2) + "╗")
    safe_addstr(stdscr, 1, 0, "║" + "STOCK MARKET TRACKER".center(width-2) + "║")
    safe_addstr(stdscr, 2, 0, "╠" + "═" * (width-2) + "╣")
    
    portfolio_text = f"Portfolio Value: ${portfolio_value:.2f}"
    safe_addstr(stdscr, 3, 0, "║" + portfolio_text.center(width-2) + "║")
    
    # Timeframe selector
    timeframes = ["1D", "7D", "30D", "1Y"]
    timeframe_text = " | ".join([
        f"[{'*' if tf == selected_timeframe else ' '}{tf}]" for tf in timeframes
    ])
    safe_addstr(stdscr, 4, 0, "╠" + "═" * (width-2) + "╣")
    safe_addstr(stdscr, 5, 0, "║" + timeframe_text.center(width-2) + "║")
    
    # Chart
    if selected_stock_idx < len(stocks):
        featured_stock = stocks[selected_stock_idx]
        chart_title = f"{featured_stock.symbol} - {selected_timeframe} View"
        safe_addstr(stdscr, 6, 0, "╠" + "═" * (width-2) + "╣")
        safe_addstr(stdscr, 7, 0, "║" + chart_title.center(width-2) + "║")
        
        chart = generate_chart(featured_stock.history[selected_timeframe], width-6, chart_height)
        for i, line in enumerate(chart, 8):
            safe_addstr(stdscr, i, 0, f"║ {line:<{width-4}} ║")
    
    # Stock list
    list_start = 8 + chart_height
    safe_addstr(stdscr, list_start, 0, "╠" + "═" * (width-2) + "╣")
    header = "Symbol    Price          Change         24h High         24h Low"
    safe_addstr(stdscr, list_start + 1, 0, "║" + header.center(width-2) + "║")
    safe_addstr(stdscr, list_start + 2, 0, "╠" + "═" * (width-2) + "╣")
    
    # Calculate maximum visible stocks based on remaining space
    max_visible_stocks = height - list_start - 6  # Reserve space for footer
    visible_stocks = stocks[:max_visible_stocks]
    
    for i, stock in enumerate(visible_stocks):
        change = stock.price - stock.prev_price
        change_pct = (change / stock.prev_price) * 100
        change_text = f"{change:+.2f} ({change_pct:+.2f}%)"
        high = max(stock.history['1D'])
        low = min(stock.history['1D'])
        
        stock_line = f"{stock.symbol:<8} {format_price(stock.price):<14} {change_text:<15} {format_price(high):<15} {format_price(low):<15}"
        if i == selected_stock_idx:
            stock_line = f"> {stock_line}"
        else:
            stock_line = f"  {stock_line}"
        safe_addstr(stdscr, list_start + 3 + i, 0, "║" + stock_line.ljust(width-2) + "║")
    
    # Input area
    input_start = min(list_start + 3 + len(visible_stocks), height - 3)
    safe_addstr(stdscr, input_start, 0, "╠" + "═" * (width-2) + "╣")
    if input_mode:
        prompt = "Enter stock symbol: " + input_buffer
        safe_addstr(stdscr, input_start + 1, 0, "║" + prompt.ljust(width-2) + "║")
    else:
        controls = "Controls: ↑/↓: Select Stock | ←/→: Change Timeframe | A: Add Stock | Q: Quit"
        safe_addstr(stdscr, input_start + 1, 0, "║" + controls.center(width-2) + "║")
    
    safe_addstr(stdscr, input_start + 2, 0, "╚" + "═" * (width-2) + "╝")
    
    stdscr.refresh()

def format_price(price):
    return f"${price:.2f}"

def main(stdscr):
    # Setup
    curses.curs_set(0)
    curses.start_color()
    curses.use_default_colors()
    
    # Initialize stocks
    stocks = [
        Stock("AAPL", 175.0),
        Stock("GOOGL", 140.0),
        Stock("MSFT", 380.0),
        Stock("AMZN", 178.0),
        Stock("TSLA", 238.0)
    ]
    
    selected_stock_idx = 0
    selected_timeframe = "1D"
    input_mode = False
    input_buffer = ""
    
    while True:
        portfolio_value = sum(stock.price * 10 for stock in stocks)
        
        for stock in stocks:
            stock.update()
        
        draw_ui(stdscr, stocks, portfolio_value, selected_stock_idx, selected_timeframe, 
                input_mode, input_buffer)
        
        try:
            key = stdscr.getch()
            if input_mode:
                if key == ord('\n'):
                    if input_buffer:
                        stocks.append(Stock(input_buffer.upper(), random.uniform(10, 1000)))
                    input_mode = False
                    input_buffer = ""
                elif key == 27:  # Escape
                    input_mode = False
                    input_buffer = ""
                elif key == curses.KEY_BACKSPACE or key == 127:
                    input_buffer = input_buffer[:-1]
                elif len(input_buffer) < 5 and chr(key).isalnum():
                    input_buffer += chr(key).upper()
            else:
                if key == ord('q'):
                    break
                elif key == ord('a'):
                    input_mode = True
                elif key == curses.KEY_UP and selected_stock_idx > 0:
                    selected_stock_idx -= 1
                elif key == curses.KEY_DOWN and selected_stock_idx < len(stocks) - 1:
                    selected_stock_idx += 1
                elif key == curses.KEY_LEFT:
                    timeframes = ["1D", "7D", "30D", "1Y"]
                    idx = timeframes.index(selected_timeframe)
                    selected_timeframe = timeframes[idx - 1] if idx > 0 else timeframes[-1]
                elif key == curses.KEY_RIGHT:
                    timeframes = ["1D", "7D", "30D", "1Y"]
                    idx = timeframes.index(selected_timeframe)
                    selected_timeframe = timeframes[(idx + 1) % len(timeframes)]
        
        except curses.error:
            pass
            
        time.sleep(0.1)

if __name__ == "__main__":
    try:
        curses.wrapper(main)
    except KeyboardInterrupt:
        pass
