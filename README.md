from kivy.app import App
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.label import Label
from kivy.uix.textinput import TextInput
from kivy.uix.button import Button
from kivy.uix.popup import Popup  # Import Popup
from kivy.clock import Clock
from datetime import datetime, timedelta
from plyer import notification
import pyttsx3
from twilio.rest import Client
import json
import os

class Medicine:
    def __init__(self, name, dose, times, quantity, start_date, duration_days):
        self.name = name
        self.dose = dose
        self.times = times  # List of times
        self.quantity = quantity
        self.start_date = start_date
        self.end_date = start_date + timedelta(days=duration_days)
        self.taken_times = {}  # Keep track of when the medicine was taken

    def take_medicine(self, time):
        current_date = datetime.now().date()
        if current_date in self.taken_times and time in self.taken_times[current_date]:
            return f"Medicine {self.name} has already been taken at {time} today!"
        if time not in self.times:
            return f"Invalid time for {self.name}. Valid times are {', '.join(self.times)}."
        if self.quantity > 0:
            self.quantity -= 1
            if current_date not in self.taken_times:
                self.taken_times[current_date] = []
            self.taken_times[current_date].append(time)
            self.update_medicine()  # Update medicine data after taking it
            if set(self.taken_times[current_date]) == set(self.times):
                return f"All doses for {self.name} have been taken for today."
            return f"Time to take {self.name} ({self.dose}) at {time}. Remaining: {self.quantity} tablets."
        else:
            return f"{self.name} is out of stock!"

    def update_medicine(self):
        data = load_data()
        data[self.name]['quantity'] = self.quantity  # Update quantity
        save_data(data)  # Save updated data to file

# Load data from a JSON file
def load_data():
    file_path = 'medicines.json'
    if os.path.exists(file_path):
        with open(file_path, 'r') as file:
            return json.load(file)
    return {}

# Save data to a JSON file
def save_data(data):
    with open('medicines.json', 'w') as file:
        json.dump(data, file, indent=4)

class MedicineReminderApp(App):
    def build(self):
        self.engine = pyttsx3.init()
        self.twilio_client = Client("TWILIO_ACCOUNT_SID", "TWILIO_AUTH_TOKEN")
        self.twilio_from = "TWILIO_PHONE_NUMBER"
        self.twilio_to = "USER_PHONE_NUMBER"

        # Load existing medicines data
        medicines_data = load_data()
        self.medicines = [
            Medicine("Thyronorm 50 mg", "50 mg", ["07:00"], medicines_data.get("Thyronorm 50 mg", {}).get("quantity", 120), datetime(2024, 3, 4), 30),
            Medicine("Glycomet GP 1", "1 mg", ["09:30", "13:30", "19:30"], medicines_data.get("Glycomet GP 1", {}).get("quantity", 30), datetime(2024, 3, 4), 15),
            Medicine("Olsertan 40", "40 mg", ["10:00", "14:00", "20:00"], medicines_data.get("Olsertan 40", {}).get("quantity", 30), datetime(2024, 3, 4), 15)
        ]

        self.layout = BoxLayout(orientation='vertical')

        # Display each medicine with its available times and quantity
        for med in self.medicines:
            med_label = Label(text=f"{med.name} at {', '.join(med.times)} - {med.quantity} tablets left")
            self.layout.add_widget(med_label)

        self.reminder_label = Label(text="")
        self.layout.add_widget(self.reminder_label)

        # Button to update reminders
        self.update_button = Button(text="Update Reminders")
        self.update_button.bind(on_press=self.update_reminders)
        self.layout.add_widget(self.update_button)

        # Button to add a new medicine
        self.add_medicine_button = Button(text="Add Medicine")
        self.add_medicine_button.bind(on_press=self.add_medicine)
        self.layout.add_widget(self.add_medicine_button)

        # Schedule a periodic check for medicine time every 60 seconds
        Clock.schedule_interval(self.check_medicine_time, 60)

        return self.layout

    # Add new medicine by opening a popup
    def add_medicine(self, instance):
        layout = BoxLayout(orientation='vertical')
        med_name_input = TextInput(hint_text='Medicine Name')
        med_dose_input = TextInput(hint_text='Dose')
        med_time_input = TextInput(hint_text='Times (HH:MM, comma separated)')
        med_quantity_input = TextInput(hint_text='Quantity')
        med_duration_input = TextInput(hint_text='Duration (Days)')
        med_start_date_input = TextInput(hint_text='Start Date (YYYY-MM-DD)', multiline=False)

        add_button = Button(text="Add")
        add_button.bind(on_press=lambda x: self.save_new_medicine(
            med_name_input.text,
            med_dose_input.text,
            med_time_input.text.split(","),
            int(med_quantity_input.text),
            med_start_date_input.text,
            int(med_duration_input.text)
        ))

        layout.add_widget(med_name_input)
        layout.add_widget(med_dose_input)
        layout.add_widget(med_time_input)
        layout.add_widget(med_quantity_input)
        layout.add_widget(med_duration_input)
        layout.add_widget(med_start_date_input)
        layout.add_widget(add_button)

        popup = Popup(title="Add New Medicine", content=layout, size_hint=(0.9, 0.9))
        popup.open()

    # Save new medicine data to the medicines list and file
    def save_new_medicine(self, name, dose, times, quantity, start_date, duration_days):
        try:
            start_date = datetime.strptime(start_date, "%Y-%m-%d")
            new_medicine = Medicine(name, dose, times, quantity, start_date, duration_days)
            self.medicines.append(new_medicine)

            data = load_data()
            data[name] = {
                'dose': dose,
                'times': times,
                'quantity': quantity,
                'start_date': start_date.strftime('%Y-%m-%d'),
                'duration_days': duration_days
            }
            save_data(data)
            self.update_reminders(None)
        except Exception as e:
            print(f"Error adding new medicine: {e}")

    # Periodic check to see if it's time to take any medicine
    def check_medicine_time(self, dt):
        current_time = datetime.now()
        for med in self.medicines:
            for med_time in med.times:
                med_time_obj = datetime.strptime(med_time, "%H:%M")
                if current_time.date() >= med.start_date.date() and current_time.date() <= med.end_date.date():
                    if current_time.hour == med_time_obj.hour and current_time.minute == med_time_obj.minute:
                        message = med.take_medicine(med_time)
                        self.reminder_label.text = message
                        notification.notify(
                            title=f"MEDTIK Reminder: {med.name} at {med_time}",
                            message=message,
                            timeout=10  # Notification duration in seconds
                        )
                        self.engine.say(message)
                        self.engine.runAndWait()
                        self.send_sms(message)

    # Update reminder labels on the UI
    def update_reminders(self, instance):
        for idx, med in enumerate(self.medicines):
            med_label = self.layout.children[-(idx + 2)]  # Accessing corresponding label
            med_label.text = f"{med.name} at {', '.join(med.times)} - {med.quantity} tablets left"
        self.reminder_label.text = "Reminders Updated!"

    # Send SMS notifications
    def send_sms(self, message):
        try:
            self.twilio_client.messages.create(
                body=message,
                from_=self.twilio_from,
                to=self.twilio_to
            )
        except Exception as e:
            print(f"Error sending SMS: {e}")

if __name__ == '__main__':
    MedicineReminderApp().run()
