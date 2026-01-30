import curses
from datetime import datetime, timedelta
import calendar
from icalendar import Calendar, Event
import os
import uuid

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
        if os.path.exists('calendar.ics'):
            with open('calendar.ics', 'rb') as f:
                cal = Calendar.from_ical(f.read())
                for component in cal.walk():
                    if component.name == "VEVENT":
                        event = {
                            'summary': str(component.get('summary')),
                            'description': str(component.get('description')),
                            'location': str(component.get('location')),
                            'dtstart': component.get('dtstart').dt,
                            'dtend': component.get('dtend').dt if component.get('dtend') else None,
                            'uid': str(component.get('uid'))
                        }
                        date = event['dtstart'].date() if isinstance(event['dtstart'], datetime) else event['dtstart']
                        if date not in events:
                            events[date] = []
                        events[date].append(event)
        return events

    def save_events(self):
        cal = Calendar()
        for date, event_list in self.events.items():
            for event in event_list:
                e = Event()
                e.add('summary', event['summary'])
                e.add('description', event['description'])
                e.add('location', event['location'])
                e.add('dtstart', event['dtstart'])
                if event['dtend']:
                    e.add('dtend', event['dtend'])
                e.add('uid', event['uid'])
                cal.add_component(e)
        
        with open('calendar.ics', 'wb') as f:
            f.write(cal.to_ical())

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

    def add_event(self, date):
        self.stdscr.clear()
        self.stdscr.addstr(0, 0, f"Add event for {date.strftime('%Y-%m-%d')}:")
        self.stdscr.addstr(2, 0, "Summary (required): ")
        curses.echo()
        summary = self.stdscr.getstr().decode('utf-8')
        curses.noecho()

        self.stdscr.addstr(3, 0, "Description (optional, press Enter to skip): ")
        curses.echo()
        description = self.stdscr.getstr().decode('utf-8')
        curses.noecho()

        self.stdscr.addstr(4, 0, "Location (optional, press Enter to skip): ")
        curses.echo()
        location = self.stdscr.getstr().decode('utf-8')
        curses.noecho()

        self.stdscr.addstr(5, 0, "End date (YYYY-MM-DD, optional, press Enter to skip): ")
        curses.echo()
        end_date_str = self.stdscr.getstr().decode('utf-8')
        curses.noecho()

        event = {
            'summary': summary,
            'description': description,
            'location': location,
            'dtstart': date,
            'dtend': datetime.strptime(end_date_str, '%Y-%m-%d').date() if end_date_str else None,
            'uid': str(uuid.uuid4())
        }

        if date not in self.events:
            self.events[date] = []
        self.events[date].append(event)
        self.save_events()

    def edit_event(self, date, event_index):
        event = self.events[date][event_index]
        self.stdscr.clear()
        self.stdscr.addstr(0, 0, f"Edit event for {date.strftime('%Y-%m-%d')}:")
        self.stdscr.addstr(2, 0, f"Summary (current: {event['summary']}): ")
        curses.echo()
        summary = self.stdscr.getstr().decode('utf-8')
        curses.noecho()
        if summary:
            event['summary'] = summary

        self.stdscr.addstr(3, 0, f"Description (current: {event['description']}): ")
        curses.echo()
        description = self.stdscr.getstr().decode('utf-8')
        curses.noecho()
        if description:
            event['description'] = description

        self.stdscr.addstr(4, 0, f"Location (current: {event['location']}): ")
        curses.echo()
        location = self.stdscr.getstr().decode('utf-8')
        curses.noecho()
        if location:
            event['location'] = location

        self.stdscr.addstr(5, 0, f"End date (current: {event['dtend']}, format YYYY-MM-DD): ")
        curses.echo()
        end_date_str = self.stdscr.getstr().decode('utf-8')
        curses.noecho()
        if end_date_str:
            event['dtend'] = datetime.strptime(end_date_str, '%Y-%m-%d').date()

        self.save_events()

    def delete_event(self, date, event_index):
        del self.events[date][event_index]
        if not self.events[date]:
            del self.events[date]
        self.save_events()

    def view_day(self, date):
        while True:
            self.stdscr.clear()
            height, width = self.stdscr.getmaxyx()
            self.stdscr.addstr(0, 0, f"Events for {date.strftime('%Y-%m-%d')}:")
            
            if date in self.events:
                for i, event in enumerate(self.events[date]):
                    if i+2 < height - 2:  # Leave space for instructions
                        self.stdscr.addstr(i+2, 0, f"{i+1}. {event['summary']}"[:width-1])
            else:
                self.stdscr.addstr(2, 0, "No events for this date.")
            
            instructions = "A: Add, E: Edit, D: Delete, V: View, Q: Return"
            try:
                self.stdscr.addstr(height-1, 0, instructions[:width-1])
            except curses.error:
                pass  # Ignore if we can't write to the last line
            
            self.stdscr.refresh()
            
            key = self.stdscr.getch()
            
            if key == ord('q'):
                break
            elif key == ord('a'):
                self.add_event(date)
            elif key == ord('e'):
                if date in self.events:
                    self.stdscr.addstr(height-2, 0, "Enter event number to edit: ")
                    curses.echo()
                    event_index = int(self.stdscr.getstr().decode('utf-8')) - 1
                    curses.noecho()
                    if 0 <= event_index < len(self.events[date]):
                        self.edit_event(date, event_index)
            elif key == ord('d'):
                if date in self.events:
                    self.stdscr.addstr(height-2, 0, "Enter event number to delete: ")
                    curses.echo()
                    event_index = int(self.stdscr.getstr().decode('utf-8')) - 1
                    curses.noecho()
                    if 0 <= event_index < len(self.events[date]):
                        self.delete_event(date, event_index)
            elif key == ord('v'):
                if date in self.events:
                    self.stdscr.addstr(height-2, 0, "Enter event number to view (or press Enter to cancel): ")
                    curses.echo()
                    user_input = self.stdscr.getstr().decode('utf-8').strip()
                    curses.noecho()
                    if user_input:
                        try:
                            event_index = int(user_input) - 1
                            if 0 <= event_index < len(self.events[date]):
                                self.view_event_details(date, event_index)
                            else:
                                self.show_message("Invalid event number. Press any key to continue.")
                        except ValueError:
                            self.show_message("Invalid input. Please enter a number. Press any key to continue.")

    def view_event_details(self, date, event_index):
        event = self.events[date][event_index]
        self.stdscr.clear()
        height, width = self.stdscr.getmaxyx()
        self.stdscr.addstr(0, 0, f"Event details for {date.strftime('%Y-%m-%d')}:")
        details = [
            f"Summary: {event['summary']}",
            f"Description: {event['description']}",
            f"Location: {event['location']}",
            f"Start: {event['dtstart']}",
            f"End: {event['dtend']}"
        ]
        for i, detail in enumerate(details):
            if i+2 < height - 1:
                self.stdscr.addstr(i+2, 0, detail[:width-1])
        self.stdscr.addstr(height-1, 0, "Press any key to return")
        self.stdscr.getch()

    def show_message(self, message):
        height, width = self.stdscr.getmaxyx()
        self.stdscr.addstr(height-2, 0, message[:width-1])
        self.stdscr.getch()  # Wait for user input

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
                selected_date = self.current_date - timedelta(days=self.current_date.weekday()) + \
                                timedelta(weeks=self.scroll_position + self.selected_row, days=self.selected_col)
                self.view_day(selected_date.date())

def main(stdscr):
    calendar_app = CalendarApp(stdscr)
    calendar_app.run()

if __name__ == '__main__':
    curses.wrapper(main)


