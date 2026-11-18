"""
DET ANDAS - ett skräckspel i Python
====================================
VARNING: spelet innehåller plötsliga jumpscares, blixtrande bilder och
höga ljud. Sänk volymen och spela inte om du är känslig för sådant.

Du vaknar i ett övergivet hus. Bara ficklampan lyser upp mörkret.
Hitta de 5 dagboksidorna så öppnas ytterdörren längst ner till höger.
Något i huset andas, och det hör och ser dig när du lyser.

Styrning:  WASD eller piltangenter = gå
           Musen = rikta ficklampan
           Vänsterklick eller F = ficklampa av/på
           M = ljud av/på
           ENTER = starta / försök igen,  ESC = avsluta

Tips: batteriet laddas när ficklampan är släckt. Släckt lampa gör att
monstret märker dig först när det är riktigt nära.

Kräver bara Python 3 (tkinter ingår). Kör:  python andas.py
Skrikljud hörs på Windows (winsound). På andra system blir det bara en pip.
"""

import math
import random
import threading
import time
import tkinter as tk
from collections import deque

try:
    import winsound
except ImportError:  # Mac / Linux
    winsound = None

# ---------------------------------------------------------------- kartan
MAP = [
    "#########################",
    "#.......#.......#.......#",
    "#...............#.......#",
    "#.......#...............#",
    "#.......#.......#.......#",
    "####.#######.#######.####",
    "#.......................#",
    "#.......................#",
    "######.#####.#####.######",
    "#.......#.......#.......#",
    "#.......#.......#.......#",
    "#...............#.......#",
    "#.......#.......#.......#",
    "#.......#...............#",
    "#.......#.......#.......#",
    "#.......#.......#.......#",
    "#####################E###",
]
ROWS, COLS = len(MAP), len(MAP[0])
ROOMS = [(1, 1, 7, 4), (9, 1, 15, 4), (17, 1, 23, 4),
         (1, 9, 7, 15), (9, 9, 15, 15), (17, 9, 23, 15)]

# ---------------------------------------------------------------- inställningar
TILE = 36
W, H = COLS * TILE, ROWS * TILE
HUD_H = 40
SUB = 2
CELL = TILE // SUB

BG = (4, 4, 6)
FLOOR_A = (58, 50, 45)
FLOOR_B = (50, 43, 39)
WALL = (104, 86, 74)
PAPER = (236, 230, 205)
BLOOD = (120, 14, 14)
RED = (220, 30, 40)
WHITE = (235, 235, 235)
DOOR_CLOSED = (110, 28, 28)
DOOR_OPEN = (60, 170, 90)

PLAYER_SPEED = 125
PLAYER_R = 7
LIGHT_RANGE = 8 * TILE
CONE_IN = 0.42
CONE_OUT = 0.78
AMBIENT_R = 52
AMBIENT = 0.20
TOTAL_PAGES = 5

WHISPERS = ["SE BAKOM DIG", "DU ÄR INTE ENSAM", "SPRING INTE", "JAG SER DIG",
            "HÄR INNE", "DEN ANDAS", "DU HÖRDE MIG"]


# ---------------------------------------------------------------- hjälpfunktioner
def hexc(c):
    return "#%02x%02x%02x" % tuple(int(v) for v in c)


def mix(c1, c2, t):
    t = max(0.0, min(1.0, t))
    return hexc(tuple(a + (b - a) * t for a, b in zip(c1, c2)))


def center(tile):
    return (tile[0] + 0.5) * TILE, (tile[1] + 0.5) * TILE


def bfs_step(grid, start, goal):
    """Nästa ruta på kortaste vägen från start till goal (eller None)."""
    if start == goal:
        return None
    prev = {start: None}
    q = deque([start])
    while q:
        cur = q.popleft()
        if cur == goal:
            break
        cx, cy = cur
        for dx, dy in ((1, 0), (-1, 0), (0, 1), (0, -1)):
            nx, ny = cx + dx, cy + dy
            if 0 <= nx < COLS and 0 <= ny < ROWS and (nx, ny) not in prev \
                    and grid[ny][nx] == 0:
                prev[(nx, ny)] = cur
                q.append((nx, ny))
    if goal not in prev:
        return None
    node = goal
    while prev[node] != start:
        node = prev[node]
    return node


# ---------------------------------------------------------------- spelet
class Game:
    def __init__(self, root):
        self.root = root
        root.title("DET ANDAS")
        root.resizable(False, False)
        self.canvas = tk.Canvas(root, width=W, height=H + HUD_H,
                                bg=hexc(BG), highlightthickness=0, cursor="crosshair")
        self.canvas.pack()
        self.keys = set()
        self.mx, self.my = W, H // 2
        root.bind("<KeyPress>", self.on_press)
        root.bind("<KeyRelease>", self.on_release)
        self.canvas.bind("<Motion>", self.on_motion)
        self.canvas.bind("<Button-1>", self.on_click)

        self.state = "title"          # title / play / dead / win
        self.sound = True
        self.beep_busy = False
        self.t = 0.0
        self.last = time.time()
        self.setup()
        self.tick()

    # ------------------------------------------------------------ start
    def setup(self):
        self.grid = [[1 if ch in "#E" else 0 for ch in row] for row in MAP]
        self.door = next((x, y) for y, row in enumerate(MAP)
                         for x, ch in enumerate(row) if ch == "E")
        self.door_open = False
        self.floor = [(x, y) for y in range(ROWS) for x in range(COLS)
                      if self.grid[y][x] == 0]

        self.px, self.py = center((2, 6))
        self.face = 0.0

        self.pages = []
        for (x0, y0, x1, y1) in random.sample(ROOMS, TOTAL_PAGES):
            tile = (random.randint(x0, x1), random.randint(y0, y1))
            px, py = center(tile)
            self.pages.append({"x": px, "y": py})
        self.collected = 0

        mx, my = center((20, 12))
        self.monster = {"x": mx, "y": my, "wp": None, "target": None,
                        "mode": "patrol", "lost": 0.0}

        self.stains = []
        for tile in random.sample(self.floor, 16):
            cx, cy = center(tile)
            self.stains.append((cx + random.uniform(-10, 10),
                                cy + random.uniform(-10, 10),
                                random.uniform(4, 9)))

        hall = [(x, y) for x in range(8, 23) for y in (6, 7)]
        self.triggers = {t: False for t in random.sample(hall, 3)}

        self.battery = 100.0
        self.battery_dead = False
        self.light_on = True
        self.power = 1.0
        self.blackout = 0.0
        self.pending_scare = False
        self.next_blackout = random.uniform(20, 30)
        self.next_whisper = random.uniform(12, 22)
        self.whisper = None
        self.scare = None
        self.siren_t = 0.0
        self.beat_t = 0.0
        self.msg = None

    # ------------------------------------------------------------ input
    def on_press(self, e):
        k = e.keysym.lower()
        self.keys.add(k)
        if k == "escape":
            self.root.destroy()
        elif k == "f" and self.state == "play":
            self.light_on = not self.light_on
        elif k == "m":
            self.sound = not self.sound
        elif k == "return":
            if self.state == "title":
                self.state = "play"
            elif self.state in ("dead", "win"):
                self.setup()
                self.state = "play"

    def on_release(self, e):
        self.keys.discard(e.keysym.lower())

    def on_motion(self, e):
        self.mx, self.my = e.x, e.y

    def on_click(self, e):
        if self.state == "play":
            self.light_on = not self.light_on

    # ------------------------------------------------------------ ljud
    def _beep_seq(self, seq):
        self.beep_busy = True
        try:
            for f, ms in seq:
                winsound.Beep(f, ms)
        except Exception:
            pass
        self.beep_busy = False

    def play_scream(self, death=False):
        if not self.sound:
            return
        if winsound:
            n = 30 if death else 12
            seq = [(60, 150)] + [(random.randint(500, 2800), 35) for _ in range(n)]
            threading.Thread(target=self._beep_seq, args=(seq,), daemon=True).start()
        else:
            try:
                self.root.bell()
            except Exception:
                pass

    def heartbeat(self):
        if self.sound and winsound and not self.beep_busy:
            threading.Thread(target=self._beep_seq, args=([(55, 90)],), daemon=True).start()

    # ------------------------------------------------------------ jumpscare
    def start_scare(self, kind, dur, death=False):
        self.scare = {"kind": kind, "t": 0.0, "dur": dur, "death": death}
        self.play_scream(death)

    # ------------------------------------------------------------ ljus och sikt
    def is_lit(self):
        return self.light_on and not self.battery_dead and self.blackout <= 0

    def los(self, x0, y0, x1, y1):
        """Fri sikt mellan två punkter (målrutan själv räknas inte som hinder)."""
        dx, dy = x1 - x0, y1 - y0
        d = math.hypot(dx, dy)
        if d < 1:
            return True
        tgt = (int(x1 // TILE), int(y1 // TILE))
        for i in range(1, int(d // 6) + 1):
            t = i * 6 / d
            tx = int((x0 + dx * t) // TILE)
            ty = int((y0 + dy * t) // TILE)
            if (tx, ty) == tgt:
                continue
            if self.grid[ty][tx] == 1:
                return False
        return True

    def cell_light(self, cx, cy):
        dx, dy = cx - self.px, cy - self.py
        d = math.hypot(dx, dy)
        b = 0.0
        if d < AMBIENT_R:
            b = AMBIENT * (1 - d / AMBIENT_R)
        if d < LIGHT_RANGE and self.is_lit():
            if d < 14:
                edge = 1.0
            else:
                diff = abs((math.atan2(dy, dx) - self.face + math.pi) % (2 * math.pi) - math.pi)
                if diff <= CONE_IN:
                    edge = 1.0
                elif diff < CONE_OUT:
                    edge = (CONE_OUT - diff) / (CONE_OUT - CONE_IN)
                else:
                    edge = 0.0
            if edge > 0:
                v = edge * (1 - d / LIGHT_RANGE) ** 0.7
                if v > b and self.los(self.px, self.py, cx, cy):
                    b = v
        return b * self.power

    # ------------------------------------------------------------ fysik
    def blocked(self, x, y, r):
        for cx, cy in ((x - r, y - r), (x + r, y - r), (x - r, y + r), (x + r, y + r)):
            if self.grid[int(cy // TILE)][int(cx // TILE)] == 1:
                return True
        return False

    def alert_monster(self):
        m = self.monster
        m["mode"] = "chase"
        m["lost"] = 0.0
        m["target"] = (int(self.px // TILE), int(self.py // TILE))
        m["wp"] = None

    def say(self, text, t=2.5):
        self.msg = [text, t]

    # ------------------------------------------------------------ uppdatering
    def update(self, dt):
        if self.scare:
            s = self.scare
            s["t"] += dt
            if s["t"] >= s["dur"]:
                self.scare = None
                if s["death"]:
                    self.state = "dead"
            return

        k = self.keys
        dx = (("right" in k) or ("d" in k)) - (("left" in k) or ("a" in k))
        dy = (("down" in k) or ("s" in k)) - (("up" in k) or ("w" in k))
        if dx and dy:
            dx *= 0.7071
            dy *= 0.7071
        nx = self.px + dx * PLAYER_SPEED * dt
        if not self.blocked(nx, self.py, PLAYER_R):
            self.px = nx
        ny = self.py + dy * PLAYER_SPEED * dt
        if not self.blocked(self.px, ny, PLAYER_R):
            self.py = ny
        self.face = math.atan2(self.my - self.py, self.mx - self.px)

        # utgång
        if self.door_open and int(self.py // TILE) >= ROWS - 1:
            self.state = "win"
            return

        # batteri och flimmer
        if self.is_lit():
            self.battery = max(0.0, self.battery - 2.2 * dt)
            if self.battery <= 0:
                self.battery_dead = True
        else:
            self.battery = min(100.0, self.battery + 3.5 * dt)
        if self.battery_dead and self.battery >= 18:
            self.battery_dead = False
        self.power = 1.0
        if self.battery < 25:
            self.power = 0.5 + 0.5 * random.random()
        if random.random() < 0.004:
            self.power *= 0.3

        # strömavbrott
        prev = self.blackout
        self.blackout = max(0.0, self.blackout - dt)
        self.next_blackout -= dt
        if self.next_blackout <= 0 and self.blackout <= 0:
            self.blackout = 1.6
            self.next_blackout = random.uniform(22, 38)
            self.pending_scare = random.random() < 0.55
            self.alert_monster()
        if prev > 0 and self.blackout == 0 and self.pending_scare:
            self.pending_scare = False
            self.start_scare("figure", 0.55)
            return

        # viskningar
        self.next_whisper -= dt
        if self.next_whisper <= 0:
            self.whisper = {"text": random.choice(WHISPERS), "t": 1.4,
                            "x": random.randint(140, W - 140),
                            "y": random.randint(80, H - 80)}
            self.next_whisper = random.uniform(15, 30)
        if self.whisper:
            self.whisper["t"] -= dt
            if self.whisper["t"] <= 0:
                self.whisper = None
        if self.msg:
            self.msg[1] -= dt
            if self.msg[1] <= 0:
                self.msg = None

        # dolda skräckfällor i hallen
        ptile = (int(self.px // TILE), int(self.py // TILE))
        if ptile in self.triggers and not self.triggers[ptile]:
            self.triggers[ptile] = True
            self.start_scare(random.choice(["face", "figure", "eyes"]), 0.7)
            return

        # sidor
        for p in self.pages[:]:
            if math.hypot(p["x"] - self.px, p["y"] - self.py) < 16:
                self.pages.remove(p)
                self.collected += 1
                self.alert_monster()
                if self.collected == TOTAL_PAGES:
                    self.door_open = True
                    self.grid[self.door[1]][self.door[0]] = 0
                    self.say("DÖRREN ÄR ÖPPEN. SPRING!", 6)
                else:
                    self.say(f"Sida {self.collected} av {TOTAL_PAGES}", 2.5)
                if self.collected in (2, 4):
                    self.start_scare("face", 0.8)
                    return

        self.update_monster(dt)

    def update_monster(self, dt):
        m = self.monster
        dist = math.hypot(m["x"] - self.px, m["y"] - self.py)

        # larmet drar till sig monstret
        if self.door_open:
            self.siren_t += dt
            if self.siren_t > 1.5:
                self.siren_t = 0.0
                self.alert_monster()

        sense = (11 if self.is_lit() else 4.5) * TILE
        if (dist < sense and self.los(m["x"], m["y"], self.px, self.py)) or dist < 2.2 * TILE:
            m["mode"] = "chase"
            m["lost"] = 0.0
            m["target"] = (int(self.px // TILE), int(self.py // TILE))
        elif m["mode"] == "chase":
            m["lost"] += dt
            if m["lost"] > 4.0:
                m["mode"] = "patrol"
                m["target"] = None
                m["wp"] = None

        if m["mode"] == "chase":
            speed = min(80 + 8 * self.collected, 120) + (6 if self.door_open else 0)
        else:
            speed = 52

        if m["wp"] is None:
            cur = (int(m["x"] // TILE), int(m["y"] // TILE))
            if m["mode"] == "chase" and dist < sense:
                m["target"] = (int(self.px // TILE), int(self.py // TILE))
            if m["target"] is None or cur == m["target"]:
                m["target"] = random.choice(self.floor)
                m["mode"] = "patrol"
            m["wp"] = bfs_step(self.grid, cur, m["target"])
            if m["wp"] is None:
                m["target"] = None
                return

        wx, wy = center(m["wp"])
        ddx, ddy = wx - m["x"], wy - m["y"]
        d = math.hypot(ddx, ddy)
        step = speed * dt
        if d <= step:
            m["x"], m["y"] = wx, wy
            m["wp"] = None
        else:
            m["x"] += ddx / d * step
            m["y"] += ddy / d * step

        # hjärtslag när det är nära
        if m["mode"] == "chase" and dist < 7 * TILE:
            self.beat_t -= dt
            if self.beat_t <= 0:
                self.beat_t = 0.75
                self.heartbeat()

        # fångad
        if math.hypot(m["x"] - self.px, m["y"] - self.py) < 20:
            self.start_scare("face", 1.7, death=True)

    # ------------------------------------------------------------ huvudloop
    def tick(self):
        now = time.time()
        dt = min(now - self.last, 0.05)
        self.last = now
        self.t += dt
        if self.state == "play":
            self.update(dt)
        self.draw()
        self.root.after(16, self.tick)

    # ------------------------------------------------------------ ritning
    def draw(self):
        c = self.canvas
        c.delete("all")
        if self.state == "title":
            self.overlay("DET ANDAS",
                         ["VARNING: jumpscares, blixtrande bilder och höga ljud.",
                          "Sänk volymen först.",
                          "",
                          "Hitta 5 dagboksidor i det mörka huset och ta dig ut.",
                          "Något i mörkret andas. Det hör och ser din ficklampa.",
                          "",
                          "WASD = gå     Musen = sikta     Klick / F = ficklampa",
                          "M = ljud av/på",
                          "",
                          "ENTER för att börja"], RED)
            return
        if self.scare:
            self.draw_scare()
            return

        self.draw_world()
        self.draw_hud()
        if self.state == "dead":
            self.overlay("DU DOG", [f"Sidor: {self.collected} av {TOTAL_PAGES}", "",
                                    "ENTER för att försöka igen"], RED)
        elif self.state == "win":
            self.overlay("DU KOM UT", ["Du hörde det andas bakom dig hela vägen.", "",
                                       "ENTER för att spela igen"], DOOR_OPEN)

    def draw_world(self):
        c = self.canvas
        ptx, pty = int(self.px // TILE), int(self.py // TILE)
        r = int(LIGHT_RANGE // TILE) + 1
        for ty in range(max(0, pty - r), min(ROWS, pty + r + 1)):
            for tx in range(max(0, ptx - r), min(COLS, ptx + r + 1)):
                is_door = (tx, ty) == self.door
                if is_door:
                    base = DOOR_OPEN if self.door_open else DOOR_CLOSED
                elif self.grid[ty][tx] == 1:
                    base = WALL
                else:
                    base = FLOOR_A if (tx + ty) % 2 == 0 else FLOOR_B
                for sy in range(SUB):
                    for sx in range(SUB):
                        x0 = tx * TILE + sx * CELL
                        y0 = ty * TILE + sy * CELL
                        b = self.cell_light(x0 + CELL / 2, y0 + CELL / 2)
                        if is_door and self.door_open:
                            b = max(b, 0.6)
                        if b > 0.015:
                            c.create_rectangle(x0, y0, x0 + CELL, y0 + CELL,
                                               fill=mix(BG, base, b), outline="")

        for (sx, sy, rad) in self.stains:
            b = self.cell_light(sx, sy)
            if b > 0.05:
                c.create_oval(sx - rad, sy - rad * 0.7, sx + rad, sy + rad * 0.7,
                              fill=mix(BG, BLOOD, b), outline="")

        for p in self.pages:
            b = self.cell_light(p["x"], p["y"])
            if b > 0.06:
                col = mix(BG, PAPER, b * 1.3)
                c.create_rectangle(p["x"] - 7, p["y"] - 9, p["x"] + 7, p["y"] + 9,
                                   fill=col, outline="")
                for i in range(3):
                    yy = p["y"] - 5 + i * 5
                    c.create_line(p["x"] - 4, yy, p["x"] + 4, yy, fill=mix(BG, (60, 50, 40), b))

        self.draw_monster()

        # spelaren
        c.create_oval(self.px - 6, self.py - 6, self.px + 6, self.py + 6,
                      fill=hexc((170, 175, 185)), outline="")
        c.create_line(self.px, self.py,
                      self.px + math.cos(self.face) * 14, self.py + math.sin(self.face) * 14,
                      fill=hexc((230, 230, 240)), width=2)

        # rött sken när monstret är nära
        m = self.monster
        d = math.hypot(m["x"] - self.px, m["y"] - self.py)
        if d < 260:
            k = 1 - d / 260
            w = int(4 + 10 * k)
            c.create_rectangle(w / 2, w / 2, W - w / 2, H - w / 2,
                               outline=mix(BG, RED, k * 0.8), width=w)

        if self.whisper:
            w = self.whisper
            c.create_text(w["x"], w["y"], text=w["text"],
                          fill=mix(BG, RED, min(1.0, w["t"] / 0.8)),
                          font=("Consolas", 24, "bold"))

    def draw_monster(self):
        c = self.canvas
        m = self.monster
        x, y = m["x"], m["y"]
        b = self.cell_light(x, y)
        d = math.hypot(x - self.px, y - self.py)
        if b > 0.08:
            c.create_line(x - 9, y - 14, x - 22, y + 12, fill=hexc((14, 10, 12)), width=3)
            c.create_line(x + 9, y - 14, x + 22, y + 12, fill=hexc((14, 10, 12)), width=3)
            c.create_oval(x - 11, y - 26, x + 11, y + 14, fill=hexc((12, 9, 11)),
                          outline=mix(BG, (120, 110, 115), b))
            c.create_oval(x - 7, y - 36, x + 7, y - 18,
                          fill=mix(BG, (205, 200, 190), b), outline="")
        if d < 9 * TILE and (m["mode"] == "chase" or d < 5 * TILE) \
                and self.los(x, y, self.px, self.py):
            c.create_oval(x - 5, y - 29, x - 1, y - 25, fill=hexc(RED), outline="")
            c.create_oval(x + 1, y - 29, x + 5, y - 25, fill=hexc(RED), outline="")

    def draw_hud(self):
        c = self.canvas
        y0 = H
        c.create_rectangle(0, y0, W, H + HUD_H, fill=hexc((14, 12, 14)), outline="")
        c.create_text(14, y0 + HUD_H // 2, anchor="w", fill=hexc(WHITE),
                      font=("Consolas", 13), text=f"SIDOR {self.collected}/{TOTAL_PAGES}")
        bx = W - 230
        c.create_text(bx - 10, y0 + HUD_H // 2, anchor="e", fill=hexc(WHITE),
                      font=("Consolas", 11), text="BATTERI")
        c.create_rectangle(bx, y0 + 13, bx + 200, y0 + 27, outline=hexc((110, 110, 115)),
                           fill=hexc(BG))
        frac = self.battery / 100
        col = (200, 50, 50) if frac < 0.25 else (230, 220, 150)
        c.create_rectangle(bx + 1, y0 + 14, bx + 1 + 198 * frac, y0 + 26,
                           outline="", fill=hexc(col))
        if self.msg:
            c.create_text(W // 2, y0 + HUD_H // 2, fill=hexc(RED if self.door_open else WHITE),
                          font=("Consolas", 13, "bold"), text=self.msg[0])

    def overlay(self, title, lines, color=RED):
        c = self.canvas
        cx, cy = W // 2, H // 2
        c.create_rectangle(cx - 360, cy - 190, cx + 360, cy + 190,
                           fill=hexc(BG), outline=hexc(color), width=2)
        c.create_text(cx, cy - 140, text=title, fill=hexc(color),
                      font=("Consolas", 36, "bold"))
        c.create_text(cx, cy + 15, text="\n".join(lines), fill=hexc(WHITE),
                      font=("Consolas", 12), justify="center")

    # ------------------------------------------------------------ jumpscare-ritning
    def draw_scare(self):
        c = self.canvas
        s = self.scare
        t, k = s["t"], s["t"] / s["dur"]
        phase = int(t * 28) % 3
        bg = ((0, 0, 0), (70, 0, 0), (0, 0, 0))[phase]
        c.create_rectangle(0, 0, W, H + HUD_H, fill=hexc(bg), outline="")

        shake = 14 * (1 - 0.4 * k)
        ox = random.uniform(-shake, shake)
        oy = random.uniform(-shake, shake)
        cx, cy = W / 2 + ox, H / 2 + 20 + oy

        if s["kind"] == "face":
            self.draw_face(cx, cy, 0.9 + 0.6 * min(1.0, k * 1.5), t)
        elif s["kind"] == "figure":
            self.draw_figure(cx, cy, 0.9 + 0.5 * min(1.0, k * 1.5))
        else:
            self.draw_eyes()

        if t < 0.06:
            c.create_rectangle(0, 0, W, H + HUD_H, fill=hexc((245, 245, 245)), outline="")

    def draw_face(self, cx, cy, s, t):
        c = self.canvas

        def P(x, y):
            return cx + x * s, cy + y * s

        jaw = 22 * math.sin(t * 38) + 22
        c.create_oval(*P(-170, -215), *P(170, 215), fill="#cfc8b8", outline="#000000", width=3)
        for _ in range(6):  # ådror och sprickor
            ex = random.choice((-75, 75))
            c.create_line(*P(ex, -55), *P(ex + random.randint(-120, 120), random.randint(-200, 40)),
                          fill="#6b0000", width=2)
        c.create_oval(*P(-125, -115), *P(-25, 10), fill="#000000", outline="")
        c.create_oval(*P(25, -115), *P(125, 10), fill="#000000", outline="")
        for ex in (-75, 75):
            jx, jy = random.uniform(-6, 6), random.uniform(-6, 6)
            c.create_oval(*P(ex - 9 + jx, -55 + jy), *P(ex + 9 + jx, -37 + jy),
                          fill="#ffffff", outline="")
        c.create_line(*P(-145, -135), *P(-30, -98), fill="#000000", width=int(8 * s))
        c.create_line(*P(145, -135), *P(30, -98), fill="#000000", width=int(8 * s))
        c.create_oval(*P(-22, 20), *P(-6, 46), fill="#2a2420", outline="")
        c.create_oval(*P(6, 20), *P(22, 46), fill="#2a2420", outline="")
        c.create_oval(*P(-108, 62), *P(108, 195 + jaw), fill="#0a0000", outline="#000000")
        for i in range(9):
            x0 = -95 + i * 21
            c.create_polygon(*P(x0, 88), *P(x0 + 21, 88), *P(x0 + 10, 128),
                             fill="#e8e2cf", outline="")
            c.create_polygon(*P(x0, 170 + jaw * 0.9), *P(x0 + 21, 170 + jaw * 0.9),
                             *P(x0 + 10, 132 + jaw * 0.9), fill="#e8e2cf", outline="")

    def draw_figure(self, cx, cy, s):
        c = self.canvas

        def P(x, y):
            return cx + x * s, cy + y * s

        c.create_polygon(*P(-70, 300), *P(-45, -60), *P(45, -60), *P(70, 300),
                         fill="#020202", outline="")
        c.create_oval(*P(-55, -230), *P(55, -40), fill="#000000", outline="#1a1a1a", width=2)
        c.create_line(*P(-45, -60), *P(-230, 220), fill="#000000", width=int(14 * s))
        c.create_line(*P(45, -60), *P(230, 220), fill="#000000", width=int(14 * s))
        for ex in (-22, 22):
            c.create_oval(*P(ex - 11, -140), *P(ex + 11, -108), fill="#ffffff", outline="")

    def draw_eyes(self):
        c = self.canvas
        for _ in range(45):
            x = random.randint(20, W - 20)
            y = random.randint(20, H + HUD_H - 20)
            r = random.randint(5, 16)
            for dx in (-r, r):
                c.create_oval(x + dx - r * 0.7, y - r * 0.5, x + dx + r * 0.7, y + r * 0.5,
                              fill="#f0f0f0", outline="")
                c.create_oval(x + dx - r * 0.25, y - r * 0.25, x + dx + r * 0.25, y + r * 0.25,
                              fill="#000000", outline="")


def main():
    root = tk.Tk()
    Game(root)
    root.mainloop()


if __name__ == "__main__":
    main()
