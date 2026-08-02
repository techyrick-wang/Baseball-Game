# Baseball-Game
Use an RP2040. Controls are listed on the screen.









import machine
import test.st7789 as st7789
import time
import math
import random
from machine import Pin, ADC
from test.fonts import vga2_8x8 as font1
from test.fonts import vga1_16x32 as font2


# =========================
# SCREEN SETUP
# =========================

st7789_res = 0
st7789_dc = 1

disp_width = 240
disp_height = 240

spi_sck = machine.Pin(2)
spi_tx = machine.Pin(3)

spi0 = machine.SPI(
    0,
    baudrate=40000000,
    phase=1,
    polarity=1,
    sck=spi_sck,
    mosi=spi_tx
)

display = st7789.ST7789(
    spi0,
    disp_width,
    disp_height,
    reset=machine.Pin(st7789_res, machine.Pin.OUT),
    dc=machine.Pin(st7789_dc, machine.Pin.OUT),
    xstart=0,
    ystart=0,
    rotation=1
)


# =========================
# CONTROLS
# =========================

xAxis = ADC(Pin(28))
yAxis = ADC(Pin(29))

buttonB = Pin(5, Pin.IN, Pin.PULL_UP)
buttonA = Pin(6, Pin.IN, Pin.PULL_UP)


# =========================
# COLORS
# =========================

BLACK = 0x0000
WHITE = 0xFFFF
GREEN = 0x07E0
DARK_GREEN = 0x03E0
BLUE = 0x001F
RED = 0xF800
YELLOW = 0xFFE0
BROWN = 0xA145
PINK = 0xF81F
GRAY = 0x8410


# =========================
# HELPERS
# =========================

def a_pressed():
    return buttonA.value() == 0


def b_pressed():
    return buttonB.value() == 0


def joy_up():
    return xAxis.read_u16() < 20000


def joy_down():
    return xAxis.read_u16() > 45000


def wait_release():
    time.sleep(0.15)
    while a_pressed() or b_pressed():
        time.sleep(0.03)


def clamp(v, low, high):
    if v < low:
        return low
    if v > high:
        return high
    return v


def fill_circle(cx, cy, r, color):
    cx = int(cx)
    cy = int(cy)
    r = int(r)

    for yy in range(-r, r + 1):
        x_len = int(math.sqrt(r * r - yy * yy))
        display.fill_rect(cx - x_len, cy + yy, x_len * 2 + 1, 1, color)


def draw_circle(cx, cy, r, color):
    cx = int(cx)
    cy = int(cy)
    r = int(r)

    for yy in range(-r, r + 1):
        x_len = int(math.sqrt(r * r - yy * yy))
        display.fill_rect(cx - x_len, cy + yy, 1, 1, color)
        display.fill_rect(cx + x_len, cy + yy, 1, 1, color)


def center_text_small(text, y, color):
    x = 120 - (len(text) * 4)
    display.text(font1, text, x, y, color=color)


def center_text_big(text, y, color):
    x = 120 - (len(text) * 8)
    display.text(font2, text, x, y, color=color)


# =========================
# DRAWING
# =========================

def draw_start_screen():
    display.fill(BLACK)
    center_text_big("BASEBALL", 45, YELLOW)
    center_text_small("Joystick UP/DOWN", 105, WHITE)
    center_text_small("A = swing", 130, WHITE)
    center_text_small("B = restart", 155, WHITE)
    center_text_small("3 innings", 180, GREEN)
    center_text_small("Press A", 210, YELLOW)


def draw_game(score, inning, outs, strikes, bat_y, ball_x, ball_y, message):
    display.fill(GREEN)

    # scoreboard
    display.fill_rect(0, 0, 240, 40, BLACK)
    display.text(font1, "BASEBALL", 5, 5, color=YELLOW)
    display.text(font1, "RUNS:" + str(score), 5, 22, color=WHITE)
    display.text(font1, "INN:" + str(inning), 75, 22, color=WHITE)
    display.text(font1, "OUT:" + str(outs), 125, 22, color=WHITE)
    display.text(font1, "STR:" + str(strikes), 180, 22, color=WHITE)

    # field diamond
    display.fill_rect(103, 160, 34, 34, BROWN)
    display.fill_rect(116, 145, 12, 12, WHITE)
    display.fill_rect(116, 200, 12, 12, WHITE)
    display.fill_rect(82, 174, 12, 12, WHITE)
    display.fill_rect(146, 174, 12, 12, WHITE)

    # pitcher
    fill_circle(55, 120, 15, BROWN)
    fill_circle(55, 120, 5, WHITE)

    # strike zone
    display.rect(176, 80, 26, 68, WHITE)

    # batter
    fill_circle(214, bat_y - 22, 7, PINK)
    display.fill_rect(208, bat_y - 14, 12, 28, BLUE)
    display.fill_rect(198, bat_y - 4, 25, 5, BROWN)

    # bat
    display.fill_rect(188, bat_y - 3, 25, 4, YELLOW)

    # ball
    fill_circle(ball_x, ball_y, 4, WHITE)
    draw_circle(ball_x, ball_y, 4, BLACK)

    display.text(font1, "A swing", 5, 220, color=WHITE)
    display.text(font1, "B reset", 170, 220, color=WHITE)

    if message != "":
        center_text_small(message, 48, YELLOW)


def draw_game_over(score):
    display.fill(BLACK)
    center_text_big("GAME", 45, YELLOW)
    center_text_big("OVER", 85, YELLOW)
    center_text_small("Final runs: " + str(score), 140, WHITE)
    center_text_small("A restart", 180, GREEN)
    center_text_small("B restart", 205, WHITE)


# =========================
# GAME
# =========================

def wait_start():
    draw_start_screen()

    while True:
        if a_pressed():
            wait_release()
            return

        time.sleep(0.05)


def baseball_game():
    score = 0
    inning = 1
    outs = 0
    strikes = 0

    bat_y = 115

    ball_x = 45
    ball_y = random.randint(85, 145)
    ball_speed = random.randint(5, 8)

    message = "TIME IT!"

    game_over = False

    while True:
        if game_over:
            draw_game_over(score)

            if a_pressed() or b_pressed():
                wait_release()
                return

            time.sleep(0.05)
            continue

        if b_pressed():
            wait_release()
            return

        if joy_up():
            bat_y -= 4

        if joy_down():
            bat_y += 4

        bat_y = clamp(bat_y, 80, 160)

        ball_x += ball_speed

        if a_pressed():
            if 168 <= ball_x <= 205 and abs(ball_y - bat_y) < 26:
                hit = random.randint(1, 8)

                if hit <= 4:
                    score += 1
                    message = "SINGLE! +1"
                elif hit <= 6:
                    score += 2
                    message = "DOUBLE! +2"
                elif hit == 7:
                    score += 3
                    message = "TRIPLE! +3"
                else:
                    score += 4
                    message = "HOME RUN!"

                strikes = 0
            else:
                strikes += 1
                message = "MISS!"

                if strikes >= 3:
                    strikes = 0
                    outs += 1
                    message = "STRIKEOUT!"

            ball_x = 45
            ball_y = random.randint(85, 145)
            ball_speed = random.randint(5, 9)
            wait_release()

        if ball_x > 235:
            strikes += 1
            message = "STRIKE!"

            if strikes >= 3:
                strikes = 0
                outs += 1
                message = "OUT!"

            ball_x = 45
            ball_y = random.randint(85, 145)
            ball_speed = random.randint(5, 9)

        if outs >= 3:
            inning += 1
            outs = 0
            strikes = 0
            message = "NEXT INNING"

        if inning > 3:
            game_over = True

        draw_game(score, inning, outs, strikes, bat_y, ball_x, ball_y, message)

        time.sleep(0.03)


# =========================
# Start
# =========================

while True:
    wait_start()
    baseball_game()
