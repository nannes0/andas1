import pygame
import random
import math

# Starta pygame
pygame.init()

# Skapa ett fönster
WIDTH = 800
HEIGHT = 600
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("DET ANDAS")
clock = pygame.time.Clock()

# Färger
WHITE = (255, 255, 255)
RED = (220, 30, 40)
GREEN = (60, 170, 90)
FLOOR = (60, 52, 46)
WALL_COLOR = (110, 90, 75)
PAPER = (236, 230, 205)

# Spelaren
player = pygame.Rect(50, 300, 20, 20)
player_speed = 3

# Väggar (rektanglar som spelaren inte kan gå igenom)
walls = [
    pygame.Rect(200, 0, 20, 220),
    pygame.Rect(200, 320, 20, 280),
    pygame.Rect(400, 150, 20, 300),
    pygame.Rect(600, 0, 20, 300),
    pygame.Rect(600, 400, 20, 200),
]

# Dörren längst ner till höger, den öppnas när alla sidor är tagna
door = pygame.Rect(WIDTH - 80, HEIGHT - 20, 60, 20)
door_open = False

# Dagboksidor på slumpade platser (inte inne i en vägg)
pages = []
while len(pages) < 5:
    page = pygame.Rect(random.randint(20, WIDTH - 40), random.randint(20, HEIGHT - 60), 14, 18)
    if page.collidelist(walls) == -1 and not page.colliderect(player):
        pages.append(page)
pages_collected = 0

# Monstret (x och y är decimaltal så att det kan röra sig långsamt)
monster = pygame.Rect(700, 100, 30, 30)
monster_x = 700.0
monster_y = 100.0
target_x = 700
target_y = 100

# Ficklampa och batteri
lamp_on = True
battery = 100.0

# Är spelet slut? "" = nej, "win" = vunnit, "lose" = förlorat
game_over = ""

# Typsnitt
font = pygame.font.Font(None, 36)
big_font = pygame.font.Font(None, 80)

# Variabel för att styra när spelet körs
running = True

# Huvudloopen
while running:
    clock.tick(60)

    # Kontrollera händelser
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
        if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
            running = False

        # F eller musklick slår av/på ficklampan
        if (event.type == pygame.KEYDOWN and event.key == pygame.K_f) or event.type == pygame.MOUSEBUTTONDOWN:
            if lamp_on:
                lamp_on = False
            elif battery > 20:
                lamp_on = True

    # Allt nedan händer bara om spelet inte är slut
    if game_over == "":

        # Spelarens rörelse
        keys = pygame.key.get_pressed()
        dx = 0
        dy = 0
        if keys[pygame.K_a] or keys[pygame.K_LEFT]:
            dx = -player_speed
        if keys[pygame.K_d] or keys[pygame.K_RIGHT]:
            dx = player_speed
        if keys[pygame.K_w] or keys[pygame.K_UP]:
            dy = -player_speed
        if keys[pygame.K_s] or keys[pygame.K_DOWN]:
            dy = player_speed

        # Det spelaren inte kan gå igenom (dörren räknas bara när den är stängd)
        solid = walls.copy()
        if not door_open:
            solid.append(door)

        # Flytta i x-led, backa om vi krockar med något
        player.x += dx
        for wall in solid:
            if player.colliderect(wall):
                player.x -= dx

        # Flytta i y-led, backa om vi krockar med något
        player.y += dy
        for wall in solid:
            if player.colliderect(wall):
                player.y -= dy

        # Håll spelaren inne i fönstret
        player.clamp_ip(screen.get_rect())

        # Plocka upp sidor
        for page in pages.copy():
            if player.colliderect(page):
                pages.remove(page)
                pages_collected += 1

        # Alla sidor tagna: öppna dörren
        if pages_collected == 5:
            door_open = True

        # Batteriet minskar när lampan lyser och laddas när den är släckt
        if lamp_on:
            battery -= 0.04
            if battery <= 0:
                battery = 0
                lamp_on = False
        else:
            battery += 0.06
            if battery > 100:
                battery = 100

        # Monstret märker spelaren på kortare håll om lampan är släckt
        distance = math.hypot(player.centerx - monster_x, player.centery - monster_y)
        if lamp_on:
            notice_distance = 350
        else:
            notice_distance = 120

        if distance < notice_distance:
            # Jaga spelaren, snabbare för varje sida som tagits
            target_x = player.centerx
            target_y = player.centery
            monster_speed = 1.2 + 0.3 * pages_collected
        else:
            monster_speed = 1

        # Har monstret nått sitt mål? Välj då ett nytt slumpat mål
        if math.hypot(target_x - monster_x, target_y - monster_y) < 10:
            target_x = random.randint(0, WIDTH)
            target_y = random.randint(0, HEIGHT)

        # Flytta monstret mot målet (det kan gå genom väggar)
        move_x = target_x - monster_x
        move_y = target_y - monster_y
        length = math.hypot(move_x, move_y)
        if length > 0:
            monster_x += move_x / length * monster_speed
            monster_y += move_y / length * monster_speed
        monster.center = (monster_x, monster_y)

        # Förlust om monstret rör spelaren, vinst om spelaren når den öppna dörren
        if monster.colliderect(player):
            game_over = "lose"
        if door_open and player.colliderect(door):
            game_over = "win"

    # Fyller bakgrunden med golvfärg
    screen.fill(FLOOR)

    # Ritar väggar, dörr, sidor, monster och spelare
    for wall in walls:
        pygame.draw.rect(screen, WALL_COLOR, wall)

    if door_open:
        pygame.draw.rect(screen, GREEN, door)
    else:
        pygame.draw.rect(screen, (110, 28, 28), door)

    for page in pages:
        pygame.draw.rect(screen, PAPER, page)

    pygame.draw.rect(screen, (200, 195, 185), monster)
    pygame.draw.rect(screen, (170, 175, 185), player)

    # Mörkret: en svart yta över hela skärmen där vi klipper hål för ljuset
    dark = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
    dark.fill((0, 0, 0, 245))

    # Lite svagt ljus runt spelaren
    pygame.draw.circle(dark, (0, 0, 0, 150), player.center, 60)

    # Ficklampans ljuskon pekar mot musen (två trianglar: ytterkant och mitt)
    if lamp_on:
        mouse_x, mouse_y = pygame.mouse.get_pos()
        angle = math.atan2(mouse_y - player.centery, mouse_x - player.centerx)
        px, py = player.center

        outer1 = (px + math.cos(angle - 0.5) * 380, py + math.sin(angle - 0.5) * 380)
        outer2 = (px + math.cos(angle + 0.5) * 380, py + math.sin(angle + 0.5) * 380)
        pygame.draw.polygon(dark, (0, 0, 0, 110), [player.center, outer1, outer2])

        inner1 = (px + math.cos(angle - 0.3) * 330, py + math.sin(angle - 0.3) * 330)
        inner2 = (px + math.cos(angle + 0.3) * 330, py + math.sin(angle + 0.3) * 330)
        pygame.draw.polygon(dark, (0, 0, 0, 0), [player.center, inner1, inner2])

    screen.blit(dark, (0, 0))

    # Monstrets röda ögon syns i mörkret när det är nära
    if distance < 200:
        pygame.draw.rect(screen, RED, (monster.centerx - 8, monster.centery - 5, 4, 4))
        pygame.draw.rect(screen, RED, (monster.centerx + 4, monster.centery - 5, 4, 4))

    # Rita antal sidor och batteri
    pages_text = font.render("Sidor: " + str(pages_collected) + "/5", True, WHITE)
    screen.blit(pages_text, (10, 10))
    pygame.draw.rect(screen, WHITE, (WIDTH - 214, 10, 204, 20), 2)
    pygame.draw.rect(screen, (230, 220, 150), (WIDTH - 212, 12, 2 * battery, 16))

    # Visa text när spelet är slut
    if game_over == "lose":
        text = big_font.render("DU HAR FÖRLORAT", True, RED)
        screen.blit(text, (WIDTH // 2 - text.get_width() // 2, HEIGHT // 2))
    if game_over == "win":
        text = big_font.render("DU KOM UT", True, GREEN)
        screen.blit(text, (WIDTH // 2 - text.get_width() // 2, HEIGHT // 2))

    # Uppdaterar skärmen
    pygame.display.flip()

# Avslutar pygame
pygame.quit()
