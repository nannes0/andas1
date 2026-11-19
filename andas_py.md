import math, random, threading, time, tkinter as tk
from collections import deque
try:
    import winsound
except ImportError:
    winsound = None
MAP = ["#########################", "#.......#.......#.......#", "#...............#.......#",
       "#.......#...............#", "#.......#.......#.......#", "####.#######.#######.####",
       "#.......................#", "#.......................#", "######.#####.#####.######",
       "#.......#.......#.......#", "#.......#.......#.......#", "#...............#.......#",
       "#.......#.......#.......#", "#.......#...............#", "#.......#.......#.......#",
       "#.......#.......#.......#", "#####################E###"]

ROWS, COLS = len(MAP), len(MAP[0])
ROOMS = [(1, 1, 7, 4), (9, 1, 15, 4), (17, 1, 23, 4), (1, 9, 7, 15), (9, 9, 15, 15), (17, 9, 23, 15)]
TILE, HUD_H, SUB = 36, 40, 2
CELL = TILE // SUB
W, H = COLS * TILE, ROWS * TILE
BG, FLOOR_A, FLOOR_B, WALL = (4, 4, 6), (58, 50, 45), (50, 43, 39), (104, 86, 74)
PAPER, BLOOD, RED, WHITE = (236, 230, 205), (120, 14, 14), (220, 30, 40), (235, 235, 235)
DOOR_CLOSED, DOOR_OPEN = (110, 28, 28), (60, 170, 90)
SPEED, PR, RANGE = 125, 7, 8 * TILE
CONE_IN, CONE_OUT, AMB_R, AMB, PAGES = 0.42, 0.78, 52, 0.20, 5

hyp = math.hypot
def hexc(c):
    return "#%02x%02x%02x" % tuple(int(v) for v in c)
def mix(c1, c2, t):
    t = max(0.0, min(1.0, t))
    return hexc([a + (b - a) * t for a, b in zip(c1, c2)])
def center(t):
    return (t[0] + .5) * TILE, (t[1] + .5) * TILE
def tile(x, y):
    return int(x // TILE), int(y // TILE)
def bfs_step(grid, start, goal):
    if start == goal:
        return None
    prev, q = {start: None}, deque([start])
    while q and goal not in prev:
        cx, cy = q.popleft()
        for nx, ny in ((cx + 1, cy), (cx - 1, cy), (cx, cy + 1), (cx, cy - 1)):
            if 0 <= nx < COLS and 0 <= ny < ROWS and (nx, ny) not in prev and not grid[ny][nx]:
                prev[(nx, ny)] = (cx, cy)
                q.append((nx, ny))
    if goal not in prev:
        return None
    while prev[goal] != start:
        goal = prev[goal]
    return goal

class Game:
    def __init__(self, root):
        self.root = root
        root.title("DET ANDAS")
        root.resizable(False, False)
        self.c = tk.Canvas(root, width=W, height=H + HUD_H, bg=hexc(BG), highlightthickness=0, cursor="crosshair")
        self.c.pack()
        self.keys, self.mx, self.my = set(), W, H // 2
        root.bind("<KeyPress>", self.on_press)
        root.bind("<KeyRelease>", lambda e: self.keys.discard(e.keysym.lower()))
        self.c.bind("<Motion>", lambda e: setattr(self, "mx", e.x) or setattr(self, "my", e.y))
        self.c.bind("<Button-1>", lambda e: self.toggle_light())
        self.sound, self.beeping, self.last = True, False, time.time()
        self.setup()
        self.tick()
    def setup(self):
        self.state = "play"
        self.grid = [[ch in "#E" for ch in row] for row in MAP]
        self.door = (MAP[-1].index("E"), ROWS - 1)
        self.floor = [(x, y) for y in range(ROWS) for x in range(COLS) if not self.grid[y][x]]
        self.px, self.py = center((2, 6))
        self.face = 0.0
        self.pages = [center((random.randint(x0, x1), random.randint(y0, y1)))
                      for x0, y0, x1, y1 in random.sample(ROOMS, PAGES)]
        self.collected, self.door_open = 0, False
        mx, my = center((20, 12))
        self.mon = dict(x=mx, y=my, wp=None, target=None, mode="patrol", lost=0.0)
        self.stains = [(cx + random.uniform(-10, 10), cy + random.uniform(-10, 10), random.uniform(4, 9))
                       for cx, cy in map(center, random.sample(self.floor, 16))]
        self.battery, self.dead_bat, self.light_on, self.power = 100.0, False, True, 1.0
        self.blackout, self.next_blackout = 0.0, random.uniform(20, 30)
        self.siren_t = self.beat_t = 0.0
    def toggle_light(self):
        if self.state == "play":
            self.light_on = not self.light_on
    def on_press(self, e):
        k = e.keysym.lower()
        self.keys.add(k)
        if k == "escape":
            self.root.destroy()
        elif k == "f":
            self.toggle_light()
        elif k == "m":
            self.sound = not self.sound
        elif k == "return" and self.state != "play":
            self.setup()
    def beep(self, seq):
        def run():
            self.beeping = True
            try:
                for f, ms in seq:
                    winsound.Beep(f, ms)
            except Exception:
                pass
            self.beeping = False
        threading.Thread(target=run, daemon=True).start()
    def lit(self):
        return self.light_on and not self.dead_bat and self.blackout <= 0
    def los(self, x0, y0, x1, y1):
        dx, dy, d = x1 - x0, y1 - y0, hyp(x1 - x0, y1 - y0)
        tgt = tile(x1, y1)
        for i in range(1, int(d // 6) + 1):
            t = tile(x0 + dx * i * 6 / d, y0 + dy * i * 6 / d)
            if t != tgt and self.grid[t[1]][t[0]]:
                return False
        return True
    def light(self, cx, cy):
        dx, dy = cx - self.px, cy - self.py
        d = hyp(dx, dy)
        b = AMB * (1 - d / AMB_R) if d < AMB_R else 0.0
        if d < RANGE and self.lit():
            diff = abs((math.atan2(dy, dx) - self.face + math.pi) % (2 * math.pi) - math.pi)
            edge = 1.0 if d < 14 or diff <= CONE_IN else max(0.0, (CONE_OUT - diff) / (CONE_OUT - CONE_IN))
            v = edge * (1 - d / RANGE) ** 0.7
            if v > b and edge > 0 and self.los(self.px, self.py, cx, cy):
                b = v
        return b * self.power
    def blocked(self, x, y):
        return any(self.grid[int((y + oy) // TILE)][int((x + ox) // TILE)] for ox in (-PR, PR) for oy in (-PR, PR))
    def alert(self):
        self.mon.update(mode="chase", lost=0.0, target=tile(self.px, self.py), wp=None)
    def update(self, dt):
        k = self.keys
        dx = (("right" in k) or ("d" in k)) - (("left" in k) or ("a" in k))
        dy = (("down" in k) or ("s" in k)) - (("up" in k) or ("w" in k))
        if dx and dy:
            dx, dy = dx * .7071, dy * .7071
        if not self.blocked(self.px + dx * SPEED * dt, self.py):
            self.px += dx * SPEED * dt
        if not self.blocked(self.px, self.py + dy * SPEED * dt):
            self.py += dy * SPEED * dt
        self.face = math.atan2(self.my - self.py, self.mx - self.px)
        if self.door_open and tile(self.px, self.py)[1] >= ROWS - 1:
            self.state = "win"
            return
        if self.lit():
            self.battery = max(0.0, self.battery - 2.2 * dt)
            self.dead_bat = self.battery <= 0
        else:
            self.battery = min(100.0, self.battery + 3.5 * dt)
            if self.dead_bat and self.battery >= 18:
                self.dead_bat = False
        self.power = 0.5 + 0.5 * random.random() if self.battery < 25 else 1.0
        if random.random() < 0.004:
            self.power *= 0.3
        self.blackout = max(0.0, self.blackout - dt)
        self.next_blackout -= dt
        if self.next_blackout <= 0 and self.blackout <= 0:
            self.blackout, self.next_blackout = 1.6, random.uniform(22, 38)
            self.alert()
        for p in self.pages[:]:
            if hyp(p[0] - self.px, p[1] - self.py) < 16:
                self.pages.remove(p)
                self.collected += 1
                self.alert()
                if self.collected == PAGES:
                    self.door_open = True
                    self.grid[self.door[1]][self.door[0]] = False
        self.update_monster(dt)
    def update_monster(self, dt):
        m = self.mon
        dist = hyp(m["x"] - self.px, m["y"] - self.py)
        if self.door_open:
            self.siren_t += dt
            if self.siren_t > 1.5:
                self.siren_t = 0.0
                self.alert()
        sense = (11 if self.lit() else 4.5) * TILE
        if (dist < sense and self.los(m["x"], m["y"], self.px, self.py)) or dist < 2.2 * TILE:
            m.update(mode="chase", lost=0.0, target=tile(self.px, self.py))
        elif m["mode"] == "chase":
            m["lost"] += dt
            if m["lost"] > 4.0:
                m.update(mode="patrol", target=None, wp=None)
        chase = m["mode"] == "chase"
        speed = min(80 + 8 * self.collected, 120) + (6 if self.door_open else 0) if chase else 52
        if m["wp"] is None:
            cur = tile(m["x"], m["y"])
            if chase and dist < sense:
                m["target"] = tile(self.px, self.py)
            if m["target"] is None or cur == m["target"]:
                m["target"], m["mode"] = random.choice(self.floor), "patrol"
            m["wp"] = bfs_step(self.grid, cur, m["target"])
            if m["wp"] is None:
                m["target"] = None
                return
        wx, wy = center(m["wp"])
        ddx, ddy = wx - m["x"], wy - m["y"]
        d, step = hyp(ddx, ddy), speed * dt
        if d <= step:
            m["x"], m["y"], m["wp"] = wx, wy, None
        else:
            m["x"] += ddx / d * step
            m["y"] += ddy / d * step
        if chase and dist < 7 * TILE:
            self.beat_t -= dt
            if self.beat_t <= 0:
                self.beat_t = 0.75
                if self.sound and winsound and not self.beeping:
                    self.beep([(55, 90)])
        if hyp(m["x"] - self.px, m["y"] - self.py) < 20:
            self.state = "dead"
    def tick(self):
        now = time.time()
        dt, self.last = min(now - self.last, 0.05), now
        if self.state == "play":
            self.update(dt)
        self.draw()
        self.root.after(16, self.tick)
    def draw(self):
        c = self.c
        c.delete("all")
        if self.state == "dead":
            c.create_text(W // 2, (H + HUD_H) // 2, text="DU HAR FÖRLORAT", fill=hexc(RED), font=("Consolas", 36, "bold"))
            return
        self.draw_world()
        self.draw_hud()
        if self.state == "win":
            c.create_text(W // 2, H // 2, text="DU KOM UT", fill=hexc(DOOR_OPEN), font=("Consolas", 36, "bold"))
    def draw_world(self):
        c = self.c
        ptx, pty = tile(self.px, self.py)
        r = RANGE // TILE + 1
        for ty in range(max(0, pty - r), min(ROWS, pty + r + 1)):
            for tx in range(max(0, ptx - r), min(COLS, ptx + r + 1)):
                is_door = (tx, ty) == self.door
                base = ((DOOR_OPEN if self.door_open else DOOR_CLOSED) if is_door else
                        WALL if self.grid[ty][tx] else FLOOR_A if (tx + ty) % 2 == 0 else FLOOR_B)
                for sy in range(SUB):
                    for sx in range(SUB):
                        x0, y0 = tx * TILE + sx * CELL, ty * TILE + sy * CELL
                        b = self.light(x0 + CELL / 2, y0 + CELL / 2)
                        if is_door and self.door_open:
                            b = max(b, 0.6)
                        if b > 0.015:
                            c.create_rectangle(x0, y0, x0 + CELL, y0 + CELL, fill=mix(BG, base, b), outline="")
        for sx, sy, rad in self.stains:
            b = self.light(sx, sy)
            if b > 0.05:
                c.create_oval(sx - rad, sy - rad * .7, sx + rad, sy + rad * .7, fill=mix(BG, BLOOD, b), outline="")
        for x, y in self.pages:
            b = self.light(x, y)
            if b > 0.06:
                c.create_rectangle(x - 7, y - 9, x + 7, y + 9, fill=mix(BG, PAPER, b * 1.3), outline="")
                for i in range(3):
                    c.create_line(x - 4, y - 5 + i * 5, x + 4, y - 5 + i * 5, fill=mix(BG, (60, 50, 40), b))
        self.draw_monster()
        c.create_oval(self.px - 6, self.py - 6, self.px + 6, self.py + 6, fill=hexc((170, 175, 185)), outline="")
        c.create_line(self.px, self.py, self.px + math.cos(self.face) * 14, self.py + math.sin(self.face) * 14,
                      fill=hexc((230, 230, 240)), width=2)
        d = hyp(self.mon["x"] - self.px, self.mon["y"] - self.py)
        if d < 260:
            k = 1 - d / 260
            w = int(4 + 10 * k)
            c.create_rectangle(w / 2, w / 2, W - w / 2, H - w / 2, outline=mix(BG, RED, k * .8), width=w)
    def draw_monster(self):
        c, m = self.c, self.mon
        x, y = m["x"], m["y"]
        b, d = self.light(x, y), hyp(x - self.px, y - self.py)
        if b > 0.08:
            dark = hexc((14, 10, 12))
            c.create_line(x - 9, y - 14, x - 22, y + 12, fill=dark, width=3)
            c.create_line(x + 9, y - 14, x + 22, y + 12, fill=dark, width=3)
            c.create_oval(x - 11, y - 26, x + 11, y + 14, fill=hexc((12, 9, 11)), outline=mix(BG, (120, 110, 115), b))
            c.create_oval(x - 7, y - 36, x + 7, y - 18, fill=mix(BG, (205, 200, 190), b), outline="")
        if d < 9 * TILE and (m["mode"] == "chase" or d < 5 * TILE) and self.los(x, y, self.px, self.py):
            for ex in (-5, 1):
                c.create_oval(x + ex, y - 29, x + ex + 4, y - 25, fill=hexc(RED), outline="")
    def draw_hud(self):
        c, y0 = self.c, H
        c.create_rectangle(0, y0, W, H + HUD_H, fill=hexc((14, 12, 14)), outline="")
        for i in range(PAGES):
            c.create_rectangle(14 + i * 22, y0 + 12, 28 + i * 22, y0 + 28, outline=hexc((110, 110, 115)),
                               fill=hexc(PAPER) if i < self.collected else hexc(BG))
        bx = W - 214
        c.create_rectangle(bx, y0 + 13, bx + 200, y0 + 27, outline=hexc((110, 110, 115)), fill=hexc(BG))
        f = self.battery / 100
        c.create_rectangle(bx + 1, y0 + 14, bx + 1 + 198 * f, y0 + 26, outline="",
                           fill=hexc((200, 50, 50) if f < .25 else (230, 220, 150)))
        
if __name__ == "__main__":
    root = tk.Tk()
    Game(root)
    root.mainloop()
