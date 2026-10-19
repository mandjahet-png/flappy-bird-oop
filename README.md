import tkinter as tk
from tkinter import ttk, messagebox
import random
from abc import ABC, abstractmethod
from datetime import datetime

class Config:
    T = "User Manager & Flappy Bird"
    G = 2.5
    J = -9
    S = 4

class AbstractEntity(ABC):
    @abstractmethod
    def get_info(self): pass

class LogMixin:
    def log(self, m): print(f"[LOG] {datetime.now().strftime('%H:%M')} {m}")

class Validator:
    @staticmethod
    def check(t):
        if len(t) > 2:
            return True
        else:
            return False

class User(AbstractEntity):
    def __init__(self, login, fio):
        self.__l = login
        self.__f = fio
        self.__t = datetime.now().strftime("%H:%M")

    @property
    def login(self): return self.__l

    @login.setter
    def login(self, v):
        if Validator.check(v):
            self.__l = v
        else:
            raise ValueError("Bad login")

    @property
    def fio(self): return self.__f

    @fio.setter
    def fio(self, v): 
        self.__f = v

    @property
    def time(self): return self.__t

    def __str__(self): return self.__l
    def __repr__(self): return f"User({self.__l})"
    
    def __eq__(self, o): 
        return self.__l == o.__l
        
    def __lt__(self, o): 
        return self.__l < o.__l
        
    def get_info(self): return "user"

class Admin(User):
    def get_info(self): return "admin"

class GameMap:
    def __init__(self):
        self.__d = {"bg": "skyblue", "pipe": "green", "bird": "yellow"}
    
    def get(self, k): return self.__d.get(k)

class Bird(AbstractEntity):
    def __init__(self, c, x, y):
        self.c = c
        self.v = 0
        self.id = c.create_oval(x, y, x+30, y+30, fill="yellow")
    
    def update(self):
        self.v += Config.G
        self.c.move(self.id, 0, self.v)
    
    def jump(self): 
        self.v = Config.J
        
    def get_info(self): return "bird"

class Pipe(AbstractEntity):
    def __init__(self, c, x, h):
        self.c = c
        self.top = c.create_rectangle(x, 0, x+50, h, fill="green")
        self.bot = c.create_rectangle(x, h+150, x+50, 500, fill="green")
    
    def move(self):
        self.c.move(self.top, -Config.S, 0)
        self.c.move(self.bot, -Config.S, 0)
    
    def get_info(self): return "pipe"

class Engine(LogMixin):
    def __init__(self, canvas, app):
        self.c = canvas
        self.app = app
        self.score = 0
        self.bird = Bird(self.c, 100, 250)
        self.pipes = []
        self.run = True
        self.new_pipe()
        self.loop()

    def new_pipe(self):
        h = random.randint(100, 250)
        self.pipes.append(Pipe(self.c, 600, h))

    def loop(self):
        if not self.run: return
        self.bird.update()
        coords = self.c.coords(self.bird.id)
        y1, y2 = coords[1], coords[3]
        
        if y2 >= 500 or y1 <= 0:
            self.die()
            return

        for p in self.pipes:
            p.move()
            px = self.c.coords(p.top)[0]
            if px < 130 and px + 50 > 100:
                h = self.c.coords(p.top)[3]
                if y1 < h or y2 > h + 150:
                    self.die()
                    return
            if px < -50:
                self.c.delete(p.top)
                self.c.delete(p.bot)
                self.pipes.remove(p)
                self.new_pipe()
                self.score += 1
                self.c.itemconfig(self.app.lbl_score, text=f"Score: {self.score}")

        self.app.root.after(30, self.loop)

    def die(self):
        self.run = False
        self.log(f"Game Over. Score: {self.score}")
        self.c.create_text(300, 250, text="GAME OVER", font=("Arial", 30, "bold"), fill="red")
        self.app.btn_play.config(state="normal")

class MyEntry(tk.Entry):
    def __init__(self, m, t, **kw):
        super().__init__(m, **kw)
        self.t = t
        self.insert(0, t)
        self.bind("<FocusIn>", lambda e: self.delete(0, tk.END) if self.get() == self.t else None)

class MyButton(tk.Button):
    def __init__(self, m, **kw):
        super().__init__(m, **kw)
        self.config(bg="blue", fg="white", font="Arial 9 bold")

class MyTable(ttk.Treeview):
    def __init__(self, m, **kw):
        super().__init__(m, **kw)
        self['columns'] = ("login", "fio", "time")
        self.column("#0", width=0, stretch=tk.NO)
        for c in self['columns']:
            self.column(c, width=100)
            self.heading(c, text=c)
    
    def add(self, u):
        self.insert("", "end", values=(u.login, u.fio, u.time))

class App:
    def __init__(self, r):
        self.root = r
        r.title(Config.T)
        r.geometry("600x600")
        self.users = [
            User("admin", "Ivanov I."), User("user1", "Petrov P."), 
            Admin("root", "Admin A."), User("player", "Sidorov S."), 
            User("guest", "Smirnov S.")
        ]
        self.setup_menu()

    def setup_menu(self):
        self.menu_frame = tk.Frame(self.root)
        self.menu_frame.pack(pady=10, fill="x")

        self.e1 = MyEntry(self.menu_frame, "Login")
        self.e1.pack(side="left", padx=5)
        self.e2 = MyEntry(self.menu_frame, "FIO")
        self.e2.pack(side="left", padx=5)
        
        MyButton(self.menu_frame, text="Add User", command=self.add_user).pack(side="left", padx=5)
        
        self.table = MyTable(self.menu_frame)
        self.table.pack(side="left", fill="x", expand=True, padx=5)
        self.refresh()

        self.btn_start = MyButton(self.root, text="START GAME", command=self.start_game)
        self.btn_start.pack(pady=20)

    def add_user(self):
        l = self.e1.get()
        f = self.e2.get()
        
        if not Validator.check(l):
            messagebox.showerror("Error", "Login must be > 2 chars")
            return
            
        self.users.append(User(l, f))
        self.refresh()
        self.e1.delete(0, tk.END)
        self.e2.delete(0, tk.END)

    def refresh(self):
        self.table.delete(*self.table.get_children())
        for u in sorted(self.users):
            self.table.add(u)

    def start_game(self):
        self.menu_frame.pack_forget()
        self.btn_start.pack_forget()

        self.canvas = tk.Canvas(self.root, width=600, height=500, bg="skyblue")
        self.canvas.pack()
        
        self.lbl_score = self.canvas.create_text(50, 20, text="Score: 0", font="Arial 16")
        
        self.btn_play = MyButton(self.root, text="RESTART", command=self.restart)
        self.btn_play.pack(pady=5)
        self.btn_play.config(state="disabled")
        
        self.root.bind("<space>", lambda e: self.eng.bird.jump() if self.eng.run else None)
        self.eng = Engine(self.canvas, self)

    def restart(self):
        self.canvas.delete("all")
        self.btn_play.config(state="disabled")
        self.lbl_score = self.canvas.create_text(50, 20, text="Score: 0", font="Arial 16")
        self.eng = Engine(self.canvas, self)

if __name__ == '__main__':
    root = tk.Tk()
    app = App(root)
    root.mainloop()
