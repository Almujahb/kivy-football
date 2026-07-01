from kivy.app import App
from kivy.uix.widget import Widget
from kivy.properties import NumericProperty, StringProperty
from kivy.clock import Clock
from kivy.core.window import Window
from kivy.core.audio import SoundLoader
from random import randint

Window.size = (900, 600)  # <- Force landscape window to match your field.png


class Ball(Widget):
    velocity_x = NumericProperty(0)
    velocity_y = NumericProperty(0)

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.size = (40, 40)  # Ball stays square

    def move(self):
        self.x += self.velocity_x
        self.y += self.velocity_y
        if self.y < 0 or self.y > Window.height - self.height:  # Bounce top/bottom
            self.velocity_y *= -1


class Player(Widget):  # Base class for all players
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.size = (60, 90)  # <- TALL instead of square. 60 wide, 90 tall

class Goalie(Player):  # Enemy goalie = Right side
    def __init__(self, start_x, **kwargs):
        super().__init__(**kwargs)
        self.size = (60, 90)  # <- TALL goalie too
        self.direction = 1
        self.locked_x = start_x  # Lock X to goal line
        self.x = start_x

        goal_height = 180  # Goal mouth height
        self.y_min = Window.height / 2 - goal_height / 2
        self.y_max = Window.height / 2 + goal_height / 2 - self.height
        self.y = Window.height / 2 - self.height / 2  # Start center

    def move(self):
        self.y += self.direction * 2  # Slower patrol
        if self.y < self.y_min or self.y > self.y_max:
            self.direction *= -1
        self.x = self.locked_x  # Force stay on goal line


class GoalieLeft(Goalie):  # Your goalie = Left side
    pass


class MyDefender(Player):  # Your team = Left half
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.speed_x = randint(-1, 1)  # <- SLOWER
        self.speed_y = randint(-1, 1)  # <- SLOWER

    def move(self):
        self.x += self.speed_x
        self.y += self.speed_y
        if self.y < 0 or self.y > Window.height - self.height:
            self.speed_y *= -1
        if self.x < 50 or self.x > Window.width / 2 - 20:  # Lock to left half
            self.speed_x *= -1


class EnemyDefender(Player):  # Enemy = Red, Right half
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.speed_x = randint(-1, 1)  # <- SLOWER
        self.speed_y = randint(-1, 1)  # <- SLOWER

    def move(self):
        self.x += self.speed_x
        self.y += self.speed_y
        if self.y < 0 or self.y > Window.height - self.height:
            self.speed_y *= -1
        if self.x < Window.width / 2 + 20 or self.x > Window.width - 50:  # Lock to right half
            self.speed_x *= -1


class Game(Widget):
    score = NumericProperty(0)
    score_text = StringProperty('Score: 0')

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.kick_sound = SoundLoader.load('kick.wav')
        self.save_sound = SoundLoader.load('save.wav')
        self.goal_sound = SoundLoader.load('goal.wav')

        self.ball = Ball(center_x=Window.width / 2, center_y=Window.height / 2)

        # 1. Measure the actual painted goal lines separately
        left_goal_line_x = 80  # <- LEAVE THIS. Blue is perfect
        right_goal_line_x = Window.width - 200  # <- CHANGE ONLY THIS for Red. Try 200, 220, 240

        goalie_into_box = 60  # How far inside the box to stand

        self.goalie_left = GoalieLeft(start_x=left_goal_line_x + goalie_into_box,
                                      center_y=Window.height / 2)  # You left
        self.goalie_right = Goalie(start_x=right_goal_line_x - goalie_into_box - 60, # <- -60 not -70 because width is 60 now
                                   center_y=Window.height / 2)  # Enemy right

        # Your team left half - SLOWER
        self.my_defenders = []
        for i in range(3):
            d = MyDefender(x=randint(80, Window.width / 2 - 80), y=randint(50, Window.height - 110))
            self.my_defenders.append(d)
            self.add_widget(d)

        # Enemy team right half - SLOWER
        self.enemy_defenders = []
        for i in range(3):
            d = EnemyDefender(x=randint(Window.width / 2 + 40, Window.width - 110), y=randint(50, Window.height - 110))
            self.enemy_defenders.append(d)
            self.add_widget(d)

        self.add_widget(self.ball)
        self.add_widget(self.goalie_left)
        self.add_widget(self.goalie_right)
        Clock.schedule_interval(self.update, 1.0 / 60.0)

    def on_touch_down(self, touch):
        if touch.x > Window.width / 2:  # Only shoot from right half
            dx = touch.x - self.ball.center_x
            dy = touch.y - self.ball.center_y
            mag = (dx ** 2 + dy ** 2) ** 0.5
            if mag != 0:
                self.ball.velocity_x = dx / mag * 10  # <- SLOWER SHOT
                self.ball.velocity_y = dy / mag * 10
                if self.kick_sound:
                    self.kick_sound.play()

    def update(self, dt):
        self.ball.move()
        self.goalie_left.move()
        self.goalie_right.move()
        for d in self.my_defenders + self.enemy_defenders:
            d.move()

        for d in self.my_defenders:  # Hit your defenders = -1 now
            if self.ball.collide_widget(d):
                self.ball.velocity_x *= -1
                self.ball.velocity_y *= -1
                self.score = max(0, self.score - 1)
                self.score_text = f'Score: {self.score}'

        if self.ball.collide_widget(self.goalie_left):  # You save
            self.ball.velocity_x *= -1
            if self.save_sound:
                self.save_sound.play()

        if self.ball.x < 0:  # You concede
            self.reset_ball()

        if self.ball.collide_widget(self.goalie_right):  # Enemy save
            self.ball.velocity_x *= -1
            self.score = max(0, self.score - 1)
            self.score_text = f'Score: {self.score}'
            if self.save_sound:
                self.save_sound.play()

        if self.ball.x > Window.width - self.ball.width:  # You score
            self.score += 5
            self.score_text = f'Score: {self.score}'
            if self.goal_sound:
                self.goal_sound.play()
            self.reset_ball()

    def reset_ball(self):
        self.ball.center = (Window.width / 2, Window.height / 2)
        self.ball.velocity_x = 0
        self.ball.velocity_y = 0


class FootballApp(App):
    def build(self):
        return Game()


if __name__ == '__main__':
    FootballApp().run()
