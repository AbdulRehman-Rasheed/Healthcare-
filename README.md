import wikipedia
import speech_recognition as sr
from deep_translator import GoogleTranslator
import webbrowser
from gtts import gTTS
import pyglet
import time
import os
import tkinter as tk
from PIL import Image, ImageTk
import requests
from io import BytesIO

# Supported Languages
SUPPORTED_LANGUAGES = {
    'en': 'English',
    'ur': 'Urdu'
}

# Speak using gTTS and pyglet
def speak(text, lang='en'):
    print(f"AI: {text}")
    tts = gTTS(text=text, lang=lang)
    tts.save("temp_audio.mp3")
    music = pyglet.media.load("temp_audio.mp3", streaming=False)
    music.play()
    time.sleep(music.duration + 0.5)
    os.remove("temp_audio.mp3")

# Listen using microphone and return recognized text in selected language
def listen(lang='en'):
    recognizer = sr.Recognizer()
    with sr.Microphone() as source:
        print(f"Listening... Speak your question in {SUPPORTED_LANGUAGES[lang]}.")
        audio = recognizer.listen(source)
    try:
        question = recognizer.recognize_google(audio, language=lang)
        print("You said:", question)
        return question
    except sr.UnknownValueError:
        speak("Sorry, I could not understand what you said.", lang)
        return None
    except sr.RequestError:
        speak("Speech recognition service failed.", lang)
        return None

# Translate to English
def translate_to_english(text):
    return GoogleTranslator(source='auto', target='en').translate(text)

# Translate from English
def translate_from_english(text, target_lang):
    return GoogleTranslator(source='en', target=target_lang).translate(text)

# Get summary from Wikipedia and return image URL and summary
def answer_question(query):
    try:
        summary = wikipedia.summary(query, sentences=2)
        page = wikipedia.page(query)
        image_url = page.images[0] if page.images else None
        return summary, image_url
    except wikipedia.exceptions.DisambiguationError as e:
        return f"Your question is too broad. Try one of these: {e.options[:5]}", None
    except wikipedia.exceptions.PageError:
        return "Sorry, I couldn't find any information on that topic.", None
    except Exception as e:
        return f"Something went wrong: {str(e)}", None

# Show image in tkinter window
def show_image(image_url):
    try:
        response = requests.get(image_url)
        img_data = BytesIO(response.content)
        img = Image.open(img_data)
        img = img.resize((300, 300))

        window = tk.Tk()
        window.title("Disease Image")
        img_tk = ImageTk.PhotoImage(img)
        label = tk.Label(window, image=img_tk)
        label.pack()
        window.mainloop()
    except Exception as e:
        print("Error loading image:", e)

# Main logic
def main():
    print("="*50)
    print("🩺 Welcome to Your Personal Health Assistant 🩺")
    print("="*50)
    speak("Hello! I am Atlas, your personal AI assistant. How can I help you today?", 'en')

    while True:
        print("\nChoose input type:\nT - Type\nS - Speak")
        choice = input("Enter T or S: ").strip().lower()

        if choice not in ['t', 's']:
            speak("Invalid input. Please enter T or S.", 'en')
            continue

        print("\nChoose language for question and answer:")
        for code, lang in SUPPORTED_LANGUAGES.items():
            print(f"{code} - {lang}")

        lang_choice = input("Enter language code (en or ur): ").strip().lower()
        if lang_choice not in SUPPORTED_LANGUAGES:
            speak("Invalid language selection.", 'en')
            continue

        # Choose voice option (male/female not controlled by gTTS)
        print("\nVoice option: Default synthetic voice will be used based on language.")

        if choice == 't':
            question = input(f"Enter your health-related question in {SUPPORTED_LANGUAGES[lang_choice]}: ")
        else:
            question = listen(lang=lang_choice)

        if question:
            translated_question = translate_to_english(question) if lang_choice != 'en' else question
            print(f"Processing your question: {translated_question}")
            response, image_url = answer_question(translated_question)

            final_response = translate_from_english(response, lang_choice) if lang_choice != 'en' else response
            speak(final_response, lang=lang_choice)

            if image_url:
                show_image(image_url)

        cont = input("\nPress Enter to continue or type 'exit' to quit: ").strip().lower()
        if cont == 'exit':
            speak("Goodbye!", lang_choice)
            break

if __name__ == "__main__":
    main()
