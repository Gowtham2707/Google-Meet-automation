# Google-Meet-automation
Google Meet automation using selenium 
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from webdriver_manager.chrome import ChromeDriverManager
import time
import speech_recognition as sr
import wave
import os
import pyttsx3
import pyaudio
import transformers
from transformers import pipeline

username = ""
password = ""
os.environ["PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION"] = "python"

service = Service(ChromeDriverManager().install())
driver = webdriver.Chrome(service=service)

driver.get('https://meet.google.com/')

time.sleep(3)

start_meet_button = driver.find_element(By.XPATH, '//*[@id="page-content"]/section[1]/div/div[1]/div[2]/div/a/button')
start_meet_button.click()
time.sleep(3)
driver.find_element(By.ID, "identifierId").send_keys(username)
time.sleep(3)
driver.find_element(By.ID, "identifierNext").click()
time.sleep(2)
driver.find_element(By.NAME, "Passwd").send_keys(password)
time.sleep(2)
driver.find_element(By.ID, "passwordNext").click()

print("logged in successfully")

def take_meeting_notes():
    r = sr.Recognizer()
    engine = pyttsx3.init()

    # Wait for the meeting audio to start
    time.sleep(5)

    # Define audio settings
    CHUNK = 1024
    FORMAT = pyaudio.paInt16
    CHANNELS = 1
    RATE = 44100
    WAVE_OUTPUT_FILENAME = "meeting_audio.wav"

    # Open the audio stream
    audio = pyaudio.PyAudio()
    stream = audio.open(format=FORMAT,
                        channels=CHANNELS,
                        rate=RATE,
                        input=True,
                        frames_per_buffer=CHUNK)

    print("Listening...")
    frames = []

    try:
        # Wait until the "Leave call" button appears, indicating the end of the meeting
        leave_call_button = WebDriverWait(driver, 600).until(
            EC.presence_of_element_located(
                (By.XPATH, '//*[@id="ow3"]/div[1]/div/div[14]/div[3]/div[11]/div/div/div[2]/div/div[8]/span/button'))
        )
        
    print("Processing...")

    # Stop recording and close the audio stream
    stream.stop_stream()
    stream.close()
    audio.terminate()

    # Save the recorded audio to a file
    wf = wave.open(WAVE_OUTPUT_FILENAME, 'wb')
    wf.setnchannels(CHANNELS)
    wf.setsampwidth(audio.get_sample_size(FORMAT))
    wf.setframerate(RATE)
    wf.writeframes(b''.join(frames))
    wf.close()

    # Convert speech to text
    with sr.AudioFile(WAVE_OUTPUT_FILENAME) as source:
        audio_data = r.record(source)
        notes = r.recognize_google(audio_data)

    print("Meeting Notes:")
    print(notes)

    # Extract action items and minutes of meeting
    action_items = extract_action_items(notes)
    minutes_of_meeting = extract_minutes_of_meeting(notes)

    # Print action items
    print("Action Items:")
    for item in action_items:
        print(item)

    # Print minutes of meeting
    print("Minutes of Meeting:")
    print(minutes_of_meeting)

    # Save the notes, action items, and minutes of meeting
    save_notes(notes)
    save_action_items(action_items)
    save_minutes_of_meeting(minutes_of_meeting)

    # Generate summary
    summary = generate_summary(notes)
    print("Meeting Summary:")
    print(summary)
    save_summary(summary)

    # Text-to-speech
    engine.say("Meeting notes captured successfully.")
    engine.runAndWait()


def extract_action_items(notes):
    action_items = []
    keywords = ["action", "task", "follow-up"]
    lines = notes.split('\n')
    for line in lines:
        for keyword in keywords:
            if keyword in line.lower():
                action_items.append(line)
                break
    return action_items


def extract_minutes_of_meeting(notes):
    minutes_of_meeting = ""
    keywords = ["decision", "conclusion", "discussion"]
    lines = notes.split('\n')
    for line in lines:
        for keyword in keywords:
            if keyword in line.lower():
                minutes_of_meeting += line + "\n"
                break
    return minutes_of_meeting


def save_notes(notes):
    with open("meeting_notes.txt", "w") as file:
        file.write(notes)


def save_action_items(action_items):
    with open("action_items.txt", "w") as file:
        for item in action_items:
            file.write(item + "\n")


def save_minutes_of_meeting(minutes_of_meeting):
    with open("minutes_of_meeting.txt", "w") as file:
        file.write(minutes_of_meeting)


def generate_summary(text):
    summarizer = pipeline("summarization")
    summary = summarizer(text, max_length=150, min_length=30, do_sample=False)
    return summary[0]['summary_text']


def save_summary(summary):
    with open("meeting_summary.txt", "w") as file:
        file.write(summary)


if __name__ == '__main__':
    take_meeting_notes()
