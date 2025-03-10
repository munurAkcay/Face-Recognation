# thinker
import os
import cv2
import time
import tkinter as tk
from tkinter import messagebox
from simple_facerec import SimpleFacerec
from PIL import Image, ImageTk
from telegram import Bot
from concurrent.futures import ThreadPoolExecutor
import asyncio

# Telegram bot ayarları
TELEGRAM_TOKEN = "YOUR_TELEGRAM_TOKEN"
TELEGRAM_CHAT_ID = "YOUR_TELEGRAM_CHAT_ID"
bot = Bot(TELEGRAM_TOKEN)

USER_DATA_FILE = "user_data.txt"
UNKNOWN_FOLDER = "yabanci"

if not os.path.exists(UNKNOWN_FOLDER):
    os.makedirs(UNKNOWN_FOLDER)

class FaceRecognitionApp:
    def __init__(self, window, window_title):
        self.window = window
        self.window.title(window_title)

        self.sfr = SimpleFacerec()
        self.sfr.load_encoding_images("images/")

        self.video_source = 0
        self.cap = cv2.VideoCapture(self.video_source)

        self.canvas = tk.Canvas(window, width=self.cap.get(3), height=self.cap.get(4))
        self.canvas.pack()

        self.face_label = tk.Label(window, text="")
        self.face_label.pack()

        self.update()

    def update(self):
        ret, frame = self.cap.read()
        if ret:
            face_locations, face_names = self.sfr.detect_known_faces(frame)
            if face_names:
                self.face_label.config(text=f"{face_names[0]} Welcome")
            else:
                self.face_label.config(text="Eşleşme bulunamadı")

            self.photo = self.convert_to_photo_image(frame)
            self.canvas.create_image(0, 0, anchor=tk.NW, image=self.photo)

        self.window.after(10, self.update)

    @staticmethod
    def convert_to_photo_image(frame):
        frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        return tk.PhotoImage(data=cv2.imencode('.png', frame)[1].tobytes())

    def __del__(self):
        if self.cap.isOpened():
            self.cap.release()

class LoginScreen:
    def __init__(self, parent):
        self.window = tk.Toplevel(parent)
        self.window.title("Giriş Ekranı")

        self.canvas = tk.Canvas(self.window, width=400, height=300)
        self.canvas.pack()

        self.cap = cv2.VideoCapture(0)
        self.update_camera()

        tk.Label(self.window, text="Kullanıcı Adı:").pack()
        self.username_entry = tk.Entry(self.window)
        self.username_entry.pack()

        tk.Label(self.window, text="Şifre:").pack()
        self.password_entry = tk.Entry(self.window, show="*")
        self.password_entry.pack()

        self.login_button = tk.Button(self.window, text="Giriş Yap", command=self.login)
        self.login_button.pack()

        self.register_button = tk.Button(self.window, text="Kayıt Ol", command=self.register)
        self.register_button.pack()

    def update_camera(self):
        ret, frame = self.cap.read()
        if ret:
            frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            self.photo = ImageTk.PhotoImage(image=Image.fromarray(frame))
            self.canvas.create_image(0, 0, image=self.photo, anchor=tk.NW)
            self.window.after(10, self.update_camera)

    def login(self):
        username = self.username_entry.get()
        password = self.password_entry.get()
        if self.authenticate(username, password):
            self.window.destroy()
            app_window = tk.Toplevel()  # Yeni bir üst seviye pencere oluştur
            app = FaceRecognitionApp(app_window, "Face Recognition App")
            app_window.mainloop()
        else:
            self.save_unknown_person()
            messagebox.showerror("Hata", "Yanlış kullanıcı adı veya şifre.")

    def authenticate(self, username, password):
        try:
            with open(USER_DATA_FILE, "r") as file:
                for line in file:
                    user, pwd = line.strip().split(",")
                    if user == username and pwd == password:
                        return True
            return False
        except FileNotFoundError:
            return False

    def save_unknown_person(self):
        ret, frame = self.cap.read()
        if ret:
            timestamp = int(time.time())
            filename = os.path.join(UNKNOWN_FOLDER, f"unknown_{timestamp}.jpg")
            cv2.imwrite(filename, frame)
            self.send_telegram_photo(filename)

    def send_telegram_photo(self, filename):
        with ThreadPoolExecutor() as executor:
            loop = asyncio.get_event_loop()
            loop.run_in_executor(executor, self.async_send_photo, filename)

    async def async_send_photo(self, filename):
        with open(filename, "rb") as photo:
            await bot.send_photo(chat_id=TELEGRAM_CHAT_ID, photo=photo, caption="Hatalı giriş denemesi")

    def register(self):
        RegistrationScreen(self.window)

    def __del__(self):
        if self.cap.isOpened():
            self.cap.release()

class RegistrationScreen:
    def __init__(self, parent):
        self.window = tk.Toplevel(parent)
        self.window.title("Kayıt Ekranı")

        tk.Label(self.window, text="Kullanıcı Adı:").pack()
        self.username_entry = tk.Entry(self.window)
        self.username_entry.pack()

        tk.Label(self.window, text="Şifre:").pack()
        self.password_entry = tk.Entry(self.window, show="*")
        self.password_entry.pack()

        self.canvas = tk.Canvas(self.window, width=400, height=300)
        self.canvas.pack()

        self.cap = cv2.VideoCapture(0)
        self.update_camera()

        self.capture_button = tk.Button(self.window, text="Fotoğraf Çek", command=self.capture_image)
        self.capture_button.pack()

        self.register_button = tk.Button(self.window, text="Kayıt Ol", command=self.register)
        self.register_button.pack()

    def update_camera(self):
        ret, frame = self.cap.read()
        if ret:
            frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            self.photo = ImageTk.PhotoImage(image=Image.fromarray(frame))
            self.canvas.create_image(0, 0, image=self.photo, anchor=tk.NW)
            self.window.after(10, self.update_camera)

    def capture_image(self):
        ret, frame = self.cap.read()
        if ret:
            username = self.username_entry.get()
            if not username:
                messagebox.showerror("Hata", "Kullanıcı adı girmelisiniz.")
                return
            cv2.imwrite(f"images/{username}.jpg", frame)

    def register(self):
        username = self.username_entry.get()
        password = self.password_entry.get()
        if not username or not password:
            messagebox.showerror("Hata", "Kullanıcı adı ve şifre boş bırakılamaz.")
            return

        with open(USER_DATA_FILE, "a") as file:
            file.write(f"{username},{password}\n")

        messagebox.showinfo("Başarılı", "Kullanıcı başarıyla kaydedildi.")
        self.window.destroy()

    def __del__(self):
        if self.cap.isOpened():
            self.cap.release()

# Ana pencereyi başlatma
root = tk.Tk()
root.withdraw()  # Ana pencereyi gizle
login_screen = LoginScreen(root)
root.mainloop()
