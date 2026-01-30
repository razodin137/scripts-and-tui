import PySimpleGUI as sg

class TodoListGUI:
    def __init__(self):
        self.tasks = []
        self.layout = [
            [sg.Text("To-Do List")],
            [sg.InputText(key='-TASK-'), sg.Button("Add Task")],
            [sg.Listbox(values=[], size=(40, 10), key='-LIST-')],
            [sg.Button("Mark as Completed"), sg.Button("Remove Task")]
        ]
        self.window = sg.Window("To-Do List App", self.layout)

    def run(self):
        while True:
            event, values = self.window.read()
            if event == sg.WINDOW_CLOSED:
                break
            if event == "Add Task":
                task = values['-TASK-']
                if task:
                    self.tasks.append({"task": task, "completed": False})
                    self.window['-TASK-'].update('')
            elif event == "Mark as Completed":
                if values['-LIST-']:
                    index = self.tasks.index(values['-LIST-'][0])
                    self.tasks[index]["completed"] = True
            elif event == "Remove Task":
                if values['-LIST-']:
                    index = self.tasks.index(values['-LIST-'][0])
                    del self.tasks[index]
            self.refresh_list()

    def refresh_list(self):
        task_list = [f"[{'✓' if task['completed'] else ' '}] {task['task']}" for task in self.tasks]
        self.window['-LIST-'].update(task_list)

    def close(self):
        self.window.close()

def main():
    app = TodoListGUI()
    app.run()
    app.close()

if __name__ == "__main__":
    main()
