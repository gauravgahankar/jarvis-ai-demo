# jarvis-ai-demo
This is an AI assistant just like jarvis in Iron Man movie.
author :- Gaurav Gahankar
topic :- source code of Jarvis AI 

"""
Simple 'Jarvis' assistant - single-file.
Features:
- Wakeword 'jarvis' (voice) or press Enter to type commands
- Text-to-speech (pyttsx3)
- Speech recognition (recognize_google via SpeechRecognition)
- Wikipedia search
- Open websites (YouTube, Google, GitHub, etc.)
- Tell time/date
- Play a music file (simple folder)
- Run shell commands (careful!)
- Fallback to typed input if microphone not available
"""

import os
import sys
import webbrowser
import subprocess
import threading
import time
import datetime
import traceback

try:
    import speech_recognition as sr
except Exception:
    sr = None

try:
    import pyttsx3
except Exception:
    pyttsx3 = None

try:
    import wikipedia
except Exception:
    wikipedia = None

# ---------- Configuration ----------
WAKE_WORD = "jarvis"
MUSIC_FOLDER = os.path.expanduser("~/Music")  # change if needed
VOICE_RATE = 150
# -----------------------------------

def log(*args, **kwargs):
    print("[Jarvis]", *args, **kwargs)

# TTS
class Talker:
    def __init__(self):
        if pyttsx3 is None:
            self.engine = None
            log("pyttsx3 not installed. Text replies will be printed only.")
        else:
            self.engine = pyttsx3.init()
            self.engine.setProperty('rate', VOICE_RATE)

    def say(self, text):
        """Speak and wait until done (no background threads)."""
        if self.engine:
            self.engine.say(text)
            self.engine.runAndWait()
        print("Jarvis:", text)


talker = Talker()

# Speech recognition
def listen_phrase(timeout=5, phrase_time_limit=8):
    """
    Returns recognized text or None.
    Uses SpeechRecognition + Google Web Speech API (no API key required),
    but requires internet for recognition. If SR isn't available, returns None.
    """
    if sr is None:
        return None
    r = sr.Recognizer()
    mic = None
    try:
        mic = sr.Microphone()
    except Exception as e:
        log("Microphone not available:", e)
        return None

    with mic as source:
        r.adjust_for_ambient_noise(source, duration=0.8)
        try:
            audio = r.listen(source, timeout=timeout, phrase_time_limit=phrase_time_limit)
        except sr.WaitTimeoutError:
            return None

    try:
        text = r.recognize_google(audio)
        return text.lower()
    except sr.UnknownValueError:
        return None
    except sr.RequestError as e:
        log("Speech API error:", e)
        return None

# Command handlers
def handle_search_wikipedia(query):
    if wikipedia is None:
        talker.say("Wikipedia module is not installed. I can open Wikipedia in your browser instead.")
        webbrowser.open("https://en.wikipedia.org/wiki/" + query.replace(" ", "_"))
        return
    try:
        talker.say("Searching Wikipedia for " + query, block=False)
        summary = wikipedia.summary(query, sentences=2, auto_suggest=True, redirect=True)
        talker.say(summary)
    except Exception as e:
        log("Wikipedia error:", e)
        talker.say("Sorry, I couldn't find that on Wikipedia. I'll open the page in your browser.")
        webbrowser.open("https://en.wikipedia.org/wiki/" + query.replace(" ", "_"))

def handle_open_website(site):
    sites = {
        "youtube": "https://www.youtube.com",
        "google": "https://www.google.com",
        "github": "https://github.com",
        "gmail": "https://mail.google.com",
    }
    url = sites.get(site, None)
    if url:
        talker.say(f"Opening {site}")
        webbrowser.open(url)
    else:
        # if user gave a phrase like 'open example.com'
        if "." in site or site.startswith("http"):
            if not site.startswith("http"):
                site = "https://" + site
            talker.say(f"Opening {site}")
            webbrowser.open(site)
        else:
            talker.say("I don't have that site mapped. I'll search Google for you.")
            webbrowser.open("https://www.google.com/search?q=" + site.replace(" ", "+"))

def handle_time():
    now = datetime.datetime.now()
    s = now.strftime("%I:%M %p on %A, %d %B %Y")
    talker.say("The time is " + s)

def handle_play_music():
    # looks for mp3 files in MUSIC_FOLDER
    try:
        files = [f for f in os.listdir(MUSIC_FOLDER) if f.lower().endswith(('.mp3', '.wav', '.m4a', '.flac'))]
    except Exception as e:
        log(e)
        files = []
    if not files:
        talker.say("I couldn't find music in the configured folder. Please update MUSIC_FOLDER at top of script.")
        return
    choice = os.path.join(MUSIC_FOLDER, files[0])
    talker.say("Playing " + files[0])
    # cross-platform attempt
    try:
        if sys.platform.startswith("darwin"):
            subprocess.Popen(["open", choice])
        elif sys.platform.startswith("win"):
            os.startfile(choice)
        else:
            subprocess.Popen(["xdg-open", choice])
    except Exception as e:
        log("Failed to play music:", e)
        talker.say("I couldn't play the file. Try opening it manually.")

def handle_run_command(shell_cmd):
    talker.say("Running command: " + shell_cmd)
    try:
        result = subprocess.run(shell_cmd, shell=True, capture_output=True, text=True, timeout=30)
        out = result.stdout.strip() or result.stderr.strip()
        if not out:
            talker.say("Command finished.")
        else:
            # speak short output and print full
            talker.say(out[:200])
            print(out)
    except Exception as e:
        log("Command error:", e)
        talker.say("There was an error running the command.")

def handle_greeting():
    talker.say("Hello! I am Jarvis. Say 'jarvis' to wake me or press Enter and type a command.")

# Main command parser
def parse_and_execute(command):
    if not command:
        return
    command = command.lower().strip()
    log("Heard:", command)

    # basic mapping
    if any(x in command for x in ["hello", "hi", "hey"]):
        handle_greeting()
    elif "time" in command or "date" in command:
        handle_time()
    elif command.startswith("wikipedia") or command.startswith("search wikipedia for"):
        # user might say "wikipedia alan turing"
        q = command.replace("wikipedia", "").replace("search", "").replace("for", "").strip()
        if not q:
            talker.say("What should I search on Wikipedia?")
        else:
            handle_search_wikipedia(q)
    elif command.startswith("search ") or command.startswith("google "):
        q = command.split(" ", 1)[1] if " " in command else ""
        webbrowser.open("https://www.google.com/search?q=" + q.replace(" ", "+"))
        talker.say("Searching Google for " + q)
    elif command.startswith("open "):
        site = command.split(" ", 1)[1]
        handle_open_website(site)
    elif "youtube" in command and ("play" in command or "open" in command):
        handle_open_website("youtube")
    elif "play music" in command or ("play" in command and "music" in command):
        handle_play_music()
    elif command.startswith("run "):
        sh = command.split(" ", 1)[1]
        handle_run_command(sh)
    elif "shutdown" in command or "restart" in command:
        talker.say("I won't perform that action automatically. Use system controls if you want to shutdown or restart.")
    elif "quit" in command or "exit" in command or "goodbye" in command:
        talker.say("Goodbye.")
        sys.exit(0)
    else:
        # fallback: open browser search
        talker.say("I can try to search the web for that.")
        webbrowser.open("https://www.google.com/search?q=" + command.replace(" ", "+"))

# Listening loop with wake word
def voice_loop():
    talker.say("Jarvis ready. Say 'jarvis' to wake me up, or press Enter to type.")
    while True:
        phrase = listen_phrase(timeout=6, phrase_time_limit=6)
        if phrase:
            log("Phrase detected:", phrase)
            if WAKE_WORD in phrase:
                talker.say("Yes?")
                # listen for the command
                cmd = listen_phrase(timeout=6, phrase_time_limit=10)
                if not cmd:
                    # fallback to typed input
                    talker.say("I didn't catch that. You can type your command or try speaking again.")
                    cmd = input("Type command (or blank to cancel): ").strip()
                parse_and_execute(cmd)
            else:
                # if user directly said a command without wakeword, optionally accept
                # comment the next two lines if you want strict wakeword only
                if phrase.strip():
                    parse_and_execute(phrase)
        else:
            # timeout happened; continue
            pass

def typed_loop():
    talker.say("Typing mode active. Type 'help' for options.")
    while True:
        try:
            cmd = input("Jarvis> ").strip()
        except (EOFError, KeyboardInterrupt):
            talker.say("Bye.")
            break
        if not cmd:
            continue
        if cmd == "help":
            print("""Commands examples:
- time / date
- wikipedia <query>
- search <query>
- open youtube / open google / open github
- play music
- run <shell command>
- exit / quit
""")
            continue
        parse_and_execute(cmd)

def main():
    # start typed loop in background if microphone available? We'll pick interactive mode:
    mic_ok = sr is not None
    if mic_ok:
        try:
            # ensure microphone works
            with sr.Microphone() as _:
                pass
        except Exception:
            mic_ok = False

    if mic_ok:
        try:
            voice_loop()
        except KeyboardInterrupt:
            talker.say("Exiting. Bye.")
    else:
        talker.say("Microphone not available or SpeechRecognition not installed. Starting typed mode.")
        typed_loop()

if __name__ == "__main__":
    try:
        main()
    except Exception as e:
        log("Fatal error:", e)
        traceback.print_exc()
        talker.say("An error occurred. Check the console for details.")
