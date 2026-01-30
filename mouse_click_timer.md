import pyautogui
import time
import sys

def timed_click(delay):
    print(f"Waiting for {delay} seconds...")
    time.sleep(delay)  # Wait for the specified time
    pyautogui.click()  # Perform the mouse click
    print("Mouse clicked!")

if __name__ == "__main__":
    # Check if delay is provided as a command-line argument
    if len(sys.argv) > 1:
        try:
            delay_in_seconds = float(sys.argv[1])
        except ValueError:
            print("Please provide a valid number for the delay.")
            sys.exit(1)
    else:
        # Prompt the user for the delay
        delay_in_seconds = float(input("Enter the delay time in seconds: "))

    timed_click(delay_in_seconds)

