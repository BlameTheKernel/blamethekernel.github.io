---
layout: post
title: "CheatEngine: Hacking games memory"
date: 2020-07-10 09:00:00 +0200
tags: [Memory, Windows, Games]
author: M
categories: Reversing Games
excerpt_separator: <!--more-->
---

Hi all! 

I like to reverse and crack stuff, and videogames is probably one of the most fun things to hack. Rules are okay but what would have happened if Neo followed the rules?? Exactly, he would have got a raise and would be happy with a wonderful fake life.

That's why we like to explain a bit how to start hacking some basic things in a videogame because breaking the rules can be fun also.

The game we choose to analyze and hack is a hilarious game made by [ChaosMonger Studio](http://www.chaosmonger.com/), a letonian company with 20 years of experience. Clunky Hero!!
<!--more-->

![the muscles]( /assets/games/clunkyHero.png)

The game consist in a simple farmer that one day going back to home after a rough day he discovers his ugly mean wife has been converted in a duck-face lady etcetcetc
How wouldn't you play this game to see what happens in the end?? look at that muscles and the shiny bucket!


### State of a game

When you want to hack a game first of all you need to play it, the more you play it the more you can analyze and see its behaviour from other perspective. Every stats means variables, so you must ask yourself some questions like "How is this stored? what triggers it? Is there an event that modifies it?"

For example, a shoot em up has a score that increments everytime you kill something like an enemy spaceship or an alien, so you ask, "can I change it to 1,000,000 points if I find the variable that stores it?", we will see it later. In Clunky Hero, we have other variables easy to spot, health and gold amount, so what if I our hero has 10 health points left but we don't have any potion? we can try to locate a variable called health and put 100 to heal him!

Is important to understand that the computer doesn't see the game as we are seeing it, with an UI and graphics. The computer only understand numbers and to recreate the state of the game each second to present it to you like you see the UI, the computer must search and modify variables like a crazy man inside a giant puppet. To find those variables in memory and modify them we have to use a memory manager that will search into the game's operating memory. 


## CheatEngine
A memory manager is probably the most important tool in a game hacker toolkit and we will use CheatEngine, one of the most famous ones. It's perfect to find the memory addresses of the game's data. Be aware that same values can appear in a lot of locations, so for each value we will have an index of registers that contains the number we are searching.
One method to know which is the one we are looking for is taking an initial value, for example we start with 9 gold after we kill the first enemy (a drunken bee O.o):

![Oro inicial]( /assets/games/GoldClunkyHero.png)

So we search for the integer 9 in the CheatEngine (imagine how many registers can have a 9 in it...)

![Oro inicial en memoria]( /assets/games/goldCheatEngine.png)

Now we avance a little and pick some more gold to look for this new value. After we kill our next enemy (a Big Head Armor Guy :O) we got 16 gold. 

![Oro pirata]( /assets/games/GoldClunkyHero2.png)

There come one of the most powerful features of CheatEngine, the increased value search. We go to Scan Type options and select "Increased value by..." and put the 7 gold increase from before and press "Next Scan" button, and these are the new results:

![Oro pirateado]( /assets/games/goldCheatEngine2.png)

We found less registers so we are near our desired register. After another drunken bee kill (O.o) we got 20 gold so we look in CheatEngine for another incremented value by 4 and we end with only 2 registers.

![Moar gold!]( /assets/games/goldCheatEngine3.png)

Now we need to manipulate these 2 registers to find the correct one adding them to our Cheat Table Panel and modify them by double-clicking in the value register. After changing both registers for a random number like 420, just because, and going to the UI game we know which one is the correct. So we found the memory address that stores the in-game Gold value of the current state! 

![yes...more]( /assets/games/GoldClunkyHero4.png)

Now let's try it with the Health of our weird but brave hero. After fighting a Big Head dude and two drunken bees (O.o) our Health Points are 40. We will scan the memory searching for '40' and then get hurt to low the HP and search again. With the help of an angry drunken bee we have our HP down to 30 and by using "Decrement value by..." we search for '10' and we find and unique register. Adding it to the Cheat Table Panel and modifying it with 1337 HP points will change the HP value of the game. 

![OVER9000]( /assets/games/HPClunkyHero2.png)

Now we can see our hero has now more life points than before and can talk with that poor bee a bit more until she calms down :)


Let's take an example in a shooting game where the final score is important because of competition around the world. The chosen one is a famous shmup, [DoDonPachi Resurrection](https://store.steampowered.com/app/464450/DoDonPachi_Resurrection/?l=spanish&curator_clanid=7589109). This game has been one of the most played in competitions in the shumps world so the final score board is important.

After killing some stuff and reaching the final boss of the first stage we can see different scores in the left panel, each one being a zone of the stage. We can't change the actual total score because is dynamic and constantly changing, so we select the first zone score, 63,189,536 points and search for it.

![low score]( /assets/games/DodonpachiScore.png)

We will change it for 64,000,000 points.

![score in memory]( /assets/games/DodonpachiRegister.png)

![cheating rules]( /assets/games/DodonpachiScore2.png)


But as you can see it doesn't change the total score, which we will try to change after killing the boss. After defeating the Girl-Convertible-Machine-Boss our final score is 80,656,123 points.
we search for it and we now find some registers that can store this value so we proceed to add them to our Cheat Table Panel and modify until we find the one that actually changes the number in the game.

![more points]( /assets/games/DodonpachiScore3.png)

![plus points]( /assets/games/DodonpachiRegister2.png)

We change our score for 90,000,000 and the game score reflects exactly that number! Number 1 World here we come!!


![WINNER WINNER CHICKEN DINNER]( /assets/games/DodonpachiScore4.png)


After some more calculations and dying at the beginning of the second stage we receive final points and go to register our name in the final scoreboard but...


![nooooo :(]( /assets/games/DodonpachiScore5.png)

whaaaaaaat? the final score that goes to the scoreboad is not the one we modified! Is the previous score (80,656,123) plus some extra points, so what happened?

We can't go deeper for legal reasons but it seems that there is some protection mechanism that keeps the score in a hidden register immutable so anyone can't cheat the global scoreboard. 
Another anti-reversing trick we found in this game is when you open the debugger of CheatEngine the client is closed immediately so these people went serious in this port to PC to maintain their competition untouchable.



### References

I always loved cheats in games and I forgot how much I like to hacks games until I found this great book by Nick Cano, [Game Hacking: Developing Autonomous Bots for Online Games](https://www.amazon.com/-/es/Nick-Cano/dp/1593276699). Seriously, if you like hacking games or you are starting this is a great beginning.

Also shouts out to [Guide Hacking Forum](https://guidedhacking.com/) a place with a lot of tutorials and great tips to learn hacking through games.

You can buy Clunky Hero by ChaosMonger in this [link](https://chaosmonger.itch.io/clunky-hero?download)! 


Thanks and don't forget, cheats makes life easier ;)
