import socket
import threading
import tkinter as tk

# ثوابت إعداد الخادم والبروكسي (يمكنك تعديلها حسب الحاجة)
HOST = "localhost"        # ثابت المضيف (غير معروض للمستخدم)
SERVER_PORT = 8080        # ثابت منفذ الخادم
PROXY_HOST = "127.0.0.1"    # ثابت عنوان البروكسي (غير معروض)
PROXY_PORT = 8081         # ثابت منفذ البروكسي

# متغيرات تتبع حالة الخادم
server_thread = None
server_socket = None
server_running = False

def handle_client(client_socket):
    try:
        data = client_socket.recv(4096)
        if not data:
            return

        # إرسال البيانات إلى البروكسي الثابت
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as proxy_socket:
            proxy_socket.connect((PROXY_HOST, PROXY_PORT))
            proxy_socket.sendall(data)
            response = proxy_socket.recv(4096)

        client_socket.sendall(response)
    except Exception as e:
        print(f"Error: {e}")
    finally:
        client_socket.close()

def accept_connections(host, port):
    global server_socket, server_running
    try:
        server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        server_socket.settimeout(1)
        server_socket.bind((host, port))
        server_socket.listen(5)
        server_running = True
        update_status("✅ Server is running...", "green")
        print(f"Server listening on {host}:{port} | Proxy: {PROXY_HOST}:{PROXY_PORT}")

        while server_running:
            try:
                client_socket, addr = server_socket.accept()
                print(f"Accepted connection from {addr}")
                threading.Thread(target=handle_client, args=(client_socket,), daemon=True).start()
            except socket.timeout:
                continue
            except Exception as e:
                if server_running:
                    print(f"Error accepting connection: {e}")
    except Exception as e:
        print(f"Error starting server: {e}")
        server_running = False
        update_status("❌ Server stopped.", "red")
    finally:
        if server_socket:
            server_socket.close()
            server_socket = None

def start_server(host, port):
    global server_thread, server_running
    if server_running:
        return

    server_thread = threading.Thread(target=accept_connections, args=(host, port), daemon=True)
    server_thread.start()
    start_button.config(state=tk.DISABLED)
    stop_button.config(state=tk.NORMAL)

def stop_server():
    global server_socket, server_thread, server_running
    if not server_running:
        return

    try:
        server_running = False
        if server_socket:
            server_socket.close()
        if server_thread:
            server_thread.join(timeout=2)
        update_status("❌ Server stopped.", "red")
        start_button.config(state=tk.NORMAL)
        stop_button.config(state=tk.DISABLED)
    except Exception as e:
        print(f"Error stopping server: {e}")
    finally:
        server_socket = None
        server_thread = None

def on_start_button_click():
    if not server_running:
        start_server(HOST, SERVER_PORT)

def on_stop_button_click():
    if server_running:
        stop_server()

def update_status(message, color):
    status_label.config(text=message, fg=color)

# إعداد واجهة المستخدم
root = tk.Tk()
root.title("QURwep - VPN/Proxy Server")
root.geometry("400x200")

# ضبط النافذة في المنتصف
root.update_idletasks()
width = root.winfo_width()
height = root.winfo_height()
x = (root.winfo_screenwidth() // 2) - (width // 2)
y = (root.winfo_screenheight() // 2) - (height // 2)
root.geometry(f"+{x}+{y}")

# عنوان التطبيق
title_label = tk.Label(root, text="QURwep", font=("Arial", 16, "bold"), fg="blue")
title_label.pack(pady=10)

# إطار للأزرار
button_frame = tk.Frame(root)
button_frame.pack(pady=10)

start_button = tk.Button(button_frame, text="Start Server", command=on_start_button_click, bg="green", fg="white")
start_button.grid(row=0, column=0, padx=10)

stop_button = tk.Button(button_frame, text="Stop Server", command=on_stop_button_click, bg="red", fg="white", state=tk.DISABLED)
stop_button.grid(row=0, column=1, padx=10)

# شريط الحالة
status_label = tk.Label(root, text="❌ Server is stopped.", fg="red", font=("Arial", 10))
status_label.pack(pady=10)

root.mainloop()
