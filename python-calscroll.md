import curses
from datetime import datetime, timedelta

def get_week_dates(year, week_number):
    """Convert week number to start/end dates"""
    # Jan 1st of the year
    first_day = datetime(year, 1, 1)
    # Find Monday of week 1
    week_1_start = first_day - timedelta(days=first_day.weekday())
    # Calculate start of requested week
    week_start = week_1_start + timedelta(weeks=week_number-1)
    return [week_start + timedelta(days=x) for x in range(7)]

def draw_week(window, week_dates, y_pos, is_selected):
    """Draw a single week row"""
    # Highlight if selected
    attr = curses.A_REVERSE if is_selected else curses.A_NORMAL
    
    # Format: Week 42, 2024: Oct 14 - Oct 20
    week_str = f"Week {week_dates[0].isocalendar()[1]}, {week_dates[0].year}: "
    week_str += f"{week_dates[0].strftime('%b %d')} - {week_dates[6].strftime('%b %d')}"
    
    window.addstr(y_pos, 2, week_str, attr)

def main(stdscr):
    # Setup
    curses.curs_set(0)  # Hide cursor
    current_date = datetime.now()
    current_year = current_date.year
    total_weeks = 52
    selected_week = current_date.isocalendar()[1]
    scroll_pos = 0
    
    while True:
        stdscr.clear()
        height, width = stdscr.getmaxyx()
        visible_weeks = height - 2  # Leave room for border
        
        # Draw visible weeks
        for i in range(visible_weeks):
            week_num = i + scroll_pos + 1
            if week_num > total_weeks:
                break
                
            week_dates = get_week_dates(current_year, week_num)
            draw_week(stdscr, week_dates, i + 1, week_num == selected_week)
        
        # Handle input
        key = stdscr.getch()
        if key == ord('q'):
            break
        elif key == curses.KEY_UP and selected_week > 1:
            selected_week -= 1
            if selected_week < scroll_pos + 1:
                scroll_pos = max(0, scroll_pos - 1)
        elif key == curses.KEY_DOWN and selected_week < total_weeks:
            selected_week += 1
            if selected_week > scroll_pos + visible_weeks:
                scroll_pos += 1
        elif key == ord('\n'):
            # TODO: Handle week selection
            pass

if __name__ == '__main__':
    curses.wrapper(main)
