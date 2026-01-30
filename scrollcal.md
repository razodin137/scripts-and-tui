import curses
from datetime import datetime, timedelta
import calendar

class CalendarScroll:
    def __init__(self, stdscr):
        self.stdscr = stdscr
        self.current_date = datetime.now()
        self.scroll_position = 0
        self.selected_row = 0
        self.selected_col = self.current_date.weekday()
        self.day_width = 10  # Width of each day block
        self.day_height = 5  # Height of each day block
        
        # Setup colors
        curses.start_color()
        curses.init_pair(1, curses.COLOR_BLACK, curses.COLOR_WHITE)  # Selected day
        curses.init_pair(2, curses.COLOR_YELLOW, curses.COLOR_BLACK)  # Current day
        curses.init_pair(3, curses.COLOR_CYAN, curses.COLOR_BLACK)    # Month separator

    def draw_day_block(self, date, y_pos, x_pos, is_selected):
        """Draw a single day block"""
        is_current = date.date() == datetime.now().date()
        style = curses.color_pair(1) if is_selected else curses.A_NORMAL
        if is_current:
            style = curses.color_pair(2)

        # Draw box borders
        self.stdscr.addstr(y_pos, x_pos, "┌" + "─"*(self.day_width-2) + "┐", style)
        self.stdscr.addstr(y_pos + self.day_height - 1, x_pos, 
                          "└" + "─"*(self.day_width-2) + "┘", style)
        
        # Draw vertical borders
        for i in range(1, self.day_height-1):
            self.stdscr.addstr(y_pos + i, x_pos, "│", style)
            self.stdscr.addstr(y_pos + i, x_pos + self.day_width-1, "│", style)

        # Draw date
        date_str = f"{date.day:2d} {date.strftime('%a')}"
        self.stdscr.addstr(y_pos + 1, x_pos + 1, date_str.center(self.day_width-2), style)

        # Draw month name if it's the first of the month
        if date.day == 1:
            month_name = date.strftime("%B %Y")
            month_style = curses.color_pair(3)
            try:
                self.stdscr.addstr(y_pos - 1, x_pos, month_name.center(self.day_width), month_style)
            except curses.error:
                pass  # Ignore if we can't draw at this position

    def draw_calendar(self):
        """Draw the entire calendar scroll"""
        height, width = self.stdscr.getmaxyx()
        visible_weeks = (height // self.day_height) + 1
        
        # Calculate the date for the top of the current scroll position
        weeks_offset = self.scroll_position
        start_date = self.current_date - timedelta(days=self.current_date.weekday()) + timedelta(weeks=weeks_offset)
        
        # Draw each visible week
        for week in range(visible_weeks):
            for day in range(7):
                current_date = start_date + timedelta(days=(week * 7) + day)
                y_pos = week * self.day_height
                x_pos = day * (self.day_width + 1)
                
                is_selected = (week == self.selected_row and day == self.selected_col)
                
                try:
                    self.draw_day_block(current_date, y_pos, x_pos, is_selected)
                except curses.error:
                    pass  # Skip if we can't draw at this position

    def run(self):
        """Main loop"""
        curses.curs_set(0)  # Hide cursor
        self.stdscr.clear()
        
        while True:
            self.stdscr.clear()
            self.draw_calendar()
            self.stdscr.refresh()
            
            key = self.stdscr.getch()
            
            if key == ord('q'):
                break
            elif key == curses.KEY_UP:
                if self.selected_row > 0:
                    self.selected_row -= 1
                else:
                    self.scroll_position -= 1
            elif key == curses.KEY_DOWN:
                self.selected_row += 1
                if self.selected_row >= (self.stdscr.getmaxyx()[0] // self.day_height):
                    self.selected_row -= 1
                    self.scroll_position += 1
            elif key == curses.KEY_LEFT:
                self.selected_col = (self.selected_col - 1) % 7
            elif key == curses.KEY_RIGHT:
                self.selected_col = (self.selected_col + 1) % 7
            elif key == ord('\n'):
                # TODO: Add event creation/editing for selected day
                pass

def main(stdscr):
    calendar_scroll = CalendarScroll(stdscr)
    calendar_scroll.run()

if __name__ == '__main__':
    curses.wrapper(main)
