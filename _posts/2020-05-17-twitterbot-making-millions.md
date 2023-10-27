---
title: "Twitterbot - Making Millions $$$$"
date: "2020-05-17"
categories: 
  - "development"
  - "play"
  - "python"
tags: 
  - "automation"
  - "development"
  - "millions"
  - "python"
  - "twitter"
coverImage: "football.png"
---

So three years ago, i made a little program and I've decided to finally do a little writeup on it.

Im a big football fan and i love to place a little bet. Every January and summer im constantly refreshing sky sports to see what players are going where. One summer, i noticed that you can bet on what team you think a player will transfer too. This sparked a little idea to i got to work realising it.

My idea was simple - beat the bookies. My application firstly went to William Hill and went onto the page where you can bet on a player to transfer to a specific team. It grabbed the HTML and parsed the players names and the options that the bookies would allow you to bet on. You cant bet on anything - only topical players and "likely" teams. You can exactly bet on Shearer to go to Bromley. Once i had a list of all the player names and their available options to bet on, i combined the player names with a set of key words which i had stored in a text file.

So lets say we had "rooney", i would combine the string "rooney" in combination with a number of my key words. For example, "rooney spotted", "rooney likely", "rooney seen", "rooney go" and so on. I would do this for every player and would then search the twitter stream for somebody tweating about this magic combination.

The thought process is that twitter is a such an amazing source of information. I thought that I could beat the bookies by getting information from first hand witnesses (fans, journalists, agents etc...). So if a fan spotted rooney at newcastle airport, there would be a good chance that rooney was on his way to sign for newcastle. This app was being ran during the transfer windows, so its unlikely that rooney would be in newcastle for a visit.

Now the bookies dont exactly have an API that allows you to place a bet. Why would they do that? People would just use it to undercut them. Betfair does have something but you have to paay a couple of hundred pounds to gain access to it. It wasnt worth it for my little project. So what i did was i used selenium (primarily an website testing automation tool) to fire up a programmable browser, navigate to the betting page and automatically log in, find the rooney and newcastle option on the page and place a 5p bet.

My first iteration of this app didnt go quite to plan. I put £10 credit into my account, left the app running over night and when i checked it in the morning i had £0 left in my account!. It turns out i had completely forgotten about retweets - derp!

I wont post the source code, it was just strung together and it has my twitter API key hardcoded. Plus its 3 years old so i doubt it works anymore.

However, here is a video of this bad boy in action: https://youtu.be/Qsc6rxFYdSE

In that video, i dont put any credit in my account but you can see that the actual process is entirely automated - i dont move my cursor once the bet is placed. Unfortunately this was 3 years ago that I made this, so i dont have any better footage. I ended up running it on a raspberry pi for a couple of weeks.

I think that there is some serious merit in this idea, i imagine that the bookies have something similar in place for deciding their odds. If they dont, they really should. I think that if i put some serious effort into this app it could actually work quite nicely. You could even narrow down the sources to be from reliable sources.

On a side note - i remember when i was testing this app i would personally tweet things like "rooney to newcastle" to test my API. as a result my twitter feed was just a wash of the same "rooney to newcastle" tweet (or subtle variations). I then got a notification that the CEO of the company i work at had followed me and i was just tweeting "rooney to newcastle" all the time - oops.

Obviously this wasnt a serious attempt to make money, it was just a bit of fun exploring an idea. I think i probably lost £20 in total, because it wasnt really fine tuned. The app would just bet on everything, so in the end i had bets for a player to go to nearly every combination. You cant make money if you bet on every possible option haha!
