import curses
from datetime import datetime, timedelta
import calendar
import os

class CalendarApp:
    def __init__(self, stdscr):
        self.stdscr = stdscr
        self.current_date = datetime.now()
        self.scroll_position = 0
        self.selected_row = 0
        self.selected_col = self.current_date.weekday()
        self.day_width = 10
        self.day_height = 5
        self.events = self.load_events()

        curses.start_color()
        curses.init_pair(1, curses.COLOR_BLACK, curses.COLOR_WHITE)  # Selected day
        curses.init_pair(2, curses.COLOR_YELLOW, curses.COLOR_BLACK)  # Current day
        curses.init_pair(3, curses.COLOR_CYAN, curses.COLOR_BLACK)    # Month separator
        curses.init_pair(4, curses.COLOR_GREEN, curses.COLOR_BLACK)   # Event indicator

    def load_events(self):
        events = {}
        if os.path.exists('calendar.txt'):
            with open('calendar.txt', 'r') as f:
                for line in f:
                    date_str, event = line.strip().split(':', 1)
                    date = datetime.strptime(date_str, '%Y-%m-%d').date()
                    if date not in events:
                        events[date] = []
                    events[date].append(event)
        return events

    def save_events(self):
        with open('calendar.txt', 'w') as f:
            for date, event_list in self.events.items():
                for event in event_list:
                    f.write(f"{date.strftime('%Y-%m-%d')}:{event}\n")

    def draw_day_block(self, date, y_pos, x_pos, is_selected):
        is_current = date.date() == datetime.now().date()
        style = curses.color_pair(1) if is_selected else curses.A_NORMAL
        if is_current:
            style = curses.color_pair(2)

        self.stdscr.addstr(y_pos, x_pos, "┌" + "─"*(self.day_width-2) + "┐", style)
        self.stdscr.addstr(y_pos + self.day_height - 1, x_pos, 
                          "└" + "─"*(self.day_width-2) + "┘", style)
        
        for i in range(1, self.day_height-1):
            self.stdscr.addstr(y_pos + i, x_pos, "│", style)
            self.stdscr.addstr(y_pos + i, x_pos + self.day_width-1, "│", style)

        date_str = f"{date.day:2d}"
        self.stdscr.addstr(y_pos + 1, x_pos + 1, date_str.center(self.day_width-2), style)

        if date.date() in self.events:
            self.stdscr.addstr(y_pos + 3, x_pos + 1, "•".center(self.day_width-2), curses.color_pair(4))

    def draw_calendar(self):
        height, width = self.stdscr.getmaxyx()
        visible_weeks = (height // self.day_height) + 1
        
        weeks_offset = self.scroll_position
        start_date = self.current_date - timedelta(days=self.current_date.weekday()) + timedelta(weeks=weeks_offset)
        
        for week in range(visible_weeks):
            for day in range(7):
                current_date = start_date + timedelta(days=(week * 7) + day)
                y_pos = week * self.day_height
                x_pos = day * (self.day_width + 1)
                
                is_selected = (week == self.selected_row and day == self.selected_col)
                
                if current_date.day == 1 or (week == 0 and day == 0):
                    month_name = current_date.strftime("%B %Y")
                    try:
                        self.stdscr.addstr(y_pos - 1, x_pos, month_name, curses.color_pair(3))
                    except curses.error:
                        pass
                
                try:
                    self.draw_day_block(current_date, y_pos, x_pos, is_selected)
                except curses.error:
                    pass

        self.stdscr.refresh()

    def add_event(self):
        selected_date = self.current_date - timedelta(days=self.current_date.weekday()) + \
                        timedelta(weeks=self.scroll_position + self.selected_row, days=self.selected_col)
        
        self.stdscr.clear()
        self.stdscr.addstr(0, 0, f"Add event for {selected_date.strftime('%Y-%m-%d')}:")
        self.stdscr.addstr(2, 0, "Enter event (or press Enter to finish):")
        
        events = []
        curses.echo()
        while True:
            self.stdscr.move(3 + len(events), 0)
            event = self.stdscr.getstr().decode('utf-8')
            if not event:
                break
            events.append(event)
        curses.noecho()

        if events:
            if selected_date.date() not in self.events:
                self.events[selected_date.date()] = []
            self.events[selected_date.date()].extend(events)
            self.save_events()

    def view_events(self):
        selected_date = self.current_date - timedelta(days=self.current_date.weekday()) + \
                        timedelta(weeks=self.scroll_position + self.selected_row, days=self.selected_col)
        
        self.stdscr.clear()
        self.stdscr.addstr(0, 0, f"Events for {selected_date.strftime('%Y-%m-%d')}:")
        
        if selected_date.date() in self.events:
            for i, event in enumerate(self.events[selected_date.date()]):
                self.stdscr.addstr(i+2, 0, f"{i+1}. {event}")
        else:
            self.stdscr.addstr(2, 0, "No events for this date.")
        
        self.stdscr.addstr(self.stdscr.getmaxyx()[0]-1, 0, "Press any key to return")
        self.stdscr.refresh()
        self.stdscr.getch()

    def run(self):
        curses.curs_set(0)  # Hide cursor
        self.stdscr.clear()
        
        while True:
            self.stdscr.clear()
            self.draw_calendar()
            
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
            elif key == ord('\n'):  # Enter key
                self.add_event()
            elif key == ord('v'):
                self.view_events()

def main(stdscr):
    calendar_app = CalendarApp(stdscr)
    calendar_app.run()

if __name__ == '__main__':
    curses.wrapper(main)