try:
    import tkinter as tk
    TK_AVAILABLE = True
except Exception:
    tk = None
    TK_AVAILABLE = False
import random
import math
import re
import unicodedata
import sqlite3
import hashlib
import hmac
import secrets
import os
import shutil
import socket
import threading
import subprocess
import sys

W, H = 1100, 720
BG_TOP = "#10132f"
BG_BOTTOM = "#24134b"
WHITE = "#f6f5ff"
MUTED = "#b8b9dd"
CYAN = "#59f0e8"
PINK = "#ff6fcf"
PURPLE = "#a98bff"
GOLD = "#ffd66b"
GREEN = "#75f0ae"
RED = "#ff718d"

INSPIRATIONS = [
    ("Đừng để một câu viral nói thay suy nghĩ của bạn.",
     "Trào lưu có thể truyền cảm hứng; lựa chọn cuối cùng vẫn thuộc về bạn.",
     "Thông điệp Trạm lọc thông tin"),
    ("Bạn có thể hòa nhập mà không đánh mất tiếng nói riêng.",
     "Một ý kiến chân thành đáng quý hơn việc lặp lại chỉ để được công nhận.",
     "Thông điệp Trạm lọc thông tin"),
    ("Trước khi nhấn gửi, hãy tự hỏi: mình muốn truyền đi điều gì?",
     "Một khoảnh khắc suy nghĩ có thể giúp bình luận rõ ràng và phù hợp hơn.",
     "Thông điệp Trạm lọc thông tin"),
    ("Sáng tạo không phải lúc nào cũng khác biệt hoàn toàn; đôi khi là biết chọn lọc và biến đổi.",
     "Học hỏi từ người khác là bình thường. Hãy ghi nhận nguồn và thêm suy nghĩ của mình.",
     "Thông điệp Trạm lọc thông tin"),
    ("Một lượt chia sẻ nhỏ cũng góp phần tạo nên môi trường mạng.",
     "Hãy cân nhắc nội dung, ngữ cảnh và ảnh hưởng trước khi lan truyền.",
     "Thông điệp Trạm lọc thông tin"),
    ("Không phải điều gì phổ biến cũng phù hợp với mọi tình huống.",
     "Cùng một câu nói có thể vui ở nơi này nhưng gây hiểu lầm ở nơi khác.",
     "Thông điệp Trạm lọc thông tin"),
    ("Người bản lĩnh không cần chạy theo mọi trào lưu.",
     "Bạn có quyền tìm hiểu, tham gia, biến tấu hoặc bỏ qua một xu hướng.",
     "Thông điệp Trạm lọc thông tin"),
    ("Lời nói trên mạng vẫn chạm đến cảm xúc thật.",
     "Tránh dùng câu viral để chế giễu, cô lập hoặc làm người khác xấu hổ.",
     "Thông điệp Trạm lọc thông tin"),
    ("Hãy là người tạo ra điều tích cực, không chỉ là người chuyển tiếp.",
     "Một bình luận tử tế, một câu hỏi hay hay một ý tưởng mới đều có thể lan tỏa.",
     "Thông điệp Trạm lọc thông tin"),
    ("Tư duy độc lập bắt đầu từ một câu hỏi nhỏ: Vì sao mình nói điều này?",
     "Đặt câu hỏi không làm bạn kém hòa đồng; nó giúp bạn lựa chọn có chủ đích.",
     "Thông điệp Trạm lọc thông tin"),
    ("Không cần hoàn hảo mới được sáng tạo.",
     "Hãy bắt đầu bằng một câu thật lòng, rõ nghĩa và tôn trọng người đọc.",
     "Thông điệp Trạm lọc thông tin"),
    ("Mạng xã hội tốt hơn khi mỗi người góp một lựa chọn có trách nhiệm.",
     "Bạn không kiểm soát được mọi xu hướng, nhưng có thể lựa chọn cách tham gia.",
     "Thông điệp Trạm lọc thông tin"),
]

class CopyQuest:
    def __init__(self, root, username="", role="user", auth_gate=None):
        self.root = root
        self.username = username
        self.role = role
        self.auth_gate = auth_gate
        self.closed = False
        self.root.title("Trạm lọc thông tin | Tiếp nhận, kiểm chứng và chia sẻ có chọn lọc")
        self.root.geometry(f"{W}x{H}")
        self.root.minsize(900, 620)
        self.canvas = tk.Canvas(root, width=W, height=H, bg=BG_TOP, highlightthickness=0)
        self.canvas.pack(fill="both", expand=True)
        self.w = W
        self.h = H
        self.page = "home"
        self.xp = self.auth_gate.get_progress(self.username)[0] if self.auth_gate else 0
        self.level = 1 + self.xp // 100
        self.streak = 0
        self.badges = set()
        self.quiz_index = 0
        self.quiz_answered = False
        self.scenario_index = 0
        self.scenario_answered = False
        self.inspiration_index = 0
        self.challenge_done = set()
        self.pledge_made = False
        self.particles = []
        self._resize_job = None
        self.root.bind("<Configure>", self.on_resize)
        self.root.bind("<Escape>", lambda e: self.show_home())
        self.make_particles()
        self.draw_home()
        self.animate()

    def on_resize(self, event):
        if event.widget == self.root:
            self.w = max(800, event.width)
            self.h = max(580, event.height)
            if self._resize_job:
                self.root.after_cancel(self._resize_job)
            self._resize_job = self.root.after(120, self.redraw)

    def redraw(self):
        if self.page == "home":
            self.draw_home()
        elif self.page == "definition":
            self.draw_definition()
        elif self.page == "scenario":
            self.draw_scenario()
        elif self.page == "inspire":
            self.draw_inspire()
        elif self.page == "trend":
            self.draw_trend_game()

    def make_particles(self):
        self.particles = []
        for _ in range(75):
            self.particles.append({
                "x": random.randint(0, W), "y": random.randint(0, H),
                "r": random.choice([1, 1, 2, 3]),
                "speed": random.uniform(0.15, 0.75),
                "phase": random.random() * 6.28,
                "color": random.choice([CYAN, PURPLE, PINK, "#6c77c9"])
            })

    def gradient(self):
        self.canvas.delete("all")

        top = (16, 19, 47)
        bottom = (37, 19, 75)
        strips = 90
        for i in range(strips):
            t = i / max(1, strips - 1)
            rgb = tuple(int(top[k] * (1-t) + bottom[k] * t) for k in range(3))
            color = "#%02x%02x%02x" % rgb
            y0 = int(self.h * i / strips)
            y1 = int(self.h * (i + 1) / strips) + 1
            self.canvas.create_rectangle(0, y0, self.w, y1, fill=color, outline=color)

        self.canvas.create_oval(self.w*0.62, -self.h*0.42, self.w*1.12, self.h*0.45,
                                fill="#241d5a", outline="")
        self.canvas.create_oval(-self.w*0.25, self.h*0.58, self.w*0.34, self.h*1.25,
                                fill="#211b53", outline="")
        for p in self.particles:
            self.canvas.create_oval(p["x"], p["y"], p["x"]+p["r"]*2, p["y"]+p["r"]*2,
                                    fill=p["color"], outline="", tags="particle")

    def animate(self):
        if self.closed:
            return
        for p in self.particles:
            p["y"] -= p["speed"]
            p["x"] += math.sin(p["phase"]) * 0.25
            p["phase"] += 0.025
            if p["y"] < -5:
                p["y"] = self.h + random.randint(0, 40)
                p["x"] = random.randint(0, self.w)

        self.canvas.delete("particle")
        for p in self.particles:
            self.canvas.create_oval(p["x"], p["y"], p["x"]+p["r"]*2, p["y"]+p["r"]*2,
                                    fill=p["color"], outline="", tags="particle")
        self.root.after(45, self.animate)

    def base(self):
        self.gradient()
        self.draw_topbar()

    def draw_topbar(self):
        self.canvas.create_text(35, 30, text="✦ TRẠM LỌC THÔNG TIN", anchor="w",
                                fill=WHITE, font=("Arial", 15, "bold"))
        self.canvas.create_text(36, 53, text="TIẾP NHẬN  •  KIỂM CHỨNG  •  CHIA SẺ CÓ CHỌN LỌC",
                                anchor="w", fill=CYAN, font=("Arial", 10, "bold"))

        self.rounded(self.w-245, 19, self.w-143, 55, 14, "#30285f", "#5e55a3")
        self.canvas.create_text(self.w-194, 37, text=f"✦ {self.xp} XP", fill=GOLD,
                                font=("Arial", 10, "bold"))
        self.rounded(self.w-132, 19, self.w-35, 55, 14, "#30285f", "#5e55a3")
        self.canvas.create_text(self.w-83, 37, text=f"LV. {self.level}", fill=CYAN,
                                font=("Arial", 10, "bold"))
        self.canvas.create_text(300, 36, text=f"TÀI KHOẢN: {self.username}", anchor="w", fill=WHITE, font=("Arial", 9, "bold"))
        if self.role == "admin":
            self.button(455, 19, 570, 54, "QUẢN LÝ", self.open_admin, GOLD)
        self.button(580, 19, 670, 54, "ĐĂNG XUẤT", self.logout, PINK)

    def rounded(self, x1, y1, x2, y2, r, fill, outline="", width=1, tags=None):

        pts = [
            x1+r,y1, x2-r,y1, x2-r/2,y1, x2,y1+r/2,
            x2,y2-r, x2,y2-r/2, x2-r/2,y2, x1+r,y2,
            x1+r/2,y2, x1,y2-r/2, x1,y1+r, x1,y1+r/2
        ]
        return self.canvas.create_polygon(pts, smooth=True, fill=fill, outline=outline,
                                          width=width, tags=tags)

    def card(self, x1, y1, x2, y2, accent=PURPLE):

        self.rounded(x1+5, y1+8, x2+5, y2+8, 22, "#0a0d25", "")
        self.rounded(x1, y1, x2, y2, 22, "#25234e", "#514a86", 1)
        self.rounded(x1+12, y1+10, x2-12, y1+14, 2, accent, "")
        self.canvas.create_line(x1+25, y1+22, x2-25, y1+22, fill="#393568", width=1)

    def button(self, x1, y1, x2, y2, label, command, color=PURPLE, subtitle=None):

        self.rounded(x1+3, y1+6, x2+3, y2+6, 14, "#0b0d25", "")
        self.rounded(x1, y1, x2, y2, 14, color, "")
        self.rounded(x1+3, y1+3, x2-3, y1+8, 3, "#ffffff", "")
        if subtitle:
            self.canvas.create_text((x1+x2)/2, (y1+y2)/2-5, text=label,
                                    fill="#171735", font=("Arial", 11, "bold"))
            self.canvas.create_text((x1+x2)/2, (y1+y2)/2+13, text=subtitle,
                                    fill="#28224b", font=("Arial", 10))
        else:
            self.canvas.create_text((x1+x2)/2, (y1+y2)/2, text=label,
                                    fill="#171735", font=("Arial", 10, "bold"))
        self.canvas.tag_bind(self.canvas.find_all()[-1], "<Button-1>", lambda e: command())

        items = self.canvas.find_overlapping(x1, y1, x2, y2)
        for item in items:
            self.canvas.tag_bind(item, "<Button-1>", lambda e, fn=command: fn())
            self.canvas.tag_bind(item, "<Enter>", lambda e: self.canvas.config(cursor="hand2"))
            self.canvas.tag_bind(item, "<Leave>", lambda e: self.canvas.config(cursor=""))

    def open_admin(self):
        if self.role == "admin" and self.auth_gate:
            self.auth_gate.open_admin_panel()

    def logout(self):
        self.closed = True
        if self.canvas.winfo_exists():
            self.canvas.destroy()
        self.root.unbind("<Configure>")
        self.root.unbind("<Escape>")
        if self.auth_gate:
            self.auth_gate.show_login()

    def save_progress(self):
        if self.auth_gate and self.username:
            self.auth_gate.save_progress(self.username, self.xp, self.level)

    def show_home(self):
        self.page = "home"
        self.draw_home()

    def draw_home(self):
        self.page = "home"
        self.base()
        cx = self.w * 0.5

        self.canvas.create_text(cx, 104, text="MẠNG XÃ HỘI LÀ SÂN CHƠI CỦA BẠN",
                                fill=CYAN, font=("Arial", 10, "bold"))
        self.canvas.create_text(cx, 145, text="Tiếp nhận thông tin.", fill=WHITE,
                                font=("Arial", 31, "bold"))
        self.canvas.create_text(cx, 183, text="Kiểm chứng trước khi chia sẻ.", fill=PINK,
                                font=("Arial", 25, "bold"))
        self.canvas.create_text(cx, 216,
            text="Tiếp nhận, kiểm chứng và chia sẻ thông tin có chọn lọc.",
            fill=MUTED, font=("Arial", 10))

        self.draw_crystal(self.w-165, 154, 1.0)
        self.draw_crystal(155, 173, 0.58)

        px1, py1, px2, py2 = 70, 252, self.w-70, 325
        self.card(px1, py1, px2, py2, CYAN)
        self.canvas.create_text(px1+24, py1+25, text="HÀNH TRÌNH CỦA BẠN",
                                anchor="w", fill=WHITE, font=("Arial", 10, "bold"))
        self.canvas.create_text(px1+24, py1+51,
                                text=f"Cấp {self.level}   •   {self.xp} XP   •   {len(self.badges)} huy hiệu đã mở",
                                anchor="w", fill=MUTED, font=("Arial", 10))
        bar_x1, bar_y = px1+420, py1+43
        bar_x2 = px2-28
        self.rounded(bar_x1, bar_y, bar_x2, bar_y+12, 6, "#111632", "")
        progress = (self.xp % 100) / 100
        if progress > 0:
            self.rounded(bar_x1, bar_y, bar_x1+(bar_x2-bar_x1)*progress, bar_y+12, 6, CYAN, "")
        self.canvas.create_text(bar_x1, bar_y-14, text="TIẾN ĐỘ LÊN CẤP",
                                anchor="w", fill="#b7b8e4", font=("Arial", 10, "bold"))

        gap = 14
        margin = 55
        card_w = (self.w - 2*margin - 3*gap) / 4
        y1, y2 = 352, 530
        missions = [
            ("01", "ĐỊNH NGHĨA", "COPYPASTA LÀ GÌ?", "Khám phá định nghĩa qua cây tri thức", CYAN, self.start_definition),
            ("02", "MÊ CUNG", "NGÔN TỪ", "Mỗi bước mở ra thử thách mới", PINK, self.start_scenario),
            ("03", "KIỂM TRA CÂU NÓI", "COPYPASTA?", "Nhập câu nói để xem xét dấu hiệu", GOLD, self.start_trend_game),
            ("04", "CAM KẾT 7 NGÀY", "MẠNG LÀNH MẠNH", "Thử thách mỗi ngày và tuyên ngôn cá nhân", GREEN, self.show_inspire),
        ]
        for i, (num, title, sub, desc, color, fn) in enumerate(missions):
            x1 = margin + i*(card_w+gap)
            x2 = x1 + card_w
            self.card(x1, y1, x2, y2, color)
            self.canvas.create_oval(x1+16, y1+25, x1+55, y1+64, fill="#353260", outline=color, width=2)
            self.canvas.create_text(x1+35, y1+45, text=num, fill=color, font=("Arial", 11, "bold"))
            self.canvas.create_text(x1+13, y1+82, text=title, anchor="w", fill=WHITE,
                                    font=("Arial", 11, "bold"))
            self.canvas.create_text(x1+13, y1+101, text=sub, anchor="w", fill=color,
                                    font=("Arial", 10, "bold"))
            self.canvas.create_text(x1+13, y1+124, text=desc, anchor="w", fill=MUTED,
                                    font=("Arial", 10), width=card_w-24, justify="left")
            self.button(x1+10, y2-36, x2-10, y2-12, "MỞ TRẠM  ↗", fn, color)

        self.canvas.create_text(self.w/2, self.h-35,
            text="Mỗi lựa chọn là một cơ hội học hỏi • Không phán xét tiếng lóng — hãy hiểu trước khi dùng",
            fill="#a8a9d3", font=("Arial", 10))

    def draw_crystal(self, x, y, s):

        self.canvas.create_oval(x-45*s, y+37*s, x+45*s, y+54*s,
                                fill="#11112f", outline="")
        pts1 = [x, y-45*s, x+35*s, y-7*s, x+18*s, y+35*s, x-18*s, y+35*s, x-35*s, y-7*s]
        self.canvas.create_polygon(pts1, fill="#6d5ce7", outline="#b6a8ff", width=2)
        self.canvas.create_polygon([x,y-45*s, x+35*s,y-7*s, x,y+4*s],
                                   fill="#c0b5ff", outline="")
        self.canvas.create_polygon([x,y-45*s, x-35*s,y-7*s, x,y+4*s],
                                   fill="#8c7cff", outline="")
        self.canvas.create_polygon([x,y+4*s, x+35*s,y-7*s, x+18*s,y+35*s],
                                   fill="#5144bc", outline="")
        self.canvas.create_polygon([x,y+4*s, x-35*s,y-7*s, x-18*s,y+35*s],
                                   fill="#7566ee", outline="")
        self.canvas.create_line(x-20*s,y-7*s,x+20*s,y-7*s,fill="#e5dfff",width=1)

    def start_definition(self):
        self.page = "definition"
        self.definition_open = [False, False, False]
        self.draw_definition()

    def toggle_definition(self, index):
        self.definition_open[index] = not self.definition_open[index]
        self.draw_definition()

    def draw_definition(self):
        self.page = "definition"
        self.base()
        self.canvas.create_text(self.w/2, 78, text="ỨNG DỤNG 01  •  CÂY TRI THỨC COPYPASTA",
                                fill=CYAN, font=("Arial", 15, "bold"))
        self.canvas.create_text(self.w/2, 108, text="Chạm vào từng nhánh để khám phá định nghĩa",
                                fill=WHITE, font=("Arial", 11))

        cx = self.w/2
        self.rounded(cx-230, 130, cx+230, 184, 18, "#176b55", "#75f0ae", 2)
        self.canvas.create_text(cx, 157, text="COPYPASTA = SAO CHÉP + LẶP LẠI",
                                fill=WHITE, font=("Arial", 14, "bold"))
        self.canvas.create_line(cx, 184, cx, 205, fill=CYAN, width=3)
        positions = [self.w*0.18, self.w*0.50, self.w*0.82]
        colors = [CYAN, GOLD, PURPLE]
        headings = ["1. NỘI DUNG", "2. HÀNH VI", "3. LAN TRUYỀN"]
        summaries = ["Câu nói / bình luận", "Sao chép / biến đổi", "Cộng đồng sử dụng"]
        details = [
            "Những câu nói, đoạn văn, bình luận hoặc thông điệp xuất hiện nhiều lần trên mạng.",
            "Người dùng sao chép nguyên văn hoặc biến đổi một phần để dùng trong tình huống mới.",
            "Nội dung được tiếp nhận và lan truyền qua bình luận, chia sẻ và tương tác.",
        ]
        for i, x in enumerate(positions):
            self.canvas.create_line(cx, 205, x, 226, fill=colors[i], width=2)
            self.rounded(x-145, 226, x+145, 286, 14, "#292653", colors[i], 2)
            self.canvas.create_text(x, 246, text=headings[i], fill=colors[i], font=("Arial", 11, "bold"))
            self.canvas.create_text(x, 267, text=summaries[i], fill=WHITE, font=("Arial", 10))
            self.button(x-125, 300, x+125, 333,
                        "ẨN NỘI DUNG" if self.definition_open[i] else "KHÁM PHÁ NHÁNH  +",
                        lambda idx=i: self.toggle_definition(idx), colors[i])
            if self.definition_open[i]:
                self.rounded(x-150, 343, x+150, 420, 12, "#201e45", colors[i], 1)
                self.canvas.create_text(x, 381, text=details[i], fill=WHITE,
                                        font=("Arial", 10), width=270, justify="center")

        self.rounded(95, 448, self.w-95, 520, 16, "#25234e", "#514a86", 1)
        self.canvas.create_text(self.w/2, 469, text="GHI NHỚ NHANH", fill=GOLD,
                                font=("Arial", 11, "bold"))
        self.canvas.create_text(self.w/2, 494,
                                text="Nội dung được sao chép → lặp lại → lan truyền trong cộng đồng",
                                fill=WHITE, font=("Arial", 11), width=self.w-220)
        self.canvas.create_text(self.w/2, 548,
                                text="Copypasta không phải lúc nào cũng tiêu cực; cần xem xét ý nghĩa và ngữ cảnh.",
                                fill=MUTED, font=("Arial", 10), width=self.w-180)
        self.button(32, self.h-55, 160, self.h-24, "← TRANG CHỦ", self.show_home, PURPLE)

    def start_scenario(self):
        self.page = "scenario"
        self.maze_level = 1
        self.maze_pos = [0, 0]
        self.maze_lives = 3
        self.maze_pending = None
        self.maze_quiz_active = False
        self.maze_won = False
        self.maze_lost = False
        self.maze_feedback = "Mỗi bước đi cần trả lời một câu hỏi. Tránh quái vật và đến cửa!"
        self.maze_answer_feedback = ""
        self.maze_last_correct = False
        self.maze_show_answer_marks = False
        self.maze_last_selected = None
        self.maze_maps = [
            ["S..#....", ".#.#.##.", ".#......", ".####.#.", "....#...", ".##...#.", "...##...", "......E."],
            ["S...#...", "....#...", "........", ".####...", "........", "...####.", "........", ".......E"],
            ["S.#.....", "..#.....", "........", ".####...", "........", "...####.", "........", ".......E"]
        ]
        self.maze_monsters = [[2, 5], [4, 6], [6, 5]]
        self.maze_questions = []
        self.maze_question_banks = [
            [
                ("Nhiều bình luận cùng chép nguyên một câu vô nghĩa đang viral. Dấu hiệu chính là gì?", ["Sao chép và lặp lại máy móc", "Mọi người đều hiểu câu đó", "Đó chắc chắn là thông tin hữu ích"], 0, "Copypasta trong đề tài này nhấn mạnh việc sao chép máy móc và lặp lại câu viral, kể cả khi câu đó vô nghĩa."),
                ("Một câu trend không có nghĩa rõ ràng nhưng bạn bè liên tục nhắc lại. Nên làm gì?", ["Lặp lại ngay cho giống mọi người", "Dừng lại, tìm hiểu và tự quyết định", "Spam câu đó nhiều hơn"], 1, "Không cần chạy theo mọi câu đang thịnh hành; hãy hiểu trước khi dùng."),
                ("Bạn nhận ra mình đang gõ lại một câu viral theo thói quen. Điều nào hữu ích?", ["Tự hỏi mình muốn nói gì", "Gửi nhanh hơn", "Chép thêm vài lần"], 0, "Tạm dừng giúp bạn nhận ra mình có đang diễn đạt ý thật của mình không."),
                ("Điều nào gần nhất với định nghĩa Copypasta của đề tài?", ["Tự viết một nhận xét riêng", "Chia sẻ một thông tin có nguồn", "Sao chép và lặp lại câu nói hoặc tiếng lóng viral một cách máy móc"], 2, "Trọng tâm là hành vi sao chép, lặp lại nhiều lần trên mạng xã hội."),
                ("Một câu nói chỉ có ý nghĩa trong video gốc nhưng bị dùng ở mọi nơi. Vấn đề có thể là gì?", ["Dùng sai ngữ cảnh", "Chữ quá dài", "Không có biểu tượng cảm xúc"], 0, "Một câu có thể không còn phù hợp khi tách khỏi ngữ cảnh ban đầu."),
                ("Bạn thấy hàng loạt bình luận chỉ lặp lại một câu vô tri. Cách phản hồi nào chủ động hơn?", ["Chép lại cho đủ xu hướng", "Viết điều mình thật sự muốn nói", "Dán câu đó vào mọi bài đăng"], 1, "Tự diễn đạt giúp bình luận có ý nghĩa và phù hợp với cuộc trò chuyện."),
                ("Câu viral được lặp lại nhiều lần có nghĩa là ai cũng hiểu nó không?", ["Có, chắc chắn", "Đúng vì nhiều lượt thích", "Không; người dùng có thể lặp lại mà chưa hiểu"], 2, "Mức độ phổ biến không đảm bảo người dùng hiểu ý nghĩa."),
                ("Khi không hiểu tiếng lóng đang hot, bạn nên…", ["Tìm hiểu nghĩa và cách dùng", "Dùng ở mọi tình huống", "Lặp lại để khỏi bị bỏ lại"], 0, "Hiểu nghĩa và ngữ cảnh trước khi sử dụng."),
                ("Điều gì cho thấy một bình luận có suy nghĩ riêng?", ["Chép nguyên mẫu đang nổi", "Diễn đạt ý kiến bằng lời của mình", "Lặp lại dù không liên quan"], 1, "Mục tiêu không phải cấm trend mà là giữ khả năng tự diễn đạt."),
                ("Một câu vô nghĩa được lặp lại dưới nhiều bài đăng. Bạn nên cân nhắc…", ["Số lần có thể spam", "Cách viết giống nhất", "Nó có liên quan và giúp cuộc trò chuyện không"], 2, "Hãy cân nhắc sự liên quan và ý nghĩa của bình luận."),
                ("Copypasta có đồng nghĩa với mọi tiếng lóng không?", ["Không; trọng tâm là sao chép/lặp lại máy móc", "Có, mọi tiếng lóng đều xấu", "Có, mọi từ mới đều phải tránh"], 0, "Đề tài không đánh đồng tiếng lóng với Copypasta."),
                ("Trước khi gửi một câu viral, hãy tự hỏi…", ["Có bao nhiêu người đã gửi?", "Mình có hiểu và có thật sự muốn nói điều này không?", "Có thể lặp lại mấy lần?"], 1, "Tự kiểm tra giúp giảm việc lặp lại theo thói quen.")
            ],
            [
                ("Một câu viral được sao chép nguyên văn ở nhiều nơi. Đó là dấu hiệu của…", ["Sáng tạo độc lập", "Trao đổi sâu sắc", "Sự lặp lại theo mẫu"], 2, "Nhiều bản sao giống nhau có thể là dấu hiệu Copypasta."),
                ("Khi câu trend bị tách khỏi ngữ cảnh ban đầu, điều gì dễ xảy ra?", ["Câu nói có thể trở nên khó hiểu hoặc không phù hợp", "Ai cũng hiểu chính xác", "Nội dung tự động có ý nghĩa"], 0, "Ngữ cảnh ảnh hưởng đến ý nghĩa của câu nói."),
                ("Bạn định chép câu đang hot chỉ để có bình luận. Nên thay bằng…", ["Lặp lại câu đó", "Một ý kiến ngắn của riêng mình", "Nhiều bản sao giống nhau"], 1, "Một câu đơn giản nhưng là ý của mình vẫn có giá trị."),
                ("Dấu hiệu nào không đủ để kết luận một câu là Copypasta?", ["Nhiều bản sao gần giống nhau", "Lặp lại nguyên văn nhiều lần", "Chỉ thấy một người dùng tiếng lóng"], 2, "Một từ lóng đơn lẻ không tự động là Copypasta."),
                ("Nếu bạn không hiểu câu nói đang lan truyền, cách tốt nhất là…", ["Tìm hiểu trước khi dùng", "Dùng càng nhiều càng tốt", "Coi lượt thích là ý nghĩa"], 0, "Đừng nhầm độ phổ biến với sự hiểu biết."),
                ("Vì sao người trẻ có thể lặp lại câu viral một cách máy móc?", ["Vì câu đó luôn đúng", "Do quen thuộc, bắt trend hoặc muốn hòa nhập", "Vì mọi câu đều có nghĩa sâu sắc"], 1, "Thói quen và mong muốn theo kịp xu hướng có thể góp phần tạo ra hành vi lặp lại."),
                ("Cách nào giúp biến trend thành giao tiếp có ý thức?", ["Chép đúng từng chữ", "Dùng bất kể tình huống", "Hiểu nội dung rồi thêm góc nhìn riêng"], 2, "Bạn có thể tham gia xu hướng mà không bỏ qua suy nghĩ của mình."),
                ("Một câu đùa đang viral nhưng không liên quan bài đăng. Điều nên làm là…", ["Không nhất thiết phải dùng; hãy bình luận đúng chủ đề", "Lặp lại nhiều lần", "Dán vào mọi cuộc trò chuyện"], 0, "Bình luận phù hợp giúp cuộc trò chuyện rõ ràng hơn."),
                ("Điều gì khác giữa chia sẻ có suy nghĩ và lặp lại máy móc?", ["Số lượng emoji", "Bạn hiểu nội dung và có lý do để chia sẻ", "Tốc độ gửi bình luận"], 1, "Mấu chốt là sự hiểu biết và chủ đích."),
                ("Nếu thấy một câu vô tri lặp đi lặp lại, phản ứng có ích là…", ["Cố gắng lặp giống hơn", "Tự động dùng theo", "Nhận diện thói quen và cân nhắc có nên tiếp tục"], 2, "Nhận biết hành vi là bước đầu để lựa chọn."),
                ("Tiếng lóng có thể dùng phù hợp khi…", ["Hiểu nghĩa và dùng đúng ngữ cảnh", "Dùng ở mọi nơi", "Không cần biết người đọc hiểu không"], 0, "Tiếng lóng không phải vấn đề tự thân; cách dùng mới quan trọng."),
                ("Mục tiêu của Trạm lọc thông tin là…", ["Cấm mọi xu hướng", "Khuyến khích hiểu trước khi lặp lại máy móc", "Bắt tất cả bình luận phải giống nhau"], 1, "Ứng dụng hướng đến nhận thức và khả năng tự diễn đạt.")
            ],
            [
                ("Copypasta theo đề tài này tập trung vào…", ["Mọi loại tiếng lóng", "Chỉ nội dung bạo lực", "Sao chép máy móc và lặp lại câu viral"], 2, "Trọng tâm là lặp lại câu nói, bình luận hoặc tiếng lóng đang viral mà không suy nghĩ."),
                ("Một câu vô nghĩa được lặp lại hàng chục lần. Bạn có thể tự hỏi…", ["Mình đang muốn nói gì qua câu này?", "Làm sao để gửi thêm?", "Ai đã spam nhiều nhất?"], 0, "Tự hỏi mục đích giúp bạn chủ động hơn."),
                ("Cùng một câu nói được dùng trong tình huống khác hẳn ban đầu. Điều cần kiểm tra là…", ["Màu chữ", "Ngữ cảnh và ý nghĩa", "Số người theo dõi"], 1, "Một câu có thể thay đổi cách hiểu theo tình huống."),
                ("Nhiều lượt thích có chứng minh câu trend có ý nghĩa không?", ["Có, luôn luôn", "Có nếu đủ viral", "Không; cần xem nội dung và ngữ cảnh"], 2, "Độ phổ biến không thay thế việc hiểu nội dung."),
                ("Bạn muốn tham gia trend mà không lặp lại máy móc. Hãy…", ["Biến tấu và nói rõ ý của mình", "Chép nguyên văn mọi nơi", "Bình luận dù không hiểu"], 0, "Tham gia xu hướng vẫn có thể đi cùng sáng tạo."),
                ("Điều nào là ví dụ rõ nhất của lặp lại máy móc?", ["Viết ý kiến cá nhân", "Dán cùng một câu viral vào nhiều bài không liên quan", "Đọc nghĩa trước khi dùng"], 1, "Sao chép cùng một câu bất kể ngữ cảnh là ví dụ gần với định nghĩa đề tài."),
                ("Khi một câu viral không liên quan cuộc trò chuyện, bạn nên…", ["Dùng vì mọi người đều dùng", "Gửi nhiều lần", "Chọn cách phản hồi liên quan hơn"], 2, "Hãy để bình luận phục vụ cuộc trò chuyện."),
                ("Tự diễn đạt suy nghĩ giúp…", ["Giữ tiếng nói riêng và tăng ý nghĩa trao đổi", "Đảm bảo chắc chắn nổi tiếng", "Khiến mọi người phải đồng ý"], 0, "Mục tiêu là diễn đạt rõ ý, không phải tìm kiếm sự chú ý bằng mọi giá."),
                ("Một người dùng từ lóng đúng ngữ cảnh có tự động là Copypasta không?", ["Có, mọi lúc", "Không nhất thiết; cần xem có sao chép/lặp lại máy móc không", "Có nếu người khác hiểu"], 1, "Dùng tiếng lóng không đồng nghĩa với hành vi Copypasta."),
                ("Cách nào thể hiện đã hiểu câu viral trước khi dùng?", ["Chỉ xem lượt chia sẻ", "Chép lại ngay", "Có thể giải thích ý nghĩa và vì sao dùng ở đây"], 2, "Nếu không thể giải thích ý nghĩa, hãy tìm hiểu thêm."),
                ("Một câu được lặp lại rất nhiều nhưng mỗi lần đều đúng chủ đề. Ta nên…", ["Xem cả nội dung, mục đích và ngữ cảnh trước khi đánh giá", "Kết luận ngay là xấu", "Cấm mọi người dùng"], 0, "Không phải mọi sự lặp lại đều tiêu cực; cần đánh giá cụ thể."),
                ("Thông điệp chính của mê cung là…", ["Không bao giờ dùng tiếng lóng", "Dừng lại, hiểu nghĩa và lựa chọn có chủ đích", "Luôn theo số đông"], 1, "Hiểu trước khi lặp lại giúp dùng mạng xã hội có ý thức.")
            ]
        ]

        self.maze_question_banks = [
            [
                ("Nhiều tài khoản đăng nguyên văn cùng một câu. Đây là dấu hiệu gì?", ["Có thể là copypasta", "Chắc chắn là tin thật", "Không thể là nội dung lan truyền"], 0, "Lặp lại nguyên văn là một dấu hiệu cần chú ý."),
                ("Bạn chưa hiểu một câu đang viral. Nên làm gì?", ["Chép ngay", "Tìm hiểu nghĩa và ngữ cảnh", "Spam câu đó"], 1, "Hiểu trước khi dùng giúp bạn lựa chọn có chủ đích."),
                ("Cách nào thể hiện tiếng nói riêng?", ["Chép nguyên văn", "Lặp lại theo số đông", "Tự diễn đạt suy nghĩ"], 2, "Tự diễn đạt giúp thể hiện quan điểm của mình."),
                ("Copypasta có phải lúc nào cũng xấu?", ["Có, luôn xấu", "Không, cần xét ngữ cảnh", "Càng lặp càng tốt"], 1, "Đánh giá nội dung và ngữ cảnh thay vì vội kết luận."),
                ("Một câu lặp lại đang dùng để trêu bạn học. Nên làm gì?", ["Hùa theo", "Chia sẻ thêm", "Không tiếp tay cho việc trêu chọc"], 2, "Đừng dùng câu viral để làm tổn thương người khác."),
                ("Điều gì quan trọng trước khi chia sẻ câu viral?", ["Hiểu ý nghĩa", "Số lượt thích", "Tốc độ bình luận"], 0, "Nên hiểu điều mình truyền đi."),
                ("Dấu hiệu nào có thể gợi ý nội dung bị sao chép?", ["Mỗi người viết khác nhau", "Nhiều bình luận giống hệt", "Một câu hỏi riêng"], 1, "Sự lặp lại gần như nguyên văn là dấu hiệu, chưa phải kết luận tuyệt đối."),
                ("Bạn muốn tham gia trào lưu nhưng giữ dấu ấn riêng. Hãy…", ["Chép y hệt", "Spam liên tục", "Biến tấu có suy nghĩ"], 2, "Có thể tham gia xu hướng mà vẫn sáng tạo."),
                ("Một câu nói nổi tiếng xuất hiện nhiều lần có chắc là copypasta không?", ["Chưa chắc, cần ngữ cảnh", "Chắc chắn 100%", "Chắc chắn là xấu"], 0, "Không nên kết luận chỉ từ một dấu hiệu."),
                ("Trước khi bình luận, bạn nên cân nhắc…", ["Chỉ số lượt xem", "Tác động đến người đọc", "Viết càng nhanh càng tốt"], 1, "Lời nói trực tuyến vẫn ảnh hưởng đến cảm xúc thật."),
                ("Học hỏi câu nói của người khác có thể chấp nhận khi…", ["Luôn giả làm tác giả", "Không cần hiểu nghĩa", "Biết nguồn và dùng phù hợp"], 2, "Tôn trọng nguồn và ngữ cảnh là điều quan trọng."),
                ("Nếu thấy nội dung lặp lại có vẻ xúc phạm, nên…", ["Không lan truyền và cân nhắc báo cáo", "Chia sẻ cho vui", "Gắn tên người bị nhắm đến"], 0, "Không khuếch đại nội dung có thể gây tổn thương.")
            ],
            [
                ("Một meme được nhiều người chia sẻ có tự động là copypasta không?", ["Có, tất cả đều vậy", "Không nhất thiết; xem dạng nội dung và ngữ cảnh", "Chắc chắn là tin giả"], 1, "Meme và copypasta có thể giao nhau nhưng không đồng nghĩa."),
                ("Câu bình luận giống hệt xuất hiện ở nhiều video. Bạn nên nghĩ gì?", ["Có thể đang được sao chép", "Ai cũng tự nghĩ ra chắc chắn", "Không cần quan tâm ngữ cảnh"], 0, "Đây là dấu hiệu đáng xem xét, không phải bằng chứng tuyệt đối."),
                ("Khi dùng tiếng lóng mới, lựa chọn tốt là…", ["Dùng dù chưa hiểu", "Dùng để chọc người khác", "Tìm hiểu ý nghĩa trước"], 2, "Hiểu nghĩa giúp tránh hiểu lầm."),
                ("Số đông dùng một câu có khiến câu đó luôn phù hợp?", ["Có", "Không, còn tùy tình huống", "Chỉ cần nhiều lượt thích"], 1, "Mức độ phổ biến không quyết định tính phù hợp."),
                ("Nếu muốn phản hồi video bằng ý kiến riêng, hãy…", ["Viết điều mình thật sự nghĩ", "Dán câu không liên quan", "Lặp câu đang hot"], 0, "Bình luận có suy nghĩ tạo ra trao đổi có ý nghĩa."),
                ("Lặp lại câu đùa nhắm vào một bạn có thể…", ["Không bao giờ ảnh hưởng", "Làm bạn ấy tổn thương", "Luôn giúp tình bạn tốt hơn"], 1, "Hãy quan tâm đến người bị nhắm đến."),
                ("Dấu hiệu nào cần xem xét cùng với việc lặp lại?", ["Màu nền ứng dụng", "Số chữ trong tên tài khoản", "Nguồn, mục đích và ngữ cảnh"], 2, "Cần nhiều thông tin hơn để hiểu một hiện tượng."),
                ("Một câu copypasta có thể được dùng để…", ["Giải trí hoặc gây phiền tùy ngữ cảnh", "Luôn có hại", "Luôn đúng"], 0, "Tác động phụ thuộc nội dung và cách sử dụng."),
                ("Bạn bè rủ spam bình luận. Bạn có thể…", ["Làm theo không suy nghĩ", "Từ chối và viết bình luận phù hợp", "Tag người khác hàng loạt"], 1, "Bạn có quyền lựa chọn cách tham gia."),
                ("Khi không chắc một câu có gây xúc phạm không, nên…", ["Đăng trước tính sau", "Thêm nhiều biểu tượng", "Dừng lại và tìm hiểu"], 2, "Dừng lại để kiểm tra có thể tránh hiểu lầm."),
                ("Điều nào giúp giảm việc lặp lại máy móc?", ["Đọc, hiểu và tự diễn đạt", "Chỉ nhìn lượt thích", "Sao chép nhanh hơn"], 0, "Hiểu nội dung giúp bạn phản hồi chủ động."),
                ("Nhận diện copypasta nên dựa vào…", ["Một từ đơn lẻ", "Mẫu lặp và ngữ cảnh", "Số người theo dõi"], 1, "Kết hợp dấu hiệu lặp với ngữ cảnh sẽ thận trọng hơn.")
            ],
            [
                ("Một cụm từ lặp lại khắp bình luận có thể là…", ["Bằng chứng mọi người đồng ý", "Chắc chắn thông tin chính xác", "Nội dung đang lan truyền theo mẫu"], 2, "Sự phổ biến không chứng minh nội dung đúng."),
                ("Trước khi chia sẻ câu đang thịnh hành, nên hỏi…", ["Mình có hiểu và thấy phù hợp không?", "Có thể spam mấy lần?", "Làm sao để nhiều người bắt chước?"], 0, "Tự kiểm tra giúp bạn quyết định có trách nhiệm."),
                ("Nếu một câu viral xúc phạm nhóm người nào đó, nên…", ["Lặp lại cho vui", "Không khuếch đại nội dung đó", "Gửi cho nhiều người"], 1, "Không tiếp tay cho lời nói gây tổn thương."),
                ("Điều gì giúp phân biệt học hỏi và sao chép máy móc?", ["Không đọc nội dung", "Chỉ thay một dấu câu", "Hiểu, ghi nhận nguồn và thêm ý riêng"], 2, "Học hỏi có suy nghĩ khác với sao chép thiếu cân nhắc."),
                ("Nội dung bị lặp lại có luôn mang ý nghĩa xấu?", ["Không, cần xem nội dung", "Có, luôn luôn", "Chỉ phụ thuộc số lượt xem"], 0, "Không nên gán nhãn tiêu cực cho mọi nội dung lặp."),
                ("Khi thấy nhiều bình luận giống nhau, nên kết luận…", ["Đó chắc chắn là bot", "Cần thêm dấu hiệu và ngữ cảnh", "Tất cả người dùng đều sai"], 1, "Không thể xác định nguyên nhân chỉ từ một dấu hiệu."),
                ("Một cách phản hồi tích cực là…", ["Chép câu để hòa nhập", "Chế giễu người khác", "Viết ý kiến chân thành, tôn trọng"], 2, "Tôn trọng và chân thành tạo môi trường trao đổi tốt hơn."),
                ("Nếu chưa biết nguồn gốc một câu viral, hãy…", ["Tìm hiểu thêm trước khi khẳng định", "Bịa ra nguồn", "Nói chắc chắn dù chưa biết"], 0, "Tránh khẳng định khi chưa đủ thông tin."),
                ("Điều gì không nên dùng để đánh giá một câu nói?", ["Nội dung và ngữ cảnh", "Tác động tới người đọc", "Chỉ dựa vào độ nổi tiếng"], 2, "Độ nổi tiếng không thay thế đánh giá nội dung."),
                ("Khi bạn tự viết một bình luận, điều đó giúp…", ["Bắt buộc mọi người đồng ý", "Thể hiện góc nhìn riêng", "Đảm bảo được nhiều lượt thích"], 1, "Giá trị nằm ở sự rõ ràng và chân thành."),
                ("Một câu nói lặp lại nhiều lần nên được xem xét…", ["Trong ngữ cảnh xuất hiện", "Chỉ theo số lượt chia sẻ", "Theo tên người đăng"], 0, "Ngữ cảnh giúp hiểu cách câu nói được sử dụng."),
                ("Mục tiêu của trò chơi này là…", ["Cấm mọi tiếng lóng", "Kết luận mọi câu viral là xấu", "Tăng khả năng suy nghĩ trước khi lặp lại"], 2, "Mục tiêu là lựa chọn có ý thức, không phải cấm trào lưu.")
            ]
        ]
        self.maze_cell_questions = {}
        for level_idx, bank in enumerate(self.maze_question_banks):
            grid = self.maze_maps[level_idx]
            cells = [(r, c) for r, row in enumerate(grid) for c, ch in enumerate(row)
                     if ch not in "#S" and (r, c) != (len(grid)-1, len(row)-1)]
            random.shuffle(cells)
            mapping = {}

            scenario_prefixes = [
                "Trong phần bình luận TikTok, ",
                "Khi đọc bài đăng Facebook, ",
                "Trong nhóm chat lớp, ",
                "Khi xem một video đang thịnh hành, ",
                "Ở một cuộc trò chuyện trực tuyến, ",
                "Khi thấy bạn bè bắt trend, ",
                "Trong một chuỗi bình luận, ",
                "Khi chuẩn bị đăng phản hồi, ",
                "Ở một cộng đồng mạng, ",
                "Khi gặp một câu nói viral, ",
                "Trong phần chia sẻ của bạn bè, ",
                "Khi lướt qua nhiều bài đăng, ",
                "Khi một câu nói xuất hiện liên tục, ",
                "Trong một cuộc thảo luận trực tuyến, ",
                "Khi không rõ nguồn gốc câu nói, ",
                "Khi thấy nhiều người dùng cùng cụm từ, ",
                "Khi câu đùa được lặp lại nhiều lần, ",
                "Khi định sao chép bình luận, ",
                "Khi muốn tham gia xu hướng, ",
                "Khi thấy một câu có thể gây hiểu lầm, ",
                "Khi bạn chưa hiểu tiếng lóng, ",
                "Khi một bình luận nhắm vào người khác, ",
                "Khi muốn thể hiện ý kiến riêng, ",
                "Trước khi nhấn nút chia sẻ, ",
                "Khi thấy một câu đang nổi, ",
                "Trong một bài đăng có nhiều phản hồi giống nhau, ",
                "Khi cân nhắc dùng một câu viral, ",
                "Khi đọc nội dung được chia sẻ rộng rãi, ",
                "Khi bình luận của nhiều người giống nhau, ",
                "Khi muốn phản hồi một trào lưu, ",
                "Khi gặp một mẫu câu lặp lại, ",
                "Khi một câu nói được dùng ngoài ngữ cảnh, ",
                "Khi thấy người khác lặp lại câu nói, ",
                "Khi một câu viral có vẻ xúc phạm, ",
                "Khi định tham gia một trào lưu mới, ",
                "Khi muốn góp phần xây dựng mạng lành mạnh, "
            ]
            for i, cell_pos in enumerate(cells):
                base_q = bank[i % len(bank)]
                prefix = scenario_prefixes[i % len(scenario_prefixes)]

                question_text = prefix + base_q[0][:1].lower() + base_q[0][1:]
                if i >= len(scenario_prefixes):
                    question_text = f"Thử thách ô {i+1}: " + base_q[0]
                unique_stem = f"({i+1}) {question_text}"
                mapping[cell_pos] = (
                    unique_stem, list(base_q[1]), base_q[2],
                    base_q[3] + " Hãy hiểu nội dung và ngữ cảnh trước khi lặp lại."
                )
            self.maze_cell_questions[level_idx] = mapping
        self.root.bind("<Up>", lambda e: self.move_maze(-1, 0))
        self.root.bind("<Down>", lambda e: self.move_maze(1, 0))
        self.root.bind("<Left>", lambda e: self.move_maze(0, -1))
        self.root.bind("<Right>", lambda e: self.move_maze(0, 1))
        self.draw_scenario()

    def draw_scenario(self):
        self.page = "scenario"
        self.base()
        self.canvas.create_text(self.w/2, 67, text="TRẠM 02  •  MÊ CUNG COPYPASTA", fill=PINK, font=("Arial", 15, "bold"))
        hearts = "♥ " * self.maze_lives + "♡ " * (3-self.maze_lives)
        self.canvas.create_text(self.w/2, 96, text=f"CẤP {self.maze_level}/3     MẠNG: {hearts}     +20 XP khi thắng cấp", fill=RED, font=("Arial", 11, "bold"))
        grid = self.maze_maps[self.maze_level-1]
        rows, cols = len(grid), len(grid[0])
        cell = min(42, (self.w*0.44)/cols, (self.h-250)/rows)
        gx, gy = self.w*0.08, 150
        self.card(gx-18, gy-20, gx+cols*cell+18, gy+rows*cell+20, PURPLE)
        monster = self.maze_monsters[self.maze_level-1]
        for r, row in enumerate(grid):
            for c, ch in enumerate(row):
                xx, yy = gx+c*cell, gy+r*cell
                fill, outline = "#37315f", "#514a86"
                if ch == "#": fill, outline = "#10132f", "#7065b7"
                elif ch == "S": fill = CYAN
                elif ch == "E": fill = GREEN
                if [r,c] == self.maze_pos: fill = GOLD
                self.rounded(xx+2, yy+2, xx+cell-2, yy+cell-2, 5, fill, outline)
                label, color = "", WHITE
                if [r,c] == self.maze_pos: label, color = "🙂", "#171735"
                elif [r,c] == monster: label, color = "👾", PINK
                elif ch == "S": label, color = "B", "#171735"
                elif ch == "E": label, color = "CỬA", "#171735"
                elif ch == "#": label, color = "", WHITE
                if label: self.canvas.create_text(xx+cell/2, yy+cell/2, text=label, fill=color, font=("Arial", 10, "bold"))
        right_x = self.w*0.60
        self.card(right_x, 145, self.w-35, self.h-18, CYAN)
        if self.maze_lost:
            self.canvas.create_text((right_x+self.w-35)/2, 220, text="QUÁI VẬT ĐÃ BẮT ĐƯỢC BẠN!", fill=RED, font=("Arial", 13, "bold"), width=self.w*0.32)
            self.canvas.create_text((right_x+self.w-35)/2, 275, text="Bạn đã dùng hết 3 mạng. Hãy thử lại để luyện nhận diện Copypasta.", fill=WHITE, font=("Arial", 10), width=self.w*0.31)
            self.button(right_x+25, 345, self.w-60, 390, "CHƠI LẠI", self.start_scenario, GREEN)
        elif self.maze_won:
            self.canvas.create_text((right_x+self.w-35)/2, 220, text="BẠN ĐÃ ĐẾN CỬA!", fill=GREEN, font=("Arial", 16, "bold"))
            self.canvas.create_text((right_x+self.w-35)/2, 275, text=f"Hoàn thành cấp {self.maze_level}! Bạn nhận +20 XP.", fill=WHITE, font=("Arial", 11))
            if self.maze_level < 3:
                self.button(right_x+25, 345, self.w-60, 390, "SANG CẤP TIẾP  →", self.next_maze_level, GREEN)
            else:
                self.button(right_x+25, 345, self.w-60, 390, "NHẬN HUY HIỆU", self.finish_maze, GOLD)
        elif self.maze_quiz_active or getattr(self, "maze_show_answer_marks", False):
            q = self.maze_current_question
            self.canvas.create_text((right_x+self.w-35)/2, 178, text="TRẢ LỜI ĐÚNG ĐỂ ĐI TIẾP", fill=CYAN, font=("Arial", 10, "bold"))
            self.canvas.create_text((right_x+self.w-35)/2, 220, text=q[0], fill=WHITE, font=("Arial", 11, "bold"), width=self.w*0.31)
            for i, ans in enumerate(q[1]):
                x1, y1, x2, y2 = right_x+20, 245+i*42, self.w-55, 278+i*42
                label = f"{chr(65+i)}. {ans}"
                showing_marks = getattr(self, "maze_show_answer_marks", False)
                is_correct = showing_marks and i == q[2]
                is_wrong_pick = showing_marks and i == getattr(self, "maze_last_selected", None) and i != q[2]
                if showing_marks:
                    if is_correct:

                        self.rounded(x1-7, y1-7, x2+7, y2+7, 17, "#f8ff9a", "")
                        self.rounded(x1-4, y1-4, x2+4, y2+4, 16, "#baff78", "#f8ff9a", 3)
                        label = "✓  ĐÁP ÁN ĐÚNG: " + label
                    elif is_wrong_pick:
                        label = "✗  BẠN CHỌN SAI: " + label
                    callback = lambda: None
                else:
                    callback = lambda idx=i: self.answer_maze(idx)
                button_color = GREEN if is_correct else (RED if is_wrong_pick else [CYAN, PINK, GOLD][i])
                self.button(x1, y1, x2, y2, label, callback, button_color)
            if getattr(self, "maze_answer_feedback", ""):

                panel_left = right_x
                panel_right = self.w - 35
                panel_center = (panel_left + panel_right) / 2
                feedback_top = 435
                feedback_width = max(240, panel_right - panel_left - 54)

                feedback_font = 12 if self.h >= 760 else 10
                if len(self.maze_answer_feedback) > 430:
                    feedback_font = 11 if self.h >= 760 else 9
                self.canvas.create_text(
                    panel_center, feedback_top,
                    text=self.maze_answer_feedback,
                    fill=GREEN if getattr(self, "maze_last_correct", False) else GOLD,
                    font=("Arial", feedback_font, "bold"),
                    width=feedback_width, justify="center", anchor="n"
                )
            else:
                self.canvas.create_text((right_x+self.w-35)/2, 455,
                    text="Sai sẽ mất 1 mạng và chưa được qua ô đó.", fill=RED,
                    font=("Arial", 10 if self.h >= 760 else 9, "bold"),
                    width=self.w*0.32, justify="center", anchor="n")
        else:
            self.canvas.create_text((right_x+self.w-35)/2, 205, text="MỖI BƯỚC = MỘT CÂU HỎI", fill=CYAN, font=("Arial", 12, "bold"))
            self.canvas.create_text((right_x+self.w-35)/2, 260, text="Dùng phím mũi tên để chọn ô kế tiếp. Trả lời câu hỏi nhận diện Copypasta để di chuyển. Né quái vật 👾 và tìm cửa màu xanh.", fill=WHITE, font=("Arial", 11), width=self.w*0.31)
            self.canvas.create_text((right_x+self.w-35)/2, 350, text=self.maze_feedback, fill=MUTED, font=("Arial", 10), width=self.w*0.31)
        self.button(28, self.h-55, 145, self.h-25, "← TRANG CHỦ", self.show_home, PURPLE)

    def move_maze(self, dr, dc):
        if self.page != "scenario" or self.maze_quiz_active or self.maze_lost or self.maze_won:
            return
        grid = self.maze_maps[self.maze_level-1]
        r, c = self.maze_pos
        nr, nc = r+dr, c+dc
        if nr < 0 or nr >= len(grid) or nc < 0 or nc >= len(grid[nr]): return
        if grid[nr][nc] == "#":
            self.maze_feedback = "Đường bị chặn! Hãy chọn hướng khác."
            self.draw_scenario()
            return
        self.maze_pending = [nr, nc]
        self.maze_show_answer_marks = False
        self.maze_last_selected = None
        self.maze_answer_feedback = ""
        level_idx = self.maze_level - 1
        self.maze_current_question = self.maze_cell_questions[level_idx].get(
            (nr, nc),
            self.maze_question_banks[level_idx][(nr * 7 + nc) % len(self.maze_question_banks[level_idx])]
        )
        self.maze_quiz_active = True
        self.draw_scenario()

    def answer_maze(self, idx):
        if not self.maze_quiz_active or self.maze_lost or self.maze_won: return
        q = self.maze_current_question
        self.maze_last_selected = idx
        self.maze_show_answer_marks = True
        if idx == q[2]:
            self.maze_quiz_active = False
            self.maze_last_correct = True
            self.maze_answer_feedback = "CHÚC MỪNG! ✓ Đáp án chính xác!\n" + q[3] + "\nVì sao đúng: lựa chọn này phù hợp nhất với dấu hiệu và ngữ cảnh trong câu hỏi. Khi đánh giá Copypasta, cần xem xét cả việc lặp lại, mục đích sử dụng và tác động đến người đọc — không chỉ nhìn độ viral.\nDùng phím mũi tên để tiếp tục đến ô tiếp theo."
            self.maze_pos = self.maze_pending
            self.maze_feedback = "Đúng rồi! Bạn đã qua câu này. Hãy di chuyển đến ô tiếp theo để nhận câu hỏi mới."
            r, c = self.maze_pos
            if [r,c] == self.maze_monsters[self.maze_level-1]:
                self.maze_feedback = "Quái vật chặn đường! Hãy tiếp tục trả lời đúng để tìm lối thoát."
                self.maze_lives -= 1
                self.maze_monsters[self.maze_level-1] = [max(0, r-1), c]
                if self.maze_lives <= 0: self.maze_lost = True
            if self.maze_lives > 0 and self.maze_maps[self.maze_level-1][self.maze_pos[0]][self.maze_pos[1]] == "E":
                self.maze_won = True
                if self.maze_level not in getattr(self, "maze_done", set()):
                    if not hasattr(self, "maze_done"): self.maze_done = set()
                    self.maze_done.add(self.maze_level)
                    self.xp += 20
                    self.update_level()
                self.maze_feedback = f"ĐÃ ĐẾN CỬA CẤP {self.maze_level}! +20 XP"
        else:
            self.maze_last_correct = False
            self.maze_lives -= 1
            correct_answer = q[1][q[2]]

            self.maze_answer_feedback = (
                "CHƯA ĐÚNG — ✗ BẠN MẤT 1 MẠNG!\n"
                f"✗ Lựa chọn của bạn: {q[1][idx]}\n"
                f"✓ Đáp án đúng: {correct_answer}\n"
                f"Giải thích: {q[3]}\n"
                "Phân tích thêm: Hãy đối chiếu đáp án với tình huống cụ thể, không chỉ dựa vào việc một câu nói có nổi tiếng hay được nhiều người dùng. Với Copypasta, cần xem câu đó có bị sao chép/lặp lại theo mẫu hay không, có phù hợp ngữ cảnh không, và có thể ảnh hưởng đến người khác thế nào. Một dấu hiệu riêng lẻ chưa đủ để kết luận tuyệt đối; cách đánh giá tốt nhất là kết hợp nội dung, mức độ lặp lại và mục đích sử dụng.\n"
                f"Mạng còn lại: {self.maze_lives}/3. Bạn vẫn đứng ở ô cũ; dấu ✗ và ✓ giúp bạn nhận ra lựa chọn sai và đáp án đúng."
            )
            self.maze_quiz_active = False

            if self.maze_lives <= 0:
                self.maze_lost = True
                self.maze_feedback = "Bạn đã hết 3 mạng. Chơi lại để thử sức nhé!"
            else:
                self.maze_feedback = (
                    f"Sai rồi, mất 1 mạng. Bạn vẫn ở ô cũ; chưa thể qua ô vừa chọn. "
                    f"Đáp án đúng: {correct_answer}. Giải thích: {q[3]}"
                )
        self.draw_scenario()

    def next_maze_level(self):
        if not self.maze_won or self.maze_level >= 3: return
        self.maze_level += 1
        self.maze_pos = [0, 0]
        self.maze_lives = 3
        self.maze_pending = None
        self.maze_quiz_active = False
        self.maze_won = False
        self.maze_lost = False
        self.maze_feedback = "Cấp mới! Mỗi bước vẫn là một câu hỏi trắc nghiệm."
        self.maze_answer_feedback = ""
        self.maze_last_correct = False
        self.maze_show_answer_marks = False
        self.maze_last_selected = None
        self.draw_scenario()

    def finish_maze(self):
        self.badges.add("Nhà thám hiểm mê cung")
        self.badges.add("Thám tử ngôn ngữ")
        self.show_home()

    def start_trend_game(self):
        self.page = "trend"
        self.trend_phrase = ""
        self.trend_context = ""
        self.trend_result = None
        self.draw_trend_game()

    def draw_trend_game(self):
        self.page = "trend"
        self.base()
        self.canvas.create_text(
            self.w/2, 78,
            text="TRẠM 03  •  CÂU BẠN ĐANG DÙNG CÓ PHẢI COPYPASTA KHÔNG?",
            fill=GOLD, font=("Arial", 15, "bold"), width=self.w-100
        )
        self.canvas.create_text(
            self.w/2, 111,
            text="Nhập câu nói bạn thường sử dụng để xem các dấu hiệu cần kiểm tra.",
            fill=WHITE, font=("Arial", 11), width=self.w-120
        )
        self.card(75, 140, self.w-75, self.h-80, GOLD)
        self.canvas.create_text(105, 170, text="CÂU NÓI CỦA BẠN",
                                anchor="w", fill=CYAN, font=("Arial", 11, "bold"))
        self.trend_phrase_entry = tk.Entry(
            self.root, font=("Arial", 12), bg="#f6f5ff", fg="#171735",
            insertbackground="#171735", relief="flat"
        )
        self.trend_phrase_entry.insert(0, self.trend_phrase)
        self.canvas.create_window(self.w/2, 204, width=self.w-250, height=34,
                                  window=self.trend_phrase_entry)

        self.canvas.create_text(105, 244, text="NGỮ CẢNH BẠN DÙNG CÂU NÀY (không bắt buộc)",
                                anchor="w", fill=CYAN, font=("Arial", 10, "bold"))
        self.trend_context_entry = tk.Entry(
            self.root, font=("Arial", 11), bg="#f6f5ff", fg="#171735",
            insertbackground="#171735", relief="flat"
        )
        self.trend_context_entry.insert(0, self.trend_context)
        self.canvas.create_window(self.w/2, 276, width=self.w-250, height=32,
                                  window=self.trend_context_entry)

        self.button(self.w/2-120, 298, self.w/2+120, 334,
                    "KIỂM TRA CÂU NÓI", self.analyze_trend_phrase, GREEN)

        if self.trend_result:

            self.canvas.create_line(100, 350, self.w-100, 350, fill="#514a86", width=1)
            self.canvas.create_text(105, 370, text="KẾT QUẢ ĐÁNH GIÁ SƠ BỘ",
                                    anchor="w", fill=GOLD, font=("Arial", 12, "bold"))
            yy = 400
            for heading, detail in self.trend_result:
                self.canvas.create_text(105, yy, text=heading, anchor="w",
                                        fill=CYAN, font=("Arial", 10, "bold"))
                self.canvas.create_text(105, yy+22, text=detail, anchor="w",
                                        fill=WHITE, font=("Arial", 10),
                                        width=self.w-230, justify="left")
                yy += 54
            self.canvas.create_text(
                self.w/2, self.h-105,
                text="AI phân tích có thể mắc sai sót. Kết quả chỉ mang tính tham khảo; hãy đối chiếu với ngữ cảnh và thông tin thực tế.",
                fill=MUTED, font=("Arial", 9), width=self.w-170, justify="center"
            )
        else:
            self.canvas.create_text(
                self.w/2, 390,
                text="Công cụ sẽ xem xét dấu hiệu lặp lại, ý nghĩa/ngữ cảnh, dữ liệu phổ biến và khả năng bị bắt chước.",
                fill=MUTED, font=("Arial", 11), width=self.w-230, justify="center"
            )
        self.button(32, self.h-55, 160, self.h-24, "← TRANG CHỦ",
                    self.show_home, PURPLE)

    def analyze_trend_phrase(self):
        phrase = self.trend_phrase_entry.get().strip()
        context = self.trend_context_entry.get().strip()
        self.trend_phrase = phrase
        self.trend_context = context

        if not phrase:
            self.trend_result = [
                ("VUI LÒNG NHẬP CÂU NÓI", "Hãy nhập một câu hoặc cụm từ trước khi kiểm tra.")
            ]
            self.draw_trend_game()
            return

        def norm(text):
            text = text.lower().replace("đ", "d")
            text = unicodedata.normalize("NFD", text)
            text = "".join(ch for ch in text if unicodedata.category(ch) != "Mn")
            text = re.sub(r"[^a-z0-9\s]", " ", text)
            return " ".join(text.split())

        p = norm(phrase)

        high_patterns = [
            (["ba khi"], "“Bá khí”"),
            (["ai hoi"], "“Ai hỏi?”"),
            (["go xa cu a", "day la go xa cu a"], "“Gỗ xà cừ ạ”"),
            (["hoi chieu troi mua"], "“Hồi chiều trời mưa”"),
            (["ua alo"], "“Ủa alo?”"),
            (["u la troi tui kieu", "ultr tui kiu", "ultr tui ciu", "u la troi tui kiu"], "“U là trời tui kiểu / Ultr tui kiểu”"),
            (["farm aura", "aura farming"], "“Farm aura / aura farming”"),
            (["67"], "“67”"),
            (["tran thanh khong muon lai di muon truong giang"], "“Trấn Thành không muốn, lại đi muốn Trường Giang à?”"),
            (["ba dao tren tung hat gao"], "“Bá đạo trên từng hạt gạo”"),
            (["kiep nan thu 82"], "“Kiếp nạn thứ 82”"),
            (["ratio"], "“Ratio”"),
            (["mai keo"], "“Mãi keo”"),
            (["xit keo"], "“Xịt keo”"),
            (["dung nhan sai cai"], "“Đúng nhận sai cãi”"),
            (["tuyet doi dien anh"], "“Tuyệt đối điện ảnh”"),
        ]

        medium_patterns = [
            (["alo", "a lo"], "các biến thể quanh “alo”"),
            (["sieu ba khi", "sieu cap ba khi vo dich"], "biến thể nhấn mạnh của “bá khí”"),
            (["aii hoi", "ai hoi"], "cách viết biến thể của “Ai hỏi?”"),
            (["tui kieu", "tui kiu", "tui ciu"], "các cách viết gần giống “tui kiểu”"),
            (["aura"], "cụm từ có yếu tố “aura”"),
            (["ba do", "ba khi"], "các cách viết gần giống “bá khí / bá dơ”"),
            (["hoi chieu"], "biến thể rút gọn của “hồi chiều trời mưa”"),
            (["ba khi", "ba do"], "các biến thể gần giống “bá khí / bá dơ”"),
            (["go xa cu", "go xa cu a"], "biến thể gần giống “gỗ xà cừ ạ”"),
            (["cong vien lam lang"], "cụm từ có thể liên quan đến trend “công viên lãm làng”"),
            (["ai co the", "lili van"], "cụm từ có thể liên quan đến một câu viral đang lan truyền"),
            (["sonion", "son"], "từ ngắn có thể là tên gọi hoặc cách nói theo trend; cần thêm ngữ cảnh"),
            (["tung tung", "sahur", "sha hua"], "cụm từ có thể liên quan đến meme/âm thanh viral"),
            (["u la troi", "u la choi"], "biến thể cảm thán gần giống “U là trời”"),
            (["slay"], "từ lóng thường gặp trên mạng; cần ngữ cảnh để xác định có phải mẫu Copypasta"),
            (["peak", "out meta", "sigma", "simp"], "từ/cụm từ mạng có thể xuất hiện trong trào lưu"),
            (["khong nho", "ko nho", "k nho", "kho nho", "khong biet", "khong ro",
              "khong nho ro", "khong nho nua", "khong nho noi", "quen mat roi",
              "chua nho", "khong nhan ra", "khong co cau nao", "khong co",
              "khong nghi ra", "nhieu qua khong nho"],
             "cụm từ thể hiện người viết không nhớ hoặc chưa xác định được câu nói"),
        ]

        matched_high = None
        for aliases, label in high_patterns:
            if any(alias in p for alias in aliases):
                matched_high = (aliases, label)
                break

        supplied_copypasta_terms = [
            ("ai hoi", "“Ai hỏi?”"), ("hoi chieu troi mua", "“Hồi chiều trời mưa”"),
            ("cong vien lam lang", "“Công viên lãm làng”"), ("ua alo", "“Ủa alo?”"),
            ("ai o si ma", "“ài ố sì mà”"), ("du an ma", "“Dự án ma”"),
            ("nhu muoi bo bien", "“Như muối bỏ biển”"),
            ("farm aura", "“Farm aura”"), ("aura farming", "“Aura farming”"),
            ("ac ma con khoe nua", "“Ác mà còn khoe nữa”"), ("khong biet", "“Không biết”"),
            ("67", "“67”"), ("u la troi tui kieu", "“U là trời tui kiểu”"),
            ("sieu ba khi", "“Siêu bá khí”"), ("alo", "“Alo”"),
            ("cam kon ni un yai quen yi un li nia nia ha chi wao tao tao bay zau u chi de nung e san e li ko cam khoai can o i le", "câu biến âm/nhại âm dài"),
            ("minh tung lap lai tu ba khi", "“Mình từng lặp lại từ Bá khí”"),
            ("uoc gi tao bot dang cap 1 chut", "“Ước gì tao bớt đẳng cấp một chút”"),
            ("hong a thay dui moi noi", "“Hong á, thấy dui mới nói”"),
            ("ban chat cua dao dong dieu hoa", "câu đùa về “bản chất của dao động điều hòa”"),
            ("em chao sep", "“Em chào sếp”"),
            ("ultr tui kiu", "“Ultr tui kiểu”"), ("cang xiu", "“Căng xỉu”"),
            ("hy", "“hy”"), ("ba dao tren tung hat gao", "“Bá đạo trên từng hạt gạo”"),
            ("kiep nan thu 82", "“Kiếp nạn thứ 82”"), ("trinh la gi", "“Trình là gì?”"),
            ("sieu cap ba khi vo dich", "“Siêu cấp bá khí vô địch”"),
            ("gan nhu chua tung lap lai cau nao", "câu nói về việc gần như chưa lặp lại câu viral"),
            ("ba", "“Bá”"), ("cheu luon", "“Chệu luôn”"),
            ("qua dep trai", "“Quá đẹp trai”"), ("alo vu a vu", "“Alo Vũ à Vũ”"),
            ("mama ten", "“Mama ten”"), ("hom nay troi mua", "“Hôm nay trời mưa”"),
            ("am long", "“Ấm lòng”"), ("kho nha bro", "“Khó nha bro”"),
            ("can tho co mot cay cau nguoi ta goi no la cau can tho", "“Cần Thơ có một cây cầu…”"),
            ("cung cung", "“Cũng cũng”"), ("it quan tam nhung van de viral", "câu nói về việc ít quan tâm vấn đề viral"),
            ("quan ta cai bon", "“Quăn ta cái bon”"), ("chua tung lap lai nhieu lan cau noi nao tren mang", "câu hỏi về việc lặp lại câu nói trên mạng"),
            ("slay mama ten", "“Slay mama ten”"),
            ("slay vu", "“Slay vữ”"), ("me oi cuu con", "“Mẹ ơi cứu con”"),
            ("36 367", "“36,367”"), ("36367", "“36,367”"), ("thua gia cat luong dung cai quat", "“Thua Gia Cát Lượng đúng cái quạt”"),
            ("out meta", "“Out meta”"), ("10 diem", "“10 điểm”"), ("ba do", "“Bá dơ”"),
            ("ban tum lum", "“Bắn tùm lum”"), ("tram ai chinh", "“Trầm ai chính”"),
            ("chan nhu con gian", "“Chán như con gián”"), ("tung tung sha hua", "“Tung tung sha hua”"),
            ("lop du phong", "“Lốp dự phòng”"), ("the ma lai hay", "“Thế mà lại hay”"),
            ("ra de", "“Ra dẻ”"), ("lop", "“Lốp”"), ("truong giang qua", "“Trường Giang quá”"),
            ("bap luoc day", "“Bắp luộc đây”"), ("o may zing gut chop tam giac la tac giam", "“Ờ mây zing gút chóp, tam giác là tác giam”"),
            ("tuat", "“Tuất”"), ("sigma", "“Sigma”"),
            ("otp khong real tao di bang dau", "“OTP không real tao đi bằng đầu”"),
            ("co ut xi tin", "“Cô út xì tin”"), ("ong cha hui ba cha hom", "“Ông chã húi bà chã hôm”"),
            ("ba noi thiet ha ba tho", "“Bà nói thiệt hả bà thơ”"),
            ("minh thay cau lap di ma pho bien tren tiktok", "câu nhận xét về những câu viral trên TikTok"),
            ("ac nhu con ca thac lac", "“Ác như con cá thác lác”"), ("jet jet", "“Jet jet”"),
            ("chieu 1 dieu thuyen chieu 2 alic", "“Chiêu 1 Điêu Thuyền, chiêu 2 Alice”"),
            ("aura", "“Aura”"), ("ratio", "“Ratio”"), ("tranh thanh khong muon lai di muon truong giang", "câu đùa nhắc Trấn Thành và Trường Giang"),
            ("ai co the lili van ko", "“Ai có thẻ Lili Van không?”"),
            ("be con nho dung cho an cay qua", "câu đùa “Bé còn nhỏ đừng cho ăn cay quá…”"),
            ("kiếp sau vẫn muốn cùng em lập gia đình", "câu đùa về hai gia đình làm thông gia"),
        ]
        if not matched_high:
            for alias, label in supplied_copypasta_terms:
                normalized_alias = norm(alias)
                if normalized_alias and normalized_alias in p:
                    matched_high = ([normalized_alias], label)
                    break

        matched_medium = None
        if not matched_high:
            for aliases, label in medium_patterns:
                if any(alias in p for alias in aliases):
                    matched_medium = (aliases, label)
                    break

        if matched_high:
            label = matched_high[1]
            classification = "KHẢ NĂNG CAO LÀ MẪU COPYPASTA"
            sign_text = (
                f"Câu có dấu hiệu của một mẫu Copypasta vì ngắn, dễ nhận ra và có thể được nhiều người nhắc lại theo cùng một khuôn. "
                f"Điểm đáng chú ý ở đây là sự lặp lại theo mẫu {label}, chứ không chỉ đơn thuần là nhiều người cùng dùng một từ."
            )

            if "ba khi" in p:
                meaning_text = (
                    "“Bá khí” thường chỉ phong thái mạnh mẽ, áp đảo hoặc rất tự tin; khi dùng như câu trend, nó có thể là lời khen, lời trêu hoặc cách phóng đại một hành động. "
                    "Sắc thái phụ thuộc vào tình huống: khen thật, đùa vui hay mỉa mai."
                )
            elif "go xa cu" in p:
                meaning_text = (
                    "Cách đáp “Gỗ xà cừ ạ” gây chú ý nhờ vẻ trả lời nghiêm túc nhưng bất ngờ, tạo cảm giác lệch khỏi mạch hội thoại. "
                    "Hiệu ứng hài nằm ở sự bất ngờ; nếu lặp lại không đúng ngữ cảnh, câu nói dễ trở thành câu đáp máy móc và làm cuộc trò chuyện mất tự nhiên."
                )
            elif "ai hoi" in p:
                meaning_text = (
                    "“Ai hỏi?” thường được dùng để phản ứng khi cho rằng người khác nói điều không được yêu cầu. Câu rất ngắn nhưng có thể mang sắc thái đùa, lạnh nhạt hoặc phủ nhận lời người khác. "
                    "Dùng đúng lúc có thể tạo hài hước; dùng để hạ thấp người khác dễ khiến họ cảm thấy bị gạt đi."
                )
            elif "farm aura" in p or "aura farming" in p:
                meaning_text = (
                    "“Aura farming” thường nói về việc cố tạo hình ảnh ngầu, cuốn hút hoặc có khí chất trước người khác. Cụm từ có thể dùng để khen một khoảnh khắc ấn tượng, hoặc châm biếm hành động cố tỏ ra nổi bật. "
                    "Cần xét hình ảnh và tình huống cụ thể để hiểu đó là lời khen hay lời trêu."
                )
            elif "67" == p or p.endswith(" 67"):
                meaning_text = (
                    "Một con số đơn lẻ có thể trở thành tín hiệu nhận diện của một trào lưu khi cộng đồng cùng hiểu liên tưởng phía sau. Nếu thiếu bối cảnh, người ngoài cuộc khó hiểu vì sao con số ấy buồn cười hoặc đáng nhắc lại. "
                    "Đây là ví dụ cho việc ý nghĩa trên mạng đôi khi đến từ quy ước chung, không chỉ từ nghĩa đen."
                )
            elif "ratio" in p:
                meaning_text = (
                    "“Ratio” trong văn hóa mạng thường gắn với tình huống một bài đăng hoặc ý kiến nhận phản hồi bất lợi, chẳng hạn phản hồi áp đảo nội dung ban đầu. "
                    "Dùng từ này có thể tóm tắt phản ứng của cộng đồng, nhưng nếu chỉ lặp lại để công kích thì dễ biến trao đổi thành chế giễu thay vì tranh luận về ý tưởng."
                )
            elif "mai keo" in p:
                meaning_text = (
                    "“Mãi keo” là cách nói thân mật để thể hiện sự gắn bó, quý mến hoặc ủng hộ ai đó/điều gì đó. Sự biến đổi cách viết tạo cảm giác gần gũi, vui vẻ và dễ dùng trong bình luận. "
                    "Tuy vậy, ý nghĩa cụ thể vẫn phụ thuộc vào đối tượng được nhắc đến và giọng điệu của người viết."
                )
            elif "xit keo" in p:
                meaning_text = (
                    "“Xịt keo” thường diễn tả trạng thái bất ngờ, đứng hình hoặc không biết phản ứng ra sao. Hình ảnh ẩn dụ ngắn gọn khiến câu dễ dùng để mô tả nhiều tình huống. "
                    "Khi lặp lại, người dùng nên bảo đảm câu phù hợp cảm xúc của tình huống, thay vì chèn vào mọi bình luận chỉ vì thấy quen thuộc."
                )
            elif "dung nhan sai cai" in p:
                meaning_text = (
                    "“Đúng nhận sai cãi” tạo nhịp điệu đối xứng và giọng khẳng định mạnh, nên dễ được nhắc lại như một câu cửa miệng. "
                    "Câu hàm ý yêu cầu thừa nhận điều đúng và không phủ nhận sự thật; nhưng trong tranh luận thực tế, cần có căn cứ rõ ràng chứ không thể dùng câu này để mặc định một bên luôn đúng."
                )
            elif "kiep nan" in p:
                meaning_text = (
                    "“Kiếp nạn thứ…” là cách phóng đại một sự việc phiền phức như thể đó là thử thách tiếp theo trong chuỗi dài. Cách nói tạo sự hài hước nhờ biến chuyện thường ngày thành một “nhiệm vụ” khó khăn. "
                    "Nó phù hợp với tình huống than thở vui, nhưng có thể mất ý nghĩa nếu dùng lặp lại mà không liên quan."
                )
            elif "ba dao tren tung hat gao" in p:
                meaning_text = (
                    "“Bá đạo trên từng hạt gạo” dựa vào vần điệu và lối nói cường điệu, khiến câu nghe vui tai và dễ nhớ. Nó thường được dùng để khen hoặc trêu một hành động quá nổi bật, khác thường. "
                    "Sức hút nằm ở nhịp câu và sự phóng đại hơn là thông tin theo nghĩa đen."
                )
            elif "tuyet doi dien anh" in p:
                meaning_text = (
                    "“Tuyệt đối điện ảnh” thường dùng để gọi một khoảnh khắc có cảm giác kịch tính, đẹp hoặc ấn tượng như trong phim. Câu có sắc thái tán thưởng, đôi khi được dùng cường điệu để tạo hài hước. "
                    "Nên xem xét khoảnh khắc được nói đến để phân biệt lời khen thực sự với cách nói đùa."
                )
            elif "hoi chieu troi mua" in p:
                meaning_text = (
                    "“Hồi chiều trời mưa” tạo hiệu ứng khi được lặp lại như một câu đáp quen thuộc hoặc xuất hiện không ăn khớp với câu chuyện. Sự hài hước thường đến từ tính bất ngờ và việc người nghe nhận ra khuôn câu. "
                    "Nếu không có ngữ cảnh chung, người đọc có thể chỉ thấy câu nói rời rạc, khó hiểu."
                )
            elif "ua alo" in p:
                meaning_text = (
                    "“Ủa alo?” thể hiện sự ngạc nhiên, khó tin hoặc muốn người khác giải thích điều vừa xảy ra. Từ ngữ đời thường và ngắn khiến câu dễ được dùng như phản ứng nhanh. "
                    "Tùy giọng điệu, nó có thể thân thiện, hài hước hoặc mang ý trách móc."
                )
            elif "u la troi" in p:
                meaning_text = (
                    "“U là trời, tui kiểu…” là cách biểu đạt cảm xúc mạnh theo giọng nói thân mật, thường mở đầu cho phản ứng bất ngờ, ngượng ngùng hoặc phấn khích. "
                    "Cách viết biến thể làm câu có màu sắc cộng đồng; người dùng nên chú ý hoàn cảnh và tránh sao chép nguyên câu khi cảm xúc thật của mình khác với nội dung đó."
                )
            elif "tran thanh" in p:
                meaning_text = (
                    "Câu hỏi nhắc đến người nổi tiếng tạo tiếng cười bằng cách ghép tên riêng vào một cấu trúc bất ngờ, như thể người nghe phải nhận ra một liên tưởng chung. "
                    "Vì phụ thuộc vào bối cảnh văn hóa đại chúng, câu có thể khó hiểu với người không biết nguồn liên tưởng; không nên xem nội dung đùa là thông tin có thật."
                )
            else:
                meaning_text = (
                    f"{label} có thể được dùng như lời đùa, phản ứng hoặc dấu hiệu nhận diện của một trào lưu tùy ngữ cảnh. "
                    "Muốn hiểu đúng cần xem ai nói, nói với ai, trong tình huống nào và người đọc có cùng hiểu biết nền hay không."
                )
            popularity_text = (
                "DẤU HIỆU LAN TRUYỀN: câu có cấu trúc dễ nhớ, dễ trích dẫn và có thể được dùng lại trong nhiều tình huống. "
                "Đây là đánh giá định tính từ đặc điểm câu chữ; chưa phải thống kê lượt xem, lượt bình luận hay tần suất thực tế trên từng nền tảng."
            )
            potential_text = (
                "CAO: câu có yếu tố thuận lợi để thành Copypasta như ngắn gọn, có nhịp/điểm gây cười hoặc tạo phản ứng nhanh, đồng thời dễ biến đổi theo ngữ cảnh. "
                "Nếu nhiều người tiếp tục dùng lại cùng khuôn câu, nó có thể lan rộng; mức độ lan truyền thực tế vẫn cần dữ liệu quan sát để xác nhận."
            )
        elif matched_medium:
            label = matched_medium[1]
            classification = "MỨC TRUNG BÌNH — CÓ DẤU HIỆU LIÊN QUAN"
            sign_text = (
                f"Câu có nét tương đồng với {label}, nhưng dấu hiệu nhận diện chưa đủ đặc trưng để kết luận đây là cùng một mẫu Copypasta. "
                "Có thể là biến thể có chủ ý, cách viết sáng tạo hoặc chỉ trùng một vài từ; cần đối chiếu cách dùng thực tế."
            )
            meaning_text = (
                f"Cụm từ gần với {label}, nhưng khác biệt nhỏ về từ ngữ có thể làm đổi sắc thái, ý đùa hoặc ý định của người nói. "
                + (f"Ngữ cảnh bạn cung cấp: {context}. Hãy xem câu có liên quan trực tiếp đến nội dung đang bàn hay chỉ được chèn vào để bắt trend." if context else "Chưa có ngữ cảnh cụ thể, nên chưa thể xác định câu đang được dùng để khen, đùa, mỉa mai hay phản ứng.")
            )
            popularity_text = (
                "TRUNG BÌNH: có đặc điểm gợi liên tưởng đến một kiểu câu dễ được bắt chước, nhưng chưa đủ cơ sở để nói chính cách viết này phổ biến rộng. "
                "Độ phổ biến cần được kiểm chứng bằng số lần xuất hiện, thời điểm và phạm vi người dùng."
            )
            potential_text = (
                "TRUNG BÌNH: câu có thể phát triển thành Copypasta nếu được nhiều người dùng lại với ý nghĩa hoặc tình huống tương tự. "
                "Nếu chỉ xuất hiện đơn lẻ hoặc không tạo được phản ứng chung, khả năng lan truyền sẽ thấp hơn."
            )
        else:
            classification = "CHƯA ĐỦ DẤU HIỆU ĐỂ XÁC ĐỊNH COPYPASTA"
            sign_text = (
                "Chưa nhận thấy sự trùng khớp rõ với những mẫu câu thường được nhận diện trong công cụ này. "
                "Điều đó không chứng minh câu chưa từng viral; câu có thể mới xuất hiện, dùng cách viết khác hoặc chỉ phổ biến trong một cộng đồng nhỏ."
            )
            meaning_text = (
                (f"Ngữ cảnh bạn cung cấp: {context}. " if context else "Chưa có ngữ cảnh cụ thể. ")
                + "Để phân tích chính xác, cần xác định câu đang truyền tải ý gì, nhắm đến ai, mang sắc thái khen/đùa/mỉa mai hay xúc phạm, và có phù hợp với cuộc trò chuyện không."
            )
            popularity_text = (
                "CHƯA ĐỦ DỮ KIỆN: chỉ từ một câu riêng lẻ không thể kết luận mức độ phổ biến. "
                "Cần xem tần suất xuất hiện, số người dùng độc lập, thời gian lan truyền và các biến thể của câu trên nhiều ngữ cảnh."
            )
            potential_text = (
                "CHƯA THỂ KẾT LUẬN: câu có thể trở thành Copypasta nếu dễ nhớ, dễ bắt chước, tạo cảm xúc hoặc có cách dùng linh hoạt. "
                "Ngược lại, câu quá phụ thuộc vào một tình huống riêng có thể khó được cộng đồng sử dụng rộng rãi."
            )

        self.trend_result = [
            ("KẾT LUẬN: " + classification, sign_text),
            ("1. Ý NGHĨA VÀ NGỮ CẢNH", meaning_text),
            ("2. MỨC ĐỘ PHỔ BIẾN", popularity_text),
            ("3. KHẢ NĂNG PHÁT TRIỂN THÀNH COPYPASTA", potential_text),
        ]
        self.xp += 5
        self.update_level()
        self.draw_trend_game()

    def update_level(self):
        self.level = 1 + self.xp // 100
        self.save_progress()

    def show_inspire(self):
        self.page = "inspire"
        self.draw_inspire()

    def next_inspiration(self):
        self.inspiration_index = (self.inspiration_index + 1) % len(INSPIRATIONS)
        self.draw_inspire()

    def complete_challenge(self, day):
        if day in self.challenge_done:
            self.feedback = "Bạn đã hoàn thành thử thách này rồi. Hãy duy trì điều tốt đẹp nhé!"
        else:
            self.challenge_done.add(day)
            self.xp += 15
            self.update_level()
            self.feedback = f"Hoàn thành Ngày {day}: +15 XP! Mỗi thay đổi nhỏ đều có ý nghĩa."
        self.draw_inspire()
        self.canvas.create_text(self.w/2, self.h-28, text=self.feedback,
                                fill=GREEN, font=("Arial", 10, "bold"))

    def make_pledge(self):
        if not self.pledge_made:
            self.pledge_made = True
            self.xp += 20
            self.update_level()
            self.badges.add("Đại sứ tư duy số")
        self.draw_inspire()
        self.canvas.create_text(self.w/2, self.h-28,
            text="CAM KẾT ĐÃ ĐƯỢC GHI NHẬN  •  +20 XP  •  Hãy biến lời hứa thành hành động nhỏ mỗi ngày!",
            fill=GREEN, font=("Arial", 10, "bold"))

    def draw_inspire(self):
        self.page = "inspire"
        self.base()
        self.canvas.create_text(self.w/2, 88, text="GÓC TRUYỀN CẢM HỨNG  •  HỌC QUA TRẢI NGHIỆM",
                                fill=PURPLE, font=("Arial", 10, "bold"))
        self.canvas.create_text(self.w/2, 116, text="Bạn không cần nói giống số đông để thuộc về số đông.",
                                fill=WHITE, font=("Arial", 17, "bold"))
        self.canvas.create_text(self.w/2, 141,
            text="Hãy hiểu trước khi chia sẻ. Hãy suy nghĩ trước khi bình luận. Hãy để tiếng nói của bạn có dấu ấn riêng.",
            fill=MUTED, font=("Arial", 10))

        x1, y1, x2, y2 = 55, 163, self.w-55, 247
        self.card(x1, y1, x2, y2, CYAN)
        quote, lesson, author = INSPIRATIONS[self.inspiration_index]
        self.canvas.create_text(x1+22, y1+20, text="✦ THÔNG ĐIỆP HÔM NAY",
                                anchor="w", fill=CYAN, font=("Arial", 10, "bold"))
        self.canvas.create_text(self.w/2, y1+43, text=f"“{quote}”",
                                width=self.w-150, fill=WHITE,
                                font=("Arial", 12, "bold"), justify="center")
        self.canvas.create_text(self.w/2, y1+66, text=lesson,
                                width=self.w-160, fill=MUTED,
                                font=("Arial", 10), justify="center")
        self.button(x2-178, y1+12, x2-18, y1+36, "THÔNG ĐIỆP KHÁC ↻",
                    self.next_inspiration, CYAN)

        gap = 14
        left = 55
        total_w = self.w - 110
        cw = (total_w - 2*gap) / 3
        top, bottom = 263, 438
        panels = [
            (left, "CÔNG THỨC 5 GIÂY", CYAN, [
                "1. DỪNG: Đừng chia sẻ chỉ vì thấy nhiều người dùng.",
                "2. HIỂU: Câu này có nghĩa gì? Có thể gây tổn thương không?",
                "3. KIỂM TRA: Nội dung có đúng ngữ cảnh không?",
                "4. CHỌN: Mình thật sự muốn nói điều này chứ?",
                "5. VIẾT LẠI: Dùng lời của mình khi có thể."
            ]),
            (left+cw+gap, "7 NGÀY – 7 THÓI QUEN", PINK, [
                "Ngày 1: Quan sát một câu được lặp lại nhiều lần.",
                "Ngày 2: Tìm hiểu nghĩa trước khi dùng một câu viral.",
                "Ngày 3: Viết một bình luận bằng lời của mình.",
                "Ngày 4: Không hùa theo lời trêu chọc người khác.",
                "Ngày 5: Kiểm tra ngữ cảnh trước khi chia sẻ.",
                "Ngày 6: Khen một ý kiến sáng tạo, chân thành.",
                "Ngày 7: Tự đánh giá điều mình đã học được."
            ]),
            (left+2*(cw+gap), "GHI NHỚ ĐIỀU NÀY", GOLD, [
                "• Copypasta không phải lúc nào cũng xấu.",
                "• Lặp lại có thể vui, nhưng hãy hiểu điều mình lặp.",
                "• Tiếng lóng là một phần của văn hóa mạng.",
                "• Điều quan trọng là ngữ cảnh, mục đích và tác động.",
                "• Đừng dùng trào lưu để hạ thấp hay cô lập người khác.",
                "• Sáng tạo không đòi hỏi phải hoàn hảo ngay từ đầu."
            ])
        ]
        for px, title, color, lines in panels:
            self.card(px, top, px+cw, bottom, color)
            self.canvas.create_text(px+14, top+22, text=title, anchor="w",
                                    fill=color, font=("Arial", 10, "bold"))
            yy = top+47
            for line in lines:
                self.canvas.create_text(px+14, yy, text=line, anchor="w",
                                        width=cw-26, fill=WHITE,
                                        font=("Arial", 10), justify="left")
                yy += 18 if len(line) < 58 else 22

        py1, py2 = 454, self.h-67
        self.card(55, py1, self.w-55, py2, PURPLE)
        self.canvas.create_text(75, py1+22, text="TUYÊN NGÔN NGƯỜI DÙNG MẠNG CÓ TRÁCH NHIỆM",
                                anchor="w", fill=PURPLE, font=("Arial", 10, "bold"))
        pledge_text = ("Tôi sẽ cố gắng hiểu trước khi lặp lại; suy nghĩ trước khi bình luận; "
                       "tôn trọng người khác; kiểm tra ngữ cảnh; và giữ quyền được diễn đạt bằng tiếng nói riêng.")
        self.canvas.create_text(75, py1+48, text=pledge_text, anchor="w",
                                width=self.w-150, fill=WHITE, font=("Arial", 10), justify="left")
        self.button(75, py1+76, 265, py1+109,
                    "TÔI CAM KẾT  +20 XP" if not self.pledge_made else "ĐÃ CAM KẾT  ✓",
                    self.make_pledge, GREEN if not self.pledge_made else "#9b9bb5")

        self.canvas.create_text(300, py1+85, text="THỬ THÁCH 7 NGÀY:", anchor="w",
                                fill=CYAN, font=("Arial", 10, "bold"))
        bw = min(72, (self.w-390)/7)
        for day in range(1, 8):
            bx = 430 + (day-1)*(bw+5)
            if bx+bw > self.w-72:
                break
            done = day in self.challenge_done
            self.button(bx, py1+73, bx+bw, py1+108,
                        f"NGÀY {day} ✓" if done else f"NGÀY {day}",
                        lambda d=day: self.complete_challenge(d),
                        GREEN if done else (CYAN if day % 2 else PINK))
        self.button(32, self.h-49, 150, self.h-20, "← TRANG CHỦ",
                    self.show_home, PURPLE)

class AccountSystem:
    def __init__(self, root):
        self.root = root
        app_data = os.environ.get("LOCALAPPDATA") or os.path.expanduser("~")
        data_dir = os.path.join(app_data, "CopyQuest")
        os.makedirs(data_dir, exist_ok=True)
        self.db_path = os.path.join(data_dir, "copyquest_accounts.sqlite3")
        old_db = os.path.join(os.path.dirname(os.path.abspath(__file__)), "copyquest_accounts.sqlite3")
        if not os.path.exists(self.db_path) and os.path.isfile(old_db):
            try:
                shutil.copy2(old_db, self.db_path)
            except OSError:
                pass
        self.create_database()
        self.login_frame = None
        self.app = None
        self.show_login()

    def connect(self):
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row
        return conn

    def hash_password(self, password, salt=None):
        salt = salt or secrets.token_bytes(16)
        digest = hashlib.pbkdf2_hmac("sha256", password.encode("utf-8"), salt, 240000)
        return salt.hex(), digest.hex()

    def create_database(self):
        with self.connect() as conn:
            conn.execute("CREATE TABLE IF NOT EXISTS users (username TEXT PRIMARY KEY COLLATE NOCASE, salt TEXT NOT NULL, password_hash TEXT NOT NULL, role TEXT NOT NULL DEFAULT 'user', enabled INTEGER NOT NULL DEFAULT 1, xp INTEGER NOT NULL DEFAULT 0, level INTEGER NOT NULL DEFAULT 1, created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP)")
            row = conn.execute("SELECT username FROM users WHERE username = ?", ("admin",)).fetchone()
            if row is None:
                salt, digest = self.hash_password("CopyQuest@2026")
                conn.execute("INSERT INTO users(username, salt, password_hash, role, enabled) VALUES (?, ?, ?, 'admin', 1)", ("admin", salt, digest))

    def verify_password(self, username, password):
        with self.connect() as conn:
            row = conn.execute("SELECT * FROM users WHERE username = ? COLLATE NOCASE", (username.strip(),)).fetchone()
        if not row:
            return None, "Tên đăng nhập hoặc mật khẩu không đúng."
        if not row["enabled"]:
            return None, "Tài khoản này đang bị khóa. Hãy liên hệ admin."
        salt, digest = self.hash_password(password, bytes.fromhex(row["salt"]))
        if not hmac.compare_digest(digest, row["password_hash"]):
            return None, "Tên đăng nhập hoặc mật khẩu không đúng."
        return dict(row), ""

    def register_user(self, username, password):
        username = username.strip()
        if not username:
            return False, "Vui lòng nhập tên đăng nhập."
        if not password:
            return False, "Vui lòng nhập mật khẩu."
        salt, digest = self.hash_password(password)
        try:
            with self.connect() as conn:
                conn.execute("INSERT INTO users(username, salt, password_hash, role, enabled) VALUES (?, ?, ?, 'user', 1)", (username, salt, digest))
            return True, "Tạo tài khoản thành công. Bạn có thể đăng nhập ngay."
        except sqlite3.IntegrityError:
            return False, "Username này đã tồn tại. Hãy chọn tên khác."

    def get_progress(self, username):
        with self.connect() as conn:
            row = conn.execute("SELECT xp, level FROM users WHERE username = ? COLLATE NOCASE", (username,)).fetchone()
        return (row["xp"], row["level"]) if row else (0, 1)

    def save_progress(self, username, xp, level):
        with self.connect() as conn:
            conn.execute("UPDATE users SET xp = ?, level = ? WHERE username = ? COLLATE NOCASE", (int(xp), int(level), username))

    def clear_frame(self):
        if self.login_frame is not None:
            self.login_frame.destroy()
            self.login_frame = None

    def show_login(self, register=False):
        """Màn hình đăng nhập/đăng ký mới; giữ nguyên hệ thống tài khoản và trò chơi."""
        self.clear_frame()
        self.root.title("Trạm lọc thông tin | Tiếp nhận, kiểm chứng và chia sẻ có chọn lọc")
        self.root.geometry("1100x720")
        self.root.minsize(900, 620)
        frame = tk.Frame(self.root, bg="#11132f")
        frame.pack(fill="both", expand=True)
        self.login_frame = frame

        bg = tk.Canvas(frame, bg="#11132f", highlightthickness=0)
        bg.place(relx=0, rely=0, relwidth=1, relheight=1)
        top = (17, 19, 48); bottom = (31, 21, 57)
        def paint(event=None):
            bg.delete("background")
            w, h = max(frame.winfo_width(), 900), max(frame.winfo_height(), 620)
            for i in range(75):
                t = i / 74
                rgb = tuple(int(top[k] * (1-t) + bottom[k] * t) for k in range(3))
                color = "#%02x%02x%02x" % rgb
                y0, y1 = int(h*i/75), int(h*(i+1)/75)+1
                bg.create_rectangle(0, y0, w, y1, fill=color, outline=color, tags="background")
            bg.create_oval(w*.28, h*.24, w*.52, h*.76, outline="#292b5a", width=1, tags="background")
            bg.create_oval(w*.31, h*.31, w*.49, h*.69, outline="#71336f", width=1, tags="background")
            bg.create_text(w*.095, h*.30, text="✦", fill="#59f0e8", font=("Arial", 34, "bold"), tags="background")
            bg.create_text(w*.43, h*.29, text="✦", fill="#ffd66b", font=("Arial", 20, "bold"), tags="background")
            bg.create_text(w*.082, h*.385, anchor="w", text="MẠNG XÃ HỘI CÓ TRÁCH NHIỆM", fill="#59f0e8", font=("Arial", 9, "bold"), tags="background")
            bg.create_text(w*.08, h*.49, anchor="w", text="Trạm lọc", fill="#f6f5ff", font=("Arial", 40, "bold"), tags="background")
            bg.create_text(w*.08, h*.59, anchor="w", text="thông tin", fill="#ff6fcf", font=("Arial", 40, "bold"), tags="background")
            bg.create_text(w*.08, h*.70, anchor="w", text="Tiếp nhận, kiểm chứng và chia sẻ có chọn lọc.", fill="#c6c9ef", font=("Arial", 13), tags="background")
        bg.bind("<Configure>", paint)

        panel = tk.Frame(frame, bg="#202345", highlightbackground="#414578", highlightthickness=1)
        panel.place(relx=.60, rely=.055, relwidth=.34, relheight=.89)
        inner = tk.Frame(panel, bg="#202345")
        inner.pack(fill="both", expand=True, padx=28, pady=28)
        title = "Tạo tài khoản" if register else "Chào mừng bạn"
        tk.Label(inner, text=title, font=("Arial", 21, "bold"), fg="#f6f5ff", bg="#202345", anchor="w").pack(fill="x", pady=(8, 8))
        subtitle = "Đăng ký để bắt đầu hành trình kiểm chứng thông tin." if register else "Đăng nhập để bắt đầu hành trình kiểm chứng thông tin."
        tk.Label(inner, text=subtitle, font=("Arial", 10), fg="#b8c3f2", bg="#202345", wraplength=360, justify="left", anchor="w").pack(fill="x", pady=(0, 22))

        tabs = tk.Frame(inner, bg="#11142e", padx=5, pady=5)
        tabs.pack(fill="x", pady=(0, 18))
        tk.Button(tabs, text="Đăng nhập", command=lambda: self.show_login(False), font=("Arial", 10, "bold"), relief="flat", bd=0, cursor="hand2", bg="#59eee6" if not register else "#11142e", fg="#10152e" if not register else "#c5c9ed", activebackground="#59eee6", activeforeground="#10152e", pady=10).pack(side="left", fill="x", expand=True)
        tk.Button(tabs, text="Đăng ký", command=lambda: self.show_login(True), font=("Arial", 10, "bold"), relief="flat", bd=0, cursor="hand2", bg="#59eee6" if register else "#11142e", fg="#10152e" if register else "#c5c9ed", activebackground="#59eee6", activeforeground="#10152e", pady=10).pack(side="left", fill="x", expand=True)

        def field_label(text):
            tk.Label(inner, text=text, font=("Arial", 9, "bold"), fg="#f6f5ff", bg="#202345", anchor="w").pack(fill="x", pady=(0, 6))
        field_label("Tên đăng nhập")
        username_entry = tk.Entry(inner, font=("Arial", 11), relief="flat", bd=0, bg="#11152f", fg="#f6f5ff", insertbackground="#59f0e8", highlightthickness=1, highlightbackground="#414875", highlightcolor="#59f0e8")
        username_entry.pack(fill="x", ipady=11, pady=(0, 16))
        username_entry.insert(0, "")
        username_entry.configure(insertwidth=2)
        field_label("Mật khẩu")
        password_entry = tk.Entry(inner, font=("Arial", 11), show="●", relief="flat", bd=0, bg="#11152f", fg="#f6f5ff", insertbackground="#59f0e8", highlightthickness=1, highlightbackground="#414875", highlightcolor="#59f0e8")
        password_entry.pack(fill="x", ipady=11, pady=(0, 12))
        if register:
            tk.Label(inner, text="Mật khẩu nên có ít nhất 6 ký tự.", font=("Arial", 8), fg="#8e9bd0", bg="#202345", anchor="w").pack(fill="x", pady=(0, 6))
        status = tk.Label(inner, text="", font=("Arial", 9), fg="#ff718d", bg="#202345", wraplength=350, justify="left", anchor="w")
        status.pack(fill="x", pady=(0, 8))

        def submit():
            username = username_entry.get().strip()
            password = password_entry.get()
            if register:
                if len(password) < 6:
                    status.config(text="Mật khẩu cần ít nhất 6 ký tự.", fg="#ff718d")
                    return
                ok, message = self.register_user(username, password)
                status.config(text=message, fg="#75f0ae" if ok else "#ff718d")
                if ok:
                    tk.messagebox.showinfo("Đăng ký thành công", "Tài khoản đã được tạo. Hãy đăng nhập để bắt đầu!", parent=self.root)
                    self.show_login(False)
            else:
                account, message = self.verify_password(username, password)
                if account:
                    self.clear_frame()
                    self.root.title("Trạm lọc thông tin | Tiếp nhận, kiểm chứng và chia sẻ có chọn lọc")
                    self.app = CopyQuest(self.root, account["username"], account["role"], self)
                else:
                    status.config(text=message, fg="#ff718d")
        action_text = "Tạo tài khoản   →" if register else "Đăng nhập   →"
        tk.Button(inner, text=action_text, command=submit, font=("Arial", 11, "bold"), bg="#5af0df", fg="#10152e", activebackground="#8af5d9", activeforeground="#10152e", relief="flat", bd=0, cursor="hand2", pady=13).pack(fill="x", pady=(2, 18))
        tk.Button(inner, text="Mở trên điện thoại bằng mã QR", command=self.open_qr_portal, font=("Arial", 9, "bold"), bg="#302e5d", fg="#59f0e8", activebackground="#414078", activeforeground="#ffffff", relief="flat", bd=0, cursor="hand2", pady=9).pack(fill="x", pady=(0, 8))
        foot = "Tài khoản được lưu cục bộ trên thiết bị này. Không dùng mật khẩu quan trọng hoặc chia sẻ với người khác."
        tk.Label(inner, text=foot, font=("Arial", 8), fg="#8e9bd0", bg="#202345", wraplength=360, justify="left", anchor="w").pack(fill="x", side="bottom", pady=(12, 0))
        username_entry.focus_set()
        password_entry.bind("<Return>", lambda event: submit())
        username_entry.bind("<Return>", lambda event: password_entry.focus_set())

    def open_qr_portal(self):
        """Mở cổng web đăng nhập/đăng ký cho điện thoại cùng Wi-Fi và hiển thị QR."""
        try:
            import qrcode
            from flask import Flask, request, session, redirect, url_for, render_template_string
            from PIL import ImageTk
        except ImportError:
            install = tk.messagebox.askyesno(
                "Cần cài thư viện để tạo QR",
                "Ứng dụng sẽ tự cài các thư viện cần thiết (Flask, qrcode, Pillow) để tạo mã QR và mở trang đăng nhập trên điện thoại.\n\n"
                "Hãy đảm bảo máy tính có Internet. Cài ngay bây giờ?",
                parent=self.root,
            )
            if not install:
                tk.messagebox.showinfo(
                    "Chưa tạo mã QR",
                    "Bạn có thể bấm nút QR lần nữa sau, hoặc chạy lệnh:\npython -m pip install flask qrcode[pil] pillow",
                    parent=self.root,
                )
                return
            try:
                progress = tk.Toplevel(self.root)
                progress.title("Đang cài thư viện QR")
                progress.configure(bg="#171735")
                progress.resizable(False, False)
                tk.Label(progress, text="ĐANG CHUẨN BỊ MÃ QR", font=("Arial", 12, "bold"), fg="#59f0e8", bg="#171735").pack(padx=28, pady=(22, 8))
                tk.Label(progress, text="Đang cài thư viện cần thiết...\nVui lòng chờ và giữ ứng dụng mở.", font=("Arial", 10), fg="#f6f5ff", bg="#171735", justify="center").pack(padx=28, pady=(0, 22))
                progress.update_idletasks()
                result = subprocess.run(
                    [sys.executable, "-m", "pip", "install", "flask", "qrcode[pil]", "pillow"],
                    capture_output=True, text=True, timeout=240
                )
                progress.destroy()
                if result.returncode != 0:
                    details = (result.stderr or result.stdout or "Không rõ nguyên nhân")[-1400:]
                    tk.messagebox.showerror(
                        "Chưa cài được thư viện",
                        "Không thể tự cài thư viện. Kiểm tra Internet rồi chạy lệnh sau trong Terminal/Command Prompt:\n\n"
                        "python -m pip install flask qrcode[pil] pillow\n\nChi tiết:\n" + details,
                        parent=self.root,
                    )
                    return
                tk.messagebox.showinfo("Cài đặt hoàn tất", "Đã cài thư viện QR. Bấm OK để tạo mã QR.", parent=self.root)
                return self.open_qr_portal()
            except Exception as exc:
                try:
                    progress.destroy()
                except Exception:
                    pass
                tk.messagebox.showerror("Lỗi cài thư viện", f"Không thể cài tự động: {exc}\n\nChạy: python -m pip install flask qrcode[pil] pillow", parent=self.root)
                return

        if not getattr(self, "web_server_started", False):
            web_app = Flask("tram_loc_thong_tin_mobile")
            web_app.secret_key = secrets.token_hex(32)
            web_app.config["SESSION_COOKIE_HTTPONLY"] = True
            web_app.config["SESSION_COOKIE_SAMESITE"] = "Lax"
            db_path = self.db_path

            PAGE = r"""<!doctype html><html lang="vi"><meta name="viewport" content="width=device-width,initial-scale=1"><meta charset="utf-8"><title>Trạm Lọc Thông Tin</title><style>
            *{box-sizing:border-box}body{margin:0;min-height:100vh;color:#f6f5ff;font-family:Arial,sans-serif;background:radial-gradient(circle at 15% 20%,#292454 0,transparent 34%),linear-gradient(145deg,#10132f,#211638 75%,#24134b);display:flex;align-items:center;justify-content:center;padding:22px}.wrap{width:min(100%,440px);background:#202345;border:1px solid #414578;border-radius:24px;padding:28px;box-shadow:0 24px 70px #07081c88}h1{font-size:27px;margin:0 0 8px}h1 span{color:#ff6fcf}.sub{color:#b8c3f2;line-height:1.6;margin-bottom:22px}.tabs{display:flex;gap:6px;background:#11142e;border-radius:14px;padding:5px;margin-bottom:20px}.tabs a{flex:1;text-align:center;padding:13px 5px;text-decoration:none;border-radius:10px;color:#c5c9ed;font-weight:bold}.tabs a.on{background:#59eee6;color:#10152e}label{display:block;font-size:13px;font-weight:bold;margin:15px 0 7px}input{width:100%;padding:15px;border-radius:12px;border:1px solid #414875;background:#11152f;color:white;font-size:16px;outline:none}input:focus{border-color:#59f0e8}button{width:100%;padding:15px;margin-top:20px;border:0;border-radius:12px;background:linear-gradient(90deg,#59f0e8,#80f3d1);color:#10152e;font-weight:800;font-size:16px}.msg{color:#ff9db5;line-height:1.5}.note{font-size:12px;color:#8e9bd0;line-height:1.5;margin-top:20px}.pill{display:inline-block;color:#59f0e8;border:1px solid #414875;border-radius:99px;padding:7px 11px;font-size:12px;margin-bottom:16px}</style><div class="wrap"><div class="pill">✦ MẠNG XÃ HỘI CÓ TRÁCH NHIỆM</div><h1>Trạm lọc <span>thông tin</span></h1><div class="sub">Tiếp nhận, kiểm chứng và chia sẻ có chọn lọc.</div>{% if user %}<h2>Chào {{user}}</h2><p class="sub">Bạn đã đăng nhập thành công. Đây là cổng truy cập trên điện thoại của Trạm Lọc Thông Tin.</p><p>Điểm XP: <b>{{xp}}</b> · Cấp độ: <b>{{level}}</b></p><p class="note">Các trò chơi tương tác đầy đủ hiện vẫn chạy trong ứng dụng Python trên máy tính.</p><a href="/logout" style="color:#59f0e8">Đăng xuất</a>{% else %}<div class="tabs"><a href="/?mode=login" class="{{'on' if mode=='login' else ''}}">Đăng nhập</a><a href="/?mode=register" class="{{'on' if mode=='register' else ''}}">Đăng ký</a></div>{% if msg %}<p class="msg">{{msg}}</p>{% endif %}<form method="post"><input type="hidden" name="mode" value="{{mode}}"><label>Tên đăng nhập</label><input name="username" required maxlength="40" autocomplete="username" placeholder="Nhập tên đăng nhập"><label>Mật khẩu</label><input name="password" type="password" required minlength="6" autocomplete="{{'new-password' if mode=='register' else 'current-password'}}" placeholder="Ít nhất 6 ký tự"><button>{{'Tạo tài khoản →' if mode=='register' else 'Đăng nhập →'}}</button></form><div class="note">Bản thử nghiệm trong mạng Wi-Fi nội bộ. Không dùng mật khẩu quan trọng hoặc chia sẻ mã QR ra ngoài mạng riêng.</div>{% endif %}</div></html>"""

            def connect_db():
                con = sqlite3.connect(db_path)
                con.row_factory = sqlite3.Row
                return con
            def password_hash(password, salt=None):
                salt = salt or secrets.token_bytes(16)
                digest = hashlib.pbkdf2_hmac("sha256", password.encode("utf-8"), salt, 240000)
                return salt.hex(), digest.hex()
            @web_app.route("/", methods=["GET", "POST"])
            def mobile_home():
                mode = request.values.get("mode", "login")
                if mode not in ("login", "register"):
                    mode = "login"
                msg = ""
                if request.method == "POST":
                    username = request.form.get("username", "").strip()
                    password = request.form.get("password", "")
                    if not username or not password:
                        msg = "Vui lòng nhập đầy đủ thông tin."
                    elif mode == "register":
                        if len(password) < 6:
                            msg = "Mật khẩu cần ít nhất 6 ký tự."
                        else:
                            salt, digest = password_hash(password)
                            try:
                                with connect_db() as con:
                                    con.execute("INSERT INTO users(username,salt,password_hash,role,enabled) VALUES(?,?,?,'user',1)", (username,salt,digest))
                                msg = "Đăng ký thành công! Hãy chuyển sang Đăng nhập."
                                mode = "login"
                            except sqlite3.IntegrityError:
                                msg = "Tên đăng nhập đã tồn tại. Hãy chọn tên khác."
                    else:
                        with connect_db() as con:
                            row = con.execute("SELECT * FROM users WHERE username=? COLLATE NOCASE", (username,)).fetchone()
                        if not row:
                            msg = "Tên đăng nhập hoặc mật khẩu không đúng."
                        elif not row["enabled"]:
                            msg = "Tài khoản này đang bị khóa."
                        else:
                            _, digest = password_hash(password, bytes.fromhex(row["salt"]))
                            if not hmac.compare_digest(digest, row["password_hash"]):
                                msg = "Tên đăng nhập hoặc mật khẩu không đúng."
                            else:
                                session["username"] = row["username"]
                                return redirect(url_for("mobile_home"))
                username = session.get("username")
                xp, level = (0,1)
                if username:
                    with connect_db() as con:
                        row = con.execute("SELECT xp,level FROM users WHERE username=? COLLATE NOCASE", (username,)).fetchone()
                    if row:
                        xp, level = row["xp"], row["level"]
                    else:
                        session.clear(); username = None
                return render_template_string(PAGE, mode=mode, msg=msg, user=username, xp=xp, level=level)
            @web_app.route("/logout")
            def mobile_logout():
                session.clear()
                return redirect(url_for("mobile_home"))

            def run_server():
                web_app.run(host="0.0.0.0", port=8765, debug=False, use_reloader=False, threaded=True)
            threading.Thread(target=run_server, daemon=True).start()
            self.web_server_started = True
            self.web_app = web_app

        try:

            sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
            try:
                sock.connect(("8.8.8.8", 80))
                lan_ip = sock.getsockname()[0]
            finally:
                sock.close()
            url = f"http://{lan_ip}:8765/"
            qr = qrcode.QRCode(box_size=7, border=3)
            qr.add_data(url); qr.make(fit=True)
            qr_image = qr.make_image(fill_color="#171735", back_color="white").convert("RGB")
            photo = ImageTk.PhotoImage(qr_image)
            win = tk.Toplevel(self.root)
            win.title("Mở Trạm Lọc Thông Tin bằng QR")
            win.configure(bg="#171735")
            win.resizable(False, False)
            tk.Label(win, text="QUÉT MÃ QR BẰNG ĐIỆN THOẠI", font=("Arial", 14, "bold"), fg="#59f0e8", bg="#171735").pack(padx=22, pady=(20, 8))
            tk.Label(win, image=photo, bg="white").pack(padx=24, pady=10)
            tk.Label(win, text=url, font=("Arial", 10, "bold"), fg="#ff6fcf", bg="#171735").pack(padx=16, pady=6)
            tk.Label(win, text="Điện thoại và máy tính phải dùng cùng Wi-Fi.\nGiữ ứng dụng đang mở để QR còn hoạt động.", font=("Arial", 9), fg="#c5c9ed", bg="#171735", justify="center").pack(padx=20, pady=(0,18))
            win.qr_photo = photo
        except Exception as exc:
            tk.messagebox.showerror("Không tạo được mã QR", f"Chi tiết: {exc}", parent=self.root)

    def open_admin_panel(self):
        with self.connect() as conn:
            admins = conn.execute("SELECT role FROM users WHERE username = ? COLLATE NOCASE", (self.app.username,)).fetchone()
        if not admins or admins["role"] != "admin":
            return
        win = tk.Toplevel(self.root)
        win.title("Trạm lọc thông tin | Quản lý tài khoản")
        win.geometry("690x500")
        win.configure(bg="#10132f")
        tk.Label(win, text="BẢNG QUẢN LÝ TÀI KHOẢN", font=("Arial", 16, "bold"), fg="#59f0e8", bg="#10132f").pack(pady=(16, 5))
        tk.Label(win, text="Chỉ admin mới có thể thực hiện các thao tác này.", font=("Arial", 9), fg="#b8b9dd", bg="#10132f").pack(pady=(0, 10))
        list_frame = tk.Frame(win, bg="#10132f")
        list_frame.pack(fill="both", expand=True, padx=18, pady=5)
        scrollbar = tk.Scrollbar(list_frame)
        scrollbar.pack(side="right", fill="y")
        listing = tk.Listbox(list_frame, font=("Consolas", 10), bg="#25234e", fg="#f6f5ff", selectbackground="#514a86", yscrollcommand=scrollbar.set, height=14)
        listing.pack(side="left", fill="both", expand=True)
        scrollbar.config(command=listing.yview)
        rows = []
        def refresh():
            listing.delete(0, "end")
            rows.clear()
            with self.connect() as conn:
                records = conn.execute("SELECT username, role, enabled, xp, level, created_at FROM users ORDER BY role DESC, username COLLATE NOCASE").fetchall()
            for row in records:
                rows.append(dict(row))
                status_text = "ĐANG HOẠT ĐỘNG" if row["enabled"] else "ĐÃ KHÓA"
                listing.insert("end", f"{row['username']:<20} | {row['role']:<5} | {status_text:<15} | XP {row['xp']:<5} | Cấp {row['level']}")
        def selected():
            choice = listing.curselection()
            if not choice:
                tk.messagebox.showinfo("Chọn tài khoản", "Hãy chọn một tài khoản trong danh sách.", parent=win)
                return None
            return rows[choice[0]]
        def toggle_account():
            row = selected()
            if not row:
                return
            if row["role"] == "admin":
                tk.messagebox.showwarning("Không thể thực hiện", "Không thể khóa tài khoản admin.", parent=win)
                return
            new_value = 0 if row["enabled"] else 1
            with self.connect() as conn:
                conn.execute("UPDATE users SET enabled = ? WHERE username = ? COLLATE NOCASE", (new_value, row["username"]))
            refresh()
        def reset_password():
            row = selected()
            if not row:
                return
            new_password = tk.simpledialog.askstring("Đặt lại mật khẩu", f"Nhập mật khẩu mới cho {row['username']}:", parent=win, show="●")
            if new_password is None:
                return
            if not new_password:
                tk.messagebox.showerror("Mật khẩu chưa hợp lệ", "Mật khẩu không được để trống.", parent=win)
                return
            salt, digest = self.hash_password(new_password)
            with self.connect() as conn:
                conn.execute("UPDATE users SET salt = ?, password_hash = ? WHERE username = ? COLLATE NOCASE", (salt, digest, row["username"]))
            tk.messagebox.showinfo("Thành công", "Đã đặt lại mật khẩu.", parent=win)
        def delete_account():
            row = selected()
            if not row:
                return
            if row["role"] == "admin":
                tk.messagebox.showwarning("Không thể thực hiện", "Không thể xóa tài khoản admin.", parent=win)
                return
            if tk.messagebox.askyesno("Xác nhận xóa", f"Xóa tài khoản {row['username']} và dữ liệu XP của tài khoản này?", parent=win):
                with self.connect() as conn:
                    conn.execute("DELETE FROM users WHERE username = ? COLLATE NOCASE", (row["username"],))
                refresh()
        buttons = tk.Frame(win, bg="#10132f")
        buttons.pack(pady=12)
        tk.Button(buttons, text="Khóa / Mở khóa", command=toggle_account, bg="#ffd66b", fg="#171735", relief="flat", padx=10, pady=8).grid(row=0, column=0, padx=5)
        tk.Button(buttons, text="Đặt lại mật khẩu", command=reset_password, bg="#59f0e8", fg="#171735", relief="flat", padx=10, pady=8).grid(row=0, column=1, padx=5)
        tk.Button(buttons, text="Xóa tài khoản", command=delete_account, bg="#ff718d", fg="#171735", relief="flat", padx=10, pady=8).grid(row=0, column=2, padx=5)
        tk.Button(buttons, text="Đổi mật khẩu admin", command=lambda: self.change_admin_password(win, refresh), bg="#a98bff", fg="#171735", relief="flat", padx=10, pady=8).grid(row=0, column=3, padx=5)
        refresh()

    def change_admin_password(self, parent, refresh_callback):
        old_password = tk.simpledialog.askstring("Đổi mật khẩu admin", "Nhập mật khẩu admin hiện tại:", parent=parent, show="●")
        if old_password is None:
            return
        account, message = self.verify_password(self.app.username, old_password)
        if not account or account["role"] != "admin":
            tk.messagebox.showerror("Không thành công", "Mật khẩu hiện tại không đúng.", parent=parent)
            return
        new_password = tk.simpledialog.askstring("Đổi mật khẩu admin", "Nhập mật khẩu mới (ít nhất 8 ký tự):", parent=parent, show="●")
        if new_password is None:
            return
        if len(new_password) < 8:
            tk.messagebox.showerror("Mật khẩu chưa hợp lệ", "Mật khẩu admin cần ít nhất 8 ký tự.", parent=parent)
            return
        salt, digest = self.hash_password(new_password)
        with self.connect() as conn:
            conn.execute("UPDATE users SET salt = ?, password_hash = ? WHERE username = ? COLLATE NOCASE AND role = 'admin'", (salt, digest, self.app.username))
        tk.messagebox.showinfo("Thành công", "Đã đổi mật khẩu admin.", parent=parent)

# Che do web cho dien thoai: chay file .py, sau do mo lien ket hien ra.
import json
import time
import webbrowser
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlparse

MOBILE_PORT = 8765
MOBILE_DB = os.path.join(os.path.dirname(os.path.abspath(__file__)), "tram_loc_mobile_accounts.json")
MOBILE_HTML = r"""<!doctype html>
<html lang="vi"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover"><title>Trạm Lọc Thông Tin</title><style>
:root{--bg:#15122f;--panel:#202447;--line:#414879;--cyan:#59f0e8;--pink:#ff6fcf;--muted:#c2c7ef}*{box-sizing:border-box}body{margin:0;color:#f7f5ff;font-family:system-ui,-apple-system,Segoe UI,sans-serif;background:radial-gradient(circle at 20% 20%,#292052,transparent 38%),linear-gradient(145deg,#11132f,#24133b);min-height:100vh}button,input{font:inherit}button{cursor:pointer;border:0;border-radius:13px;padding:13px 16px;font-weight:750;color:#10132f;background:linear-gradient(100deg,var(--cyan),#8bf5d7)}input{width:100%;padding:14px;border-radius:12px;border:1px solid var(--line);background:#10152e;color:white;margin:6px 0 14px}.shell{max-width:1150px;margin:auto;padding:clamp(16px,4vw,42px)}.hero{display:grid;grid-template-columns:1.05fr .95fr;gap:28px;align-items:center;min-height:90vh}.brand small{letter-spacing:2px;color:var(--cyan);font-weight:800}.brand h1{font-size:clamp(2.5rem,6vw,5.8rem);line-height:1.02;margin:25px 0}.brand h1 span{color:var(--pink)}.brand p,.muted{color:var(--muted);line-height:1.7;font-size:clamp(1rem,2vw,1.25rem)}.panel,.card{background:rgba(32,36,71,.94);border:1px solid var(--line);border-radius:24px;padding:clamp(18px,4vw,32px);box-shadow:0 20px 55px #09091c55}.panel h2{font-size:clamp(1.7rem,4vw,2.4rem);margin-top:0}.tabs,.actions{display:flex;gap:10px;flex-wrap:wrap;margin:15px 0}.tabs button{flex:1;background:#11152f;color:var(--muted)}.tabs button.active{background:var(--cyan);color:#11152f}.hidden{display:none!important}.apphead{display:flex;justify-content:space-between;align-items:center;gap:12px;flex-wrap:wrap;margin-bottom:24px}.apphead h1{font-size:clamp(1.5rem,4vw,2.5rem);margin:0}.grid{display:grid;grid-template-columns:repeat(2,minmax(0,1fr));gap:16px}.card{padding:22px;min-height:185px}.card h3{margin:8px 0}.card p{color:var(--muted);line-height:1.55}.link{color:var(--cyan);font-weight:800;cursor:pointer}.content{margin-top:20px}.maze{display:grid;grid-template-columns:repeat(7,minmax(25px,42px));gap:4px;justify-content:center;margin:18px auto}.tile{aspect-ratio:1;border-radius:6px;background:#11152e;border:1px solid #39406b;display:grid;place-items:center}.wall{background:#6669a0}.player{background:#59f0e8;color:#111;font-weight:900}.exit{background:#ff6fcf;color:#111}.answers{display:grid;gap:10px;margin-top:15px}.answers button{background:#30365f;color:white;text-align:left}.msg{padding:12px;border-radius:12px;background:#11152f;margin:12px 0}.commit{display:flex;gap:10px;align-items:flex-start;padding:12px 0;border-bottom:1px solid #414879}.commit input{width:auto;margin-top:5px}.foot{font-size:.88rem;color:#aab1df;margin-top:24px}.tree{display:grid;gap:14px;margin:20px 0}.tree-root{padding:16px;border:2px solid var(--cyan);border-radius:16px;text-align:center;background:#171b3c;font-weight:800}.tree-branches{display:grid;grid-template-columns:repeat(2,minmax(0,1fr));gap:12px}.tree-node{border:1px solid var(--line);border-top:3px solid var(--pink);border-radius:14px;padding:14px;background:#25274d}.tree-node p{margin-bottom:0;color:var(--muted);line-height:1.55}.answers button:disabled{opacity:1;cursor:default}.maze{width:min(100%,340px)}.tile{min-width:0;font-size:clamp(.65rem,2.5vw,.95rem)}#analysis p{line-height:1.65}@media(max-width:520px){.tree-branches{grid-template-columns:1fr}.actions button{flex:1;min-width:54px}.panel{padding:18px}.grid{grid-template-columns:1fr}}@media(max-width:760px){.shell{padding:15px}.hero{grid-template-columns:1fr;min-height:100vh;gap:18px;align-content:center}.brand h1{font-size:clamp(2.5rem,12vw,4.3rem);margin:18px 0}.brand p{font-size:1rem}.brand .orbit{display:none}.panel{border-radius:20px}.grid{grid-template-columns:1fr}.card{min-height:0}.apphead{align-items:flex-start}.maze{grid-template-columns:repeat(7,minmax(24px,1fr));max-width:320px}.tile{font-size:.8rem}}
</style></head><body><main class="shell"><section id="auth" class="hero"><div class="brand"><small>✦ MẠNG XÃ HỘI CÓ TRÁCH NHIỆM</small><h1>Trạm lọc <span>thông tin</span></h1><p>Tiếp nhận, kiểm chứng và chia sẻ có chọn lọc.</p></div><div class="panel"><h2 id="authTitle">Chào mừng bạn</h2><p class="muted">Đăng nhập để bắt đầu hành trình kiểm chứng thông tin.</p><div class="tabs"><button id="loginTab" class="active" onclick="setMode('login')">Đăng nhập</button><button id="regTab" onclick="setMode('register')">Đăng ký</button></div><label>Tên đăng nhập</label><input id="username" autocomplete="username" placeholder="Nhập tên đăng nhập"><label>Mật khẩu</label><input id="password" type="password" autocomplete="current-password" placeholder="Nhập mật khẩu"><button style="width:100%" onclick="authSubmit()" id="authBtn">Đăng nhập →</button><div id="authMsg" class="msg hidden"></div><p class="foot">Tài khoản được lưu trên máy đang chạy máy chủ. Không dùng mật khẩu quan trọng hoặc chia sẻ với người khác.</p></div></section>
<section id="app" class="hidden"><header class="apphead"><div><small style="color:var(--cyan);font-weight:800;letter-spacing:2px">MẠNG XÃ HỘI CÓ TRÁCH NHIỆM</small><h1>Trạm lọc <span style="color:var(--pink)">thông tin</span></h1><div class="muted" id="welcome"></div></div><button onclick="logout()">Đăng xuất</button></header><div class="grid"><article class="card"><div>🌳</div><h3>Định nghĩa Copypasta</h3><p>Tìm hiểu khái niệm và cách nhận diện hành vi sao chép máy móc.</p><div class="link" onclick="showFeature('definition')">Mở nội dung →</div></article><article class="card"><div>🧩</div><h3>Mê cung ngôn từ</h3><p>Trả lời câu hỏi Copypasta để tìm đường ra mê cung.</p><div class="link" onclick="showFeature('maze')">Bắt đầu →</div></article><article class="card"><div>🔎</div><h3>Kiểm tra câu nói</h3><p>Phân tích dấu hiệu, ngữ cảnh và khả năng là Copypasta.</p><div class="link" onclick="showFeature('check')">Kiểm tra →</div></article><article class="card"><div>✨</div><h3>Cam kết 7 ngày</h3><p>Mỗi ngày một thông điệp để chia sẻ có chọn lọc.</p><div class="link" onclick="showFeature('commit')">Mở trạm →</div></article></div><section id="feature" class="panel content hidden"></section></section></main><script>
let mode='login',user='';const $=id=>document.getElementById(id);function setMode(m){mode=m;$('loginTab').classList.toggle('active',m==='login');$('regTab').classList.toggle('active',m==='register');$('authTitle').textContent=m==='login'?'Chào mừng bạn':'Tạo tài khoản';$('authBtn').textContent=m==='login'?'Đăng nhập →':'Đăng ký →';$('authMsg').classList.add('hidden')}
async function api(path,data){let r=await fetch(path,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(data)});return r.json()}async function authSubmit(){let u=$('username').value.trim(),p=$('password').value;if(!u||!p){notice('Vui lòng nhập tên đăng nhập và mật khẩu.');return}try{let r=await api('/auth',{mode,username:u,password:p});if(!r.ok){notice(r.message);return}user=u;$('auth').classList.add('hidden');$('app').classList.remove('hidden');$('welcome').textContent='Xin chào, '+u+'!';}catch(e){notice('Không kết nối được máy chủ. Hãy chạy file Python trên thiết bị chủ.')}}function notice(t){$('authMsg').textContent=t;$('authMsg').classList.remove('hidden')}function logout(){$('app').classList.add('hidden');$('auth').classList.remove('hidden');$('password').value='';setMode('login')}
const phrases=[
 {text:'Dạ gỗ xà cừ ạ',meaning:'Câu trả lời cố ý lệch ngữ cảnh, thường được nhắc lại như một câu đùa/mẫu bình luận.',level:'cao',context:'Nếu được chép lại dưới nhiều bài đăng không liên quan, câu có dấu hiệu lặp lại máy móc.'},
 {text:'Ai hỏi',meaning:'Câu đáp ngắn dùng để trêu hoặc gạt bỏ ý kiến của người khác.',level:'cao',context:'Có thể là tiếng lóng thông thường; chỉ nghiêng về Copypasta khi bị lặp lại theo khuôn bất kể ngữ cảnh.'},
 {text:'Ủa alo',meaning:'Cách thể hiện ngạc nhiên hoặc yêu cầu chú ý.',level:'trung bình',context:'Cụm này có nghĩa giao tiếp rõ; cần xem tần suất lặp và cách dùng.'},
 {text:'Bá khí',meaning:'Từ/cụm từ khen sự mạnh mẽ, ấn tượng hoặc có khí chất.',level:'trung bình',context:'Một từ lóng đơn lẻ không đủ kết luận là Copypasta.'},
 {text:'Hồi chiều trời mưa',meaning:'Một câu thông thường, tự thân có nghĩa rõ ràng.',level:'chưa',context:'Không có dấu hiệu lặp theo mẫu nếu chỉ xuất hiện đơn lẻ.'},
 {text:'farm aura',meaning:'Cụm từ mạng dùng để nói về việc tích lũy hình tượng/phong thái.',level:'trung bình',context:'Cần ngữ cảnh và bằng chứng lặp lại để phân biệt tiếng lóng với Copypasta.'},
 {text:'ài ố sì mà',meaning:'Cách viết mô phỏng âm thanh/câu nói đang lan truyền.',level:'cao',context:'Nếu được sao chép nguyên mẫu ở nhiều ngữ cảnh không liên quan, dấu hiệu lặp lại mạnh hơn.'},
 {text:'Tôi thích đọc sách vào buổi tối',meaning:'Câu diễn đạt thông thường, có ý nghĩa rõ.',level:'chưa',context:'Chưa thấy dấu hiệu đặc trưng khi không có bằng chứng lặp lại theo trào lưu.'}
];
const mazeQuestions=[
 {q:'Nhiều bình luận chép nguyên một câu vô nghĩa dưới các bài đăng không liên quan. Dấu hiệu chính là gì?',a:['Sao chép và lặp lại máy móc','Mọi người đều hiểu câu đó','Đó chắc chắn là thông tin hữu ích'],c:0,why:'Copypasta trong đề tài này nhấn mạnh việc sao chép và lặp lại một mẫu câu, nhất là khi câu bị dùng bất kể ngữ cảnh.'},
 {q:'Một người dùng tiếng lóng đúng tình huống một lần. Có thể kết luận ngay là Copypasta không?',a:['Có, mọi tiếng lóng đều là Copypasta','Không nhất thiết; cần xem dấu hiệu sao chép/lặp lại','Có, vì câu ngắn'],c:1,why:'Tiếng lóng không tự động là Copypasta. Cần có dấu hiệu câu được lặp lại theo mẫu một cách máy móc.'},
 {q:'Bạn thấy cùng một câu xuất hiện dưới rất nhiều bài đăng khác nhau. Nên xem xét điều gì?',a:['Mẫu lặp và ngữ cảnh sử dụng','Chỉ số lượt thích','Ảnh đại diện người đăng'],c:0,why:'Mẫu câu lặp lại và sự phù hợp với ngữ cảnh là những yếu tố hữu ích hơn lượt thích hoặc danh tính người đăng.'},
 {q:'Một câu viral có ý nghĩa rõ và được dùng đúng lúc. Điều nào hợp lý nhất?',a:['Chắc chắn có hại','Chắc chắn là Copypasta','Chưa thể kết luận chỉ từ độ viral'],c:2,why:'Độ phổ biến không đủ để kết luận. Cần xem câu có bị sao chép máy móc hay không và được dùng trong hoàn cảnh nào.'},
 {q:'Điều nào gần nhất với định nghĩa Copypasta của đề tài?',a:['Tự viết một nhận xét riêng','Sao chép máy móc và lặp lại câu nói viral','Chia sẻ thông tin có nguồn'],c:1,why:'Trọng tâm của đề tài là hành vi chép và lặp lại nhiều lần những câu nói/mẫu bình luận, đôi khi vô nghĩa hoặc sai ngữ cảnh.'},
 {q:'Khi chưa có dữ liệu về tần suất một câu xuất hiện, nên nói gì?',a:['Câu chắc chắn viral toàn mạng','Không cần kiểm chứng','Mức độ phổ biến chưa được xác minh'],c:2,why:'Không nên tự khẳng định độ phổ biến khi thiếu dữ liệu có nguồn và thời điểm cụ thể.'},
 {q:'Một meme được chia sẻ nhiều lần có tự động là Copypasta không?',a:['Không nhất thiết; cần xét dạng nội dung và cách lặp','Có, tất cả meme đều là Copypasta','Có, vì có hình ảnh'],c:0,why:'Meme và Copypasta có thể giao nhau nhưng không đồng nghĩa. Cần xem có mẫu văn bản/câu nói được sao chép lặp lại hay không.'},
 {q:'Cách phản ứng có trách nhiệm khi gặp câu viral là gì?',a:['Chép lại ngay để hòa nhập','Hiểu nghĩa, kiểm tra ngữ cảnh rồi quyết định','Dùng trong mọi cuộc trò chuyện'],c:1,why:'Hiểu ý nghĩa và hoàn cảnh giúp tránh lặp lại câu không phù hợp hoặc gây hiểu lầm.'}
];
let pos=0,hearts=3,steps=0,checkedDays=[],mazeIndex=0,mazeCanMove=false,mx=1,my=1,mazeWon=false;
function showFeature(which){let f=$('feature');f.classList.remove('hidden');if(which==='definition'){f.innerHTML='<h2>🌳 Cây định nghĩa Copypasta</h2><p class="muted">Copypasta là hành vi sao chép một cách máy móc và lặp lại nhiều lần trên mạng xã hội những câu nói vô tri, không có nghĩa hoặc sai ngữ cảnh.</p><div class="tree"><div class="tree-root">🌳 COPypasta: sao chép + lặp lại máy móc</div><div class="tree-branches"><div class="tree-node"><b>1. Nguồn xuất hiện</b><p>Câu nói, bình luận hoặc tiếng lóng viral trên mạng xã hội.</p></div><div class="tree-node"><b>2. Hành vi</b><p>Nhiều người sao chép, dán lại hoặc lặp lại theo cùng một khuôn.</p></div><div class="tree-node"><b>3. Ngữ cảnh</b><p>Câu có thể vô nghĩa, lệch ngữ cảnh hoặc được dùng mà không hiểu rõ ý.</p></div><div class="tree-node"><b>4. Cách nhận diện</b><p>Kết hợp mẫu lặp, ý nghĩa, ngữ cảnh và bằng chứng phổ biến.</p></div></div><div class="tree-root">🌱 Lựa chọn có trách nhiệm: hiểu nghĩa trước khi chia sẻ</div></div><p>Không phải mọi tiếng lóng, meme hay câu viral đều là Copypasta. Cần đánh giá cách dùng cụ thể, không vội phán xét người dùng.</p><button onclick="hideFeature()">Quay lại trang chủ</button>'}else if(which==='check'){f.innerHTML='<h2>🔎 Kiểm tra câu nói Copypasta</h2><p class="muted">Nhập câu cần xem xét. Phân tích gồm dấu hiệu lặp, ý nghĩa/ngữ cảnh, mức độ phổ biến và khả năng phát triển thành mẫu Copypasta.</p><label>Nhập câu cần kiểm tra</label><textarea id="phrase" rows="3" style="width:100%;padding:14px;border-radius:12px;border:1px solid var(--line);background:#10152e;color:white;margin:6px 0 14px" placeholder="Nhập câu nói hoặc bình luận"></textarea><button onclick="analyze()">Phân tích câu</button><div id="analysis" class="msg hidden"></div><button onclick="hideFeature()">Quay lại</button>'}else if(which==='commit'){let msgs=['Đừng để một câu viral nói thay suy nghĩ của bạn.','Bạn có thể hòa nhập mà không đánh mất tiếng nói riêng.','Trước khi nhấn gửi, hãy tự hỏi: mình muốn truyền đi điều gì?','Chia sẻ có chọn lọc giúp cuộc trò chuyện có ý nghĩa hơn.','Hãy kiểm tra ngữ cảnh trước khi lặp lại câu nói.','Tôn trọng người khác bằng lời nói rõ ràng, phù hợp.','Tự tin nêu suy nghĩ của chính mình.'];f.innerHTML='<h2>✨ Cam kết 7 ngày</h2><p class="muted">Mỗi ngày đọc thông điệp và đánh dấu khi bạn đã thực hiện.</p>'+msgs.map((m,i)=>`<label class="commit"><input type="checkbox" ${checkedDays.includes(i)?'checked':''} onchange="toggleDay(${i},this.checked)"><span><b>Ngày ${i+1}</b><br>${m}</span></label>`).join('')+'<div class="msg">Đã hoàn thành '+checkedDays.length+'/7 ngày.</div><button onclick="hideFeature()">Quay lại</button>'}else if(which==='maze'){renderMaze()}f.scrollIntoView({behavior:'smooth',block:'start'})}
function hideFeature(){$('feature').classList.add('hidden')}
function normalize(s){return s.toLocaleLowerCase('vi').normalize('NFC').replace(/[.,!?;:"“”'()]/g,'').replace(/\s+/g,' ').trim()}
function analyze(){let v=$('phrase').value.trim(),out=$('analysis');if(!v){out.textContent='Hãy nhập một câu trước khi phân tích.';out.classList.remove('hidden');return}let nv=normalize(v),match=phrases.find(x=>normalize(x.text)===nv),hasRepeat=/(lặp lại|sao chép|viral|trend|bình luận|meme|copypasta)/i.test(v),short=v.split(/\s+/).length<=4;let level=match?match.level:(hasRepeat?'trung bình':'chưa');let title=level==='cao'?'KHẢ NĂNG CAO':level==='trung bình'?'TRUNG BÌNH':'CHƯA CẤU THÀNH CÂU COPYPASTA';let meaning=match?match.meaning:(short?'Câu ngắn nên có thể mang nhiều nghĩa tùy tình huống; cần thêm ngữ cảnh cụ thể.':'Câu có cấu trúc diễn đạt tương đối rõ; cần biết người nói muốn truyền đạt gì và câu xuất hiện trong tình huống nào.');let context=match?match.context:'Chưa có đủ ngữ cảnh để xác định câu được dùng đúng nghĩa, đùa vui, mỉa mai hay bị lặp lại không phù hợp.';let sign=level==='cao'?'Có dấu hiệu phù hợp với mẫu câu được nhắc lại theo khuôn; vẫn cần đối chiếu số lần lặp và ngữ cảnh thực tế.':level==='trung bình'?'Có một vài dấu hiệu có thể liên quan đến xu hướng/ngôn ngữ mạng, nhưng chưa đủ để khẳng định việc sao chép máy móc.':'Từ câu đơn lẻ hiện chưa thấy đủ dấu hiệu đặc trưng của Copypasta; không nên kết luận chỉ vì câu ngắn hoặc là tiếng lóng.';let popularity='Chưa xác minh được mức độ phổ biến bằng dữ liệu có nguồn và mốc thời gian cụ thể; không suy đoán số lượt dùng.';let growth=level==='cao'?'Khả năng phát triển: tương đối cao nếu câu dễ bắt chước, tạo phản ứng nhanh và tiếp tục được lặp lại ở nhiều bài đăng.':level==='trung bình'?'Khả năng phát triển: trung bình; có thể lan truyền nếu cộng đồng tiếp tục dùng cùng khuôn câu trong các tình huống tương tự.':'Khả năng phát triển: hiện chưa rõ; cần theo dõi việc câu có được nhiều người lặp lại theo cùng một mẫu hay không.';out.innerHTML='<h3>Kết quả: '+title+'</h3><p><b>1. Dấu hiệu Copypasta</b><br>'+sign+'</p><p><b>2. Ý nghĩa và ngữ cảnh</b><br>'+meaning+' '+context+'</p><p><b>3. Mức độ phổ biến</b><br>'+popularity+'</p><p><b>4. Khả năng phát triển thành Copypasta</b><br>'+growth+'</p><p><b>Đánh giá thận trọng:</b> Một câu chỉ trở thành Copypasta theo cách hiểu của đề tài khi có dấu hiệu sao chép/lặp lại máy móc; tiếng lóng hoặc meme đơn lẻ chưa đủ căn cứ.</p><p><i>AI phân tích có thể mắc sai sót.</i></p>';out.classList.remove('hidden')}
const mazeMap=[[1,1,1,1,1,1,1],[1,0,0,0,1,0,1],[1,0,1,0,1,0,1],[1,0,1,0,0,0,1],[1,0,1,1,1,0,1],[1,0,0,0,0,2,1],[1,1,1,1,1,1,1]];
function renderMaze(){let f=$('feature');f.classList.remove('hidden');f.innerHTML='<h2>🧩 Mê cung ngôn từ</h2><p>Trái tim: <b id="heart">❤️❤️❤️</b> · Trả lời câu hỏi để mở quyền di chuyển. Sai sẽ mất một trái tim và được giải thích vì sao.</p><div id="mazeGrid" class="maze"></div><div id="qbox" class="msg"></div><div class="actions"><button onclick="moveMaze(0,-1)">⬆</button><button onclick="moveMaze(-1,0)">⬅</button><button onclick="moveMaze(1,0)">➡</button><button onclick="moveMaze(0,1)">⬇</button></div><div id="mazeMsg" class="msg">Trả lời câu hỏi để di chuyển.</div><button onclick="restartMaze()">Chơi lại</button> <button onclick="hideFeature()">Quay lại trang chủ</button>';restartMaze()}
function restartMaze(){mx=1;my=1;hearts=3;steps=0;mazeIndex=0;mazeCanMove=false;mazeWon=false;drawMap();question();$('heart').textContent='❤️❤️❤️';$('mazeMsg').textContent='Bắt đầu từ ô xuất phát. Hãy trả lời câu hỏi.'}
function drawMap(){let g=$('mazeGrid');if(!g)return;g.innerHTML='';mazeMap.forEach((row,y)=>row.forEach((v,x)=>{let d=document.createElement('div');d.className='tile '+(v===1?'wall':'')+(x===mx&&y===my?' player':'')+(v===2?' exit':'');d.textContent=x===mx&&y===my?'🙂':v===2?'🚪':'';g.appendChild(d)}))}
function question(){let q=mazeQuestions[mazeIndex%mazeQuestions.length],box=$('qbox');box.innerHTML='<b>Câu '+(mazeIndex+1)+': '+q.q+'</b><div class="answers">'+q.a.map((a,i)=>`<button onclick="answerMaze(${i})">${String.fromCharCode(65+i)}. ${a}</button>`).join('')+'</div>';mazeCanMove=false}
function answerMaze(i){if(mazeCanMove||hearts<=0||mazeWon)return;let q=mazeQuestions[mazeIndex%mazeQuestions.length],buttons=document.querySelectorAll('.answers button');buttons.forEach((b,j)=>{b.disabled=true;if(j===q.c){b.style.background='#2e8b70';b.innerHTML='✓ ĐÁP ÁN ĐÚNG: '+String.fromCharCode(65+j)+'. '+q.a[j]}else if(j===i){b.style.background='#9f405e';b.innerHTML='✗ BẠN CHỌN SAI: '+String.fromCharCode(65+j)+'. '+q.a[j]}});if(i===q.c){$('mazeMsg').innerHTML='<b>✓ Chính xác!</b><br>Vì sao đúng: '+q.why+'<br>Chọn một hướng để di chuyển qua lối đi.';mazeCanMove=true}else{hearts--; $('heart').textContent='❤️'.repeat(hearts)+'♡'.repeat(3-hearts);$('mazeMsg').innerHTML='<b>Chưa chính xác.</b><br>Vì sao lựa chọn này sai: '+q.a[i]+' chưa phản ánh đúng dấu hiệu cần xem xét. <br><b>Đáp án đúng:</b> '+q.a[q.c]+'.<br><b>Vì sao đáp án đúng:</b> '+q.why+(hearts===0?'<br><b>Hết lượt chơi.</b> Nhấn “Chơi lại” để bắt đầu lại.':'<br>Hãy đọc phần giải thích rồi trả lời câu tiếp theo.');if(hearts===0)return} }
function moveMaze(dx,dy){if(hearts<=0||mazeWon)return;if(!mazeCanMove){$('mazeMsg').textContent='Hãy trả lời đúng câu hỏi trước khi di chuyển.';return}let nx=mx+dx,ny=my+dy;if(mazeMap[ny]?.[nx]===undefined||mazeMap[ny][nx]===1){$('mazeMsg').textContent='Ô đó là tường. Vì sao chưa đi được? Bức tường chặn lối này; hãy chọn hướng khác trong mê cung.';return}mx=nx;my=ny;mazeCanMove=false;drawMap();if(mazeMap[my][mx]===2){mazeWon=true;$('mazeMsg').innerHTML='🎉 <b>Chúc mừng! Bạn đã tìm được cửa ra!</b><br>Bạn đã hoàn thành mê cung bằng cách vận dụng các dấu hiệu Copypasta và xem xét ngữ cảnh.';$('qbox').textContent='Mê cung hoàn thành.';return}mazeIndex++;question();$('mazeMsg').textContent='Đã đi sang ô mới. Hãy trả lời câu hỏi tiếp theo để tiếp tục.'}
function toggleDay(i,on){checkedDays=on?[...new Set([...checkedDays,i])]:checkedDays.filter(x=>x!==i);showFeature('commit')}
</script></body></html>"""

class MobileHandler(BaseHTTPRequestHandler):
    def log_message(self, *args):
        pass
    def _send(self, code, payload, content_type="application/json; charset=utf-8"):
        data = payload.encode("utf-8") if isinstance(payload, str) else payload
        self.send_response(code); self.send_header("Content-Type", content_type); self.send_header("Content-Length", str(len(data))); self.send_header("Cache-Control", "no-store"); self.end_headers(); self.wfile.write(data)
    def do_GET(self):
        if urlparse(self.path).path in ("/", "/index.html"):
            self._send(200, MOBILE_HTML, "text/html; charset=utf-8")
        else: self._send(404, json.dumps({"ok":False,"message":"Not found"}))
    def do_POST(self):
        if urlparse(self.path).path != "/auth": return self._send(404, {"ok":False,"message":"Not found"})
        try:
            data=json.loads(self.rfile.read(int(self.headers.get("Content-Length", "0"))).decode("utf-8"))
            username=str(data.get("username", "")).strip(); password=str(data.get("password", "")); mode=data.get("mode")
            if not username or not password: return self._send(400, json.dumps({"ok":False,"message":"Vui lòng nhập đủ thông tin."},ensure_ascii=False))
            try:
                with open(MOBILE_DB,"r",encoding="utf-8") as f: accounts=json.load(f)
            except (OSError, ValueError): accounts={}
            key=username.casefold()
            if mode == "register":
                if key in accounts: result={"ok":False,"message":"Tên đăng nhập đã tồn tại."}
                else:
                    accounts[key]={"username":username,"password_hash":hashlib.sha256(password.encode("utf-8")).hexdigest(),"created_at":time.strftime("%Y-%m-%d %H:%M:%S")}
                    with open(MOBILE_DB,"w",encoding="utf-8") as f: json.dump(accounts,f,ensure_ascii=False,indent=2)
                    result={"ok":True,"message":"Đăng ký thành công."}
            else:
                account=accounts.get(key)
                if account and hmac.compare_digest(account.get("password_hash", ""), hashlib.sha256(password.encode("utf-8")).hexdigest()): result={"ok":True,"message":"Đăng nhập thành công."}
                else: result={"ok":False,"message":"Tên đăng nhập hoặc mật khẩu chưa đúng trên máy chủ này."}
            self._send(200, json.dumps(result,ensure_ascii=False))
        except Exception:
            self._send(500, json.dumps({"ok":False,"message":"Có lỗi khi xử lý tài khoản."},ensure_ascii=False))

def run_mobile_web():
    server = ThreadingHTTPServer(("0.0.0.0", MOBILE_PORT), MobileHandler)
    url = "http://127.0.0.1:%s" % MOBILE_PORT
    print("Tram Loc Thong Tin dang chay.")
    print("Mo tren dien thoai cung Wi-Fi: http://DIA-CHI-IP-LAPTOP:%s" % MOBILE_PORT)
    print("Tren cung thiet bi: %s" % url)
    print("Nhan Ctrl+C de dung may chu.")
    try: webbrowser.open(url)
    except Exception: pass
    server.serve_forever()


if __name__ == "__main__":
    if "--mobile" in sys.argv or not TK_AVAILABLE:
        run_mobile_web()
    else:
        root = tk.Tk()
        tk.messagebox = __import__("tkinter.messagebox", fromlist=["showinfo"])
        tk.simpledialog = __import__("tkinter.simpledialog", fromlist=["askstring"])
        account_system = AccountSystem(root)
        root.mainloop()

