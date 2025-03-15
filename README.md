import speech_recognition as sr
import pyttsx3
import datetime
import re
import random
import threading
import time

class SmartHomeDevice:
    def __init__(self, name, device_type):
        self.name = name
        self.type = device_type
        self.status = "off"
        self.properties = {}
        
        # Initialize default properties based on device type
        if device_type == "light":
            self.properties["brightness"] = 50
            self.properties["color"] = "white"
        elif device_type == "thermostat":
            self.properties["temperature"] = 72
        elif device_type == "alarm":
            self.properties["time"] = "08:00"
            self.properties["status"] = "inactive"

    def turn_on(self):
        self.status = "on"
        return f"{self.name} is now on"
    
    def turn_off(self):
        self.status = "off"
        return f"{self.name} is now off"
    
    def set_property(self, property_name, value):
        if property_name in self.properties:
            self.properties[property_name] = value
            return f"Set {self.name} {property_name} to {value}"
        return f"{self.name} doesn't have property {property_name}"
    
    def get_status(self):
        status_str = f"{self.name} is {self.status}"
        for prop, value in self.properties.items():
            status_str += f", {prop}: {value}"
        return status_str

class VirtualAssistant:
    def __init__(self, name="Home Assistant"):
        # Initialize speech engine
        self.engine = pyttsx3.init()
        self.recognizer = sr.Recognizer()
        self.name = name
        self.devices = {}
        self.is_listening = False
        
        # Add some default devices
        self.add_device("living room light", "light")
        self.add_device("bedroom light", "light")
        self.add_device("kitchen light", "light")
        self.add_device("home thermostat", "thermostat")
        self.add_device("morning alarm", "alarm")
        
        # Greeting phrases
        self.greetings = [
            "Hello! How can I help you today?",
            "Hi there! What can I do for you?",
            "Hey! I'm listening. What do you need?",
            "Good day! How may I assist you?"
        ]
        
        # Command patterns
        self.command_patterns = [
            (r"turn (on|off) (the )?([\w\s]+)", self.handle_power_command),
            (r"set (the )?([\w\s]+) (brightness|temperature|color|time) to ([\w\s]+)", self.handle_property_command),
            (r"what('s| is) the status of (the )?([\w\s]+)", self.handle_status_command),
            (r"add (a )?(new )?([\w\s]+) (light|thermostat|alarm)", self.handle_add_device),
            (r"what time is it", self.handle_time_command),
            (r"(bye|goodbye|exit|quit|stop listening)", self.handle_exit_command),
            (r"help( me)?", self.handle_help_command),
        ]
    
    def add_device(self, name, device_type):
        self.devices[name.lower()] = SmartHomeDevice(name, device_type)
        return f"Added {name} as a {device_type}"
    
    def speak(self, text):
        """Convert text to speech"""
        print(f"{self.name}: {text}")
        self.engine.say(text)
        self.engine.runAndWait()
    
    def listen(self):
        """Listen for voice commands"""
        with sr.Microphone() as source:
            print("Listening...")
            self.recognizer.adjust_for_ambient_noise(source, duration=0.5)
            try:
                audio = self.recognizer.listen(source, timeout=5)
                print("Processing...")
                text = self.recognizer.recognize_google(audio)
                print(f"You said: {text}")
                return text.lower()
            except sr.WaitTimeoutError:
                return ""
            except sr.UnknownValueError:
                self.speak("Sorry, I didn't catch that.")
                return ""
            except sr.RequestError:
                self.speak("Sorry, my speech service is down.")
                return ""
    
    def process_command(self, command):
        """Process the voice command"""
        if not command:
            return
            
        for pattern, handler in self.command_patterns:
            match = re.match(pattern, command, re.IGNORECASE)
            if match:
                return handler(*match.groups())
        
        self.speak("I'm not sure how to help with that. Try asking for help for a list of commands.")
    
    def handle_power_command(self, action, _, device_name):
        """Handle turn on/off commands"""
        device_name = device_name.strip().lower()
        if device_name in self.devices:
            if action == "on":
                response = self.devices[device_name].turn_on()
            else:
                response = self.devices[device_name].turn_off()
            self.speak(response)
        else:
            self.speak(f"I couldn't find a device called {device_name}")
    
    def handle_property_command(self, _, device_name, property_name, value):
        """Handle setting properties"""
        device_name = device_name.strip().lower()
        if device_name in self.devices:
            response = self.devices[device_name].set_property(property_name, value)
            self.speak(response)
        else:
            self.speak(f"I couldn't find a device called {device_name}")
    
    def handle_status_command(self, _, __, device_name):
        """Handle status request"""
        device_name = device_name.strip().lower()
        if device_name in self.devices:
            status = self.devices[device_name].get_status()
            self.speak(status)
        else:
            self.speak(f"I couldn't find a device called {device_name}")
    
    def handle_add_device(self, _, __, device_name, device_type):
        """Handle adding new devices"""
        device_name = device_name.strip().lower()
        response = self.add_device(device_name, device_type)
        self.speak(response)
    
    def handle_time_command(self):
        """Handle time requests"""
        current_time = datetime.datetime.now().strftime("%I:%M %p")
        self.speak(f"The current time is {current_time}")
    
    def handle_exit_command(self, _):
        """Handle exit command"""
        self.speak("Goodbye! Have a great day!")
        self.is_listening = False
        return "exit"
    
    def handle_help_command(self, _):
        """Provide help information"""
        help_text = """
        Here are some commands you can try:
        - Turn on/off the [device name]
        - Set the [device name] brightness/temperature/color/time to [value]
        - What's the status of the [device name]
        - Add a new [device name] light/thermostat/alarm
        - What time is it
        - Goodbye or exit to stop
        """
        self.speak(help_text)
    
    def simulate_voice_input(self, text):
        """For testing without microphone"""
        print(f"You said: {text}")
        return self.process_command(text.lower())
    
    def run(self, use_voice=True):
        """Run the assistant"""
        self.is_listening = True
        self.speak(random.choice(self.greetings))
        
        while self.is_listening:
            if use_voice:
                command = self.listen()
                if command and self.process_command(command) == "exit":
                    break
            else:
                # For demo/testing without microphone
                command = input("Enter a command: ")
                if command and self.process_command(command) == "exit":
                    break


def demo_run():
    """Run a demonstration of the assistant without using voice input"""
    assistant = VirtualAssistant()
    
    # Demonstrate various commands
    print("\n--- SMART HOME ASSISTANT DEMO ---\n")
    
    demo_commands = [
        "turn on the living room light",
        "set the living room light brightness to 75",
        "set the living room light color to blue",
        "what's the status of the living room light",
        "set the home thermostat temperature to 68",
        "what's the status of the home thermostat",
        "add a new bathroom light",
        "turn on the bathroom light",
        "what time is it",
        "help",
        "goodbye"
    ]
    
    for command in demo_commands:
        print(f"\nCommand: {command}")
        response = assistant.simulate_voice_input(command)
        time.sleep(1)  # Pause between commands for readability

if __name__ == "__main__":
    # Uncomment the appropriate method to run
    
    # For demo without voice input:
    demo_run()
    
    # For actual voice-enabled assistant:
    # assistant = VirtualAssistant()
    # assistant.run(use_voice=True)
