---
layout: post
title: "LoL Monitor: helping players manage their gaming sessions"
date: 2025-04-12
description: "Do you want to \"three block\" your League of Legends gaming sessions, but struggle to maintain self control? This tool helps you by automatically closing the lobby for you."
img: lolmonitor/lolmonitor_wide_azure.jpg
tags: [Project] # Personal, Opinion, Technical, Review, Project, Testing
main_page_summary: "Building a tool to close the League of Legends lobby after each game and enforce breaks"
---

## Executive Summary

In this post I give an overview of the **LoL Monitor** application that I built to help League of Legends (LoL) players manage their gaming sessions by closing the lobby after each game and enforcing breaks between games.

This includes:
* An insight into why LoL is so engaging and addictive, and why players can benefit from using LoL Monitor.
* A dive into the Go programming language that was used to build the tool, contrasting it with the more commonly used Python language.
* A perspective on software development in the generative AI era, how tools such as Copilot can help developers create products faster, and why their limitations require an ongoing role of human developers.

## Table of Contents

- [Introduction](#introduction)
- [The ten-dimensional chess board](#the-ten-dimensional-chess-board)
- [Go next](#go-next)
- [Refactoring the refactored refactor](#refactoring-the-refactored-refactor)
- [It's kibble time!](#its-kibble-time)
- [Conclusion](#conclusion)

## Introduction

[League of Legends](https://en.wikipedia.org/wiki/League_of_Legends) (LoL) has been around for over 15 years and is currently the world's largest esport, with millions of concurrent players around the world. 

The game is face-paced and deceptively complex, requiring significant time and effort to achieve mastery and progress in the ranked system. It combines the social and team based aspects of traditional sports with the complexity and strategy of a ten-dimensional chess board with constantly evolving rules. While it can be fun and engaging, it can also be emotionally overwhelming, and can lead players to fall into psychological traps that result in them playing longer than they would like to.

Let's start with an insight into the complexity and emotional aspects of the game, as motivation for the importance of developing a tool to help players have a more healthy relationship with the game.

## The ten-dimensional chess board

At face value, LoL seems like a fairly simple game. At opposing corners of the map are two bases, out of which come waves of minions who march down three lanes to defeat the opposing force. Ten champions - five on each team - act to swing the tide of the battle. In general, champions only need to press four buttons to use their active abilities, along with moving and attacking. Champions get gold and/or experience for killing minions and securing objectives that appear over the course of the game. They can use the gold to buy items, and use the experience to improve their abilites, making them stronger as the game progresses. The premise is simple and it's easy for players to get started.

There are, however, many layers of complexity that make the game so engaging and popular from an esport perspective:

* **Champions** - As of 23 January 2025, there are 170 champions in LoL, of which 10 are chosen each game. In theory, this means there are over 4 quadrillion possible combinations that could appear. In reality the number is lower, as champions tend to take on specific roles in the game and some champions are more commonly picked than others, but there's still a staggering number of possible drafts, making each game unique from the start. New champions are introduced regularly, constantly adding new interactions to the mix.

* **Abilities** - Each champion has four active abilities and one passive ability. With 170 champions and 5 abilities each, that's around 850 abilities in total. The individual abilities are generally quite unique and can contain a lot of details, and knowing these details can decide the outcome of a skirmish. For example, here's the [wiki](https://wiki.leagueoflegends.com/en-us/Briar) entry for a *single* ability (Briar's W):

![]({{site.baseurl}}/assets/img/lolmonitor/briar_w.png)

* **Items** - Throughout the course of the game, players spend gold on items. There are over 200 different items in the game, which can significantly change the strength of champions and their abilities. This adds an additional layer of complexity as the game progresses. There have also been many significant changes to the items over the years, adding further complexity over the longer term.

* **Intuition** - The best players in the world will not only have a general knowledge of all of these abilities, but will have a fairly good sense of the *cooldowns*, *range* and *damage* potential of their opponent's abilities, punishing cooldowns by playing more aggressively during a short window, tethering their oppoenents by standing just outside the range of their abilites, baiting them into using their abilities and so on. These values can change as the game progresses (in the example above, Briar's cooldown reduces as she puts more points into the ability, and the damage potential will depend on defensive stats on items purchased by her opponents). The skill ceiling on these aspects is super high.

* **Mental stack** - There are many things to focus on and think about at every instant of the game - from the constantly changing health of minions and the micro movements and ability usage of you and your lane opponent, to the macro movements and strategy of your team (and the enemy team if your team has vision of them) on the map. As humans we have an inherently limited ability to process information - our *mental stack* becomes overloaded and we fail to gather or use the information we need to make the right decisions. Managing our inherent limitations in such an environment (for example through intentional practice or building intuition over many games) can be a complex process.

* **Humans are random** - Champions are piloted by humans, who are emotional creatures. They may be dealing with real life issues, feeling tired, stressed or angry, and this makes their in-game behaviour inherently unpredictable. Managing this aspect can be complicated, particularly for the majority of players who are playing in solo queue and may find it challenging to communicate effectively with their anonymous team mates in such a fast-paced game.

All of this makes for an engaging environment which can induce strong emotions. Every game is packed with small victories which trigger dopamine hits. The randomness inherent in the process is akin to a slot machine. Games are relatively short (~30 minutes) and easy to join (matchmaking with a huge player base, no real physical exertion required), so it's easy to rationalise playing "just one more". 

Which brings us to the post game lobby window...

## Go next

There are two options coming out of a game of League of Legends, and both are bad news for playing in moderation.

Option 1: Victory! You're on a dopamine high, you're feeling great, and there's a button asking if you want some more. Of course I want more! Let's play another game!

![]({{site.baseurl}}/assets/img/lolmonitor/victory.png)

(If we pause for a moment, we can see that there is indeed a little X that would let us escape the infinite loop. It's hidden to the left of the giant glowing CONTINUE button.)

Option 2: Defeat! You played poorly and you're feeling shame, guilt, frustration. Or you played well, but you lost anyway; your teammates were underperforming, your team ignored your call and hung you out to dry, your team fought at a numbers disadvantage while you weren't there. It feels unfair and you're feeling frustrated. Either way, the easiest way to make these negative feelings go away is to sweep it under the rug and pretend it never happened. Next game you'll be better, or your team will perform better. Let's play another game!

![]({{site.baseurl}}/assets/img/lolmonitor/defeat_team.png)

In both cases, the main problem is that there's a big button that continues the endless cycle. It's like doom scrolling on social media or autoplay on YouTube. You need something to help you break the cycle. You need a moment so you can pause, reset and make a more logical decision rather than making an emotional one. You need that button to go away.

## Refactoring the refactored refactor

The premise of the idea is simple. If we can close the lobby window at the end of each game, the continue button will go away. Sure, we can reopen the game, but perhaps if we give ourselves that moment to make the logical decision, we'll make the right one, rather than the habitual one.

TODO: Go related stuff here, maybe split it out

## It's kibble time!

Given my personal interesting in the subject matter, I decided it was only appropriate I did a bit of [dogfooding](https://en.wikipedia.org/wiki/Eating_your_own_dog_food) before I wrote this blog post. This served to iron out a few wrinkles in the product and improve its design.

Before firing it up for the first time, I felt a mixure of fear and skepticism, which is funny given I'm the developer, but it's testament to the emotional side of playing LoL. I was worried it wouldn't work; that I would just override the tool and play another game and the whole thing would be a waste of time.

The first evening I went 0-3 (i.e. I lost every game) with an A rating or higher in every game, so there's a decent chance it was my team letting me down, which can be frustrating. What was surprising was how I felt when the tool kicked in and told me I had 10 seconds to look at the lobby and then had to take a break. I felt *relieved*. I was suddenly more aware of how I was feeling both physically and emotionally. After the second game, I was actually looking forward to finishing my third game and being kicked off - win or loss - because I knew it was an opportunity to go play a Cosy game or do a bit of coding. It was surprisingly effective.

TODO: update towards the end of the week

## Conclusion

Wrap up and reinforce the main takeaway (probably Thai), key points, next steps, call to action etc.[^1]

---
<small>Footnotes:</small>

[^1]: use the same symbol in the text to make the magic happen