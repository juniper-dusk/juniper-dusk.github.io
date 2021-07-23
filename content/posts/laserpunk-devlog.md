---
title: "Laserpunk Devlog"
date: 2021-07-21T15:35:08-04:00
draft: false
summary: "A write-up on my experience solo developing an action platformer in two weeks."
description: "A write-up on my experience solo developing an action platformer in two weeks."
show_reading_time: true
toc: true
featured_image: '/images/laserpunk-logo.png'
category: "devlog"
---

![Game Demo GIF](/images/laserpunk-demo.gif)

## Theme

For [Level Up Circle: Beginner Jam](https://itch.io/jam/level-up-circle-beginner-jam-1), I embarked on my first solo game jam. I felt comfortable working in Godot and GDScript at this point, so it was a natural choice for this project.

The theme was "Temperature is key". A couple ideas for core game systems sprung to mind right away. These were ideas shared among some of the participants, but I figured I could put enough spin and polish into the mechanics to make my game stand out.

Before reading the devlog, I highly recommend [checking out the project for yourself](https://juniper-dusk.itch.io/laserpunk-demo)! The web build works on Chrome.

#### Initial concepts

- Balancing offense and defense in a combat setting, using overheating as a limit on attack power.
- Real-world concepts that require careful temperature management: cooking, hot air balloons, etc.
- Puzzles involving changing the state of matter (ice, water, steam) to make progress through the level.

#### Fleshing it out

The first idea attracted my attention as a fresh challenge, as I had not implemented a real-time combat system before. My [previous project](https://coffeebreakgamestudio.itch.io/courier-connect) involved movement as the primary focus. The main inspiration I had for controls was Katana Zero: tight platforming movement would still be key to survival, but attention would be split to offense as well. I also pulled some cues from my shmup project in enemy design.

Further reinforcing this idea would be the thermal management of the protagonist's weapons. Rather than trying to prioritize both offense and defence, the heat build-up and cooldown phases would create two separate paradigms of combat: damage and evasion.


----------------------------

## Art

On a somewhat limited time schedule, character design and sprite work happened simultaneously. Since the running theme of my process was unifying everything into a single core concept, I thought it would be cool to make the character one with the weapons and thermal systems by making them an android that uses their own energy in the weapons.

I started with a set of basic animations for the player character, following some invaluable tutorials by Pedro Medeiros ([@saint11](https://twitter.com/saint11)) regarding run cycles and jumps.

![Protagonist Animations](/images/protag-animated-display.gif)

Overall, I was pretty happy with the results, especially the movement on the cape. If I were to do the sprites again with more time and experience, I would just give them more in-between and smear frames. To save time I also used the [Zughy 32 palette](https://lospec.com/palette-list/zughy-32), but a more neon-heavy saturated palette might work better.

----------------------------

## Combat

#### Putting the laser in Laserpunk

In order to keep scope small, I designed only a single attack for the player. A laser beam was the perfect weapon to pair with the thermal mechanics. The beam could be fired for a certain amount before it raised the player's temperature too high.

If the temperature reached this threshold, a cool-down phase would start where the temperature returns back to normal levels. Here, the player could only move and dash, but not fire their weapon.

#### Easy to learn, hard to master

During playtesting, I realized that this cycle could get somewhat repetitive, and there wasn't much reward for the risk of running a high temperature. In order to reward skilled players, I designed a multiplier system: **the higher your temperature, the more damage your laser would do**. This would accompanied by visible feedback to the player with a change in color and width of the laser beam.

This served a secondary purpose as well: it made the difficulty curve scale slightly better when more enemies started spawning. At the start, you could coast just destroying one enemy at a time, but a minute in you had to use the extra damage to kill enemies quickly before things got out of hand.

Another form of depth I added was by making the dash cool-down. The move was less precise than simply moving, but offered some invincibility to make up for it. As a result, it encouraged more active movement from the player.

---------

## Platforming
##

Even though combat was important, from some early playtesting by myself and others I discovered that tight platforming was required to provide good player feedback and allow evasion of enemies and projectiles. This process of feedback and testing early on to iterate upon was something I maintained throughout development.

### State Machine

One of the first problems to solve while implementing a platformer is keeping the player from jumping while already in the air. It's possible to solve this naively by detecting when a character is colliding with the floor. However, a number of edge cases arise when you tack on other actions the character must perform. For instance, a dash shouldn't be interrupted by a jump. This can be solved with something called a state machine, which transitions between exclusive states, and prevents said interruption while in the "dash" state.

[Game Programming Patterns](http://gameprogrammingpatterns.com/state.html) has a very solid write-up on the subject, so I won't repeat too much of what is contained there. It's fairly trivial to get an object-oriented state machine up and running with nodes in Godot following this guide.

I have seen systems where states are defined as resources rather than inheriting from nodes, which makes more semantic sense in my opinion: the state doesn't need to exist in the game engine, it only needs to be accessed by the player node. When I port the state machine code to my next project I will improve that.

So there's a state machine for the player to move, jump, and dash separately. What about if you want to add a combat system that can be activated at any time? Simply add another state machine for it! I also planned to give enemies their own movement and combat state machines and a rudimentary AI system, but ran out of time. Instead, I made them static with simple, time-based projectiles instead.

### Polish

It turns out a lot of things go into good platforming feel that a state machine won't solve. Although it is pretty [common knowledge](https://www.youtube.com/watch?v=yorTG9at90g) to include these things, I'll list what I implemented.

#### Inertia

Humans and animals do not start walking/running and stop on a dime. There's a build up to get to full speed, and a slowdown to get to a stop (which either takes the place of the run -> walk -> stand, or a "skid" where friction takes over).

#### Hang-time

This one is not based in physical reality but rather for the sake of good control. When jumps follow a pre-determined arc like fighting games, this is unnecessary. In most platformers, however, a player can move in the air to reposition where they will land. The principle of inertia applies during the jump as well, with parameters being tweaked to give a character a more "floaty" or "heavy" feel.

#### Screen shake & freeze frames

Manipulating the camera and adding a delay to impactful actions gives the action more weight. Solely in terms of movement, I used this pretty heavily on the player's dash to give a sense of the power it takes to perform this action.

----------------------------

## Future Improvements

As the project was done in two weeks, there are plenty of areas that could be improved.

- Create more enemy varieties, bullet patterns, and grounded enemies that navigate the environment.
- Add levels and a story mode, as opposed to an endless arena.
- Give the player more attacks. Laser swords are sick!