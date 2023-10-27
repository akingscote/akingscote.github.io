---
title: "Updated - Football Rainmeter Fixtures"
date: "2018-08-25"
categories: 
  - "miscellaneous"
coverImage: "fixed_skin.png"
---

With the new football season just started, I noticed that my Rainmeter widget to show the football fixtures had stopped working. Which breaks the method used in my [previous post](https://akingscote.co.uk/posts/rainmeter-football-skin/).

It turns out this is because the API that i was using for the data has changed from v1 to v2. This introduced a number of improvements to the API but unfortunately for me, it broke my rainmeter skin! You will now have to pass in an API key for the request for a fixture whereas previously this information was freely available. Its not a big deal though, because the rainmeter webparser plugin allows you to specify the header contents within the requests.

You will have to register and get an API key - http://www.football-data.org/client/register

The way i fixed the skin was to firstly figure out the new URL endpoint which i can use to get the data i want. I used fiddler and passed my API key into the header as

`X-Auth-Token: <api_key_here>`

Unfortunately the API filters dont seem to work properly, the dateFrom filter didnt work but the status filter did, so i simply filtered for matches which are scheduled.

http://api.football-data.org/v2/teams/67/matches?status=SCHEDULED

If you try this without the api key in the header, you wont see any data.

I then added the response into regexr and switched to Perl mode which is what rainmeter uses.

![](/images/regexr.png)

I parse the JSON response looking for the utcDate and seperating into groups on a dash. I then look for the two team names within the fixture and add them into groups. This gives me the information i need for a single upcoming fixture.

```
(?siU)utcDate":"(.*)-(.*)-(.*)T(.*)name":"(.*)"}(.*)name":"(.*)"}
```

I then (lazily) repeat the Regex to get a second team, i literally just copy the existing expression and add it to the end of itself.

```(?siU)utcDate":"(.*)-(.*)-(.*)T(.*)name":"(.*)"}(.*)name":"(.*)"}(.*)utcDate":"(.*)-(.*)-(.*)T(.*)name":"(.*)"}(.*)name":"(.*)"}```

Fortunately i didnt have to change the Rainmeter fixtures.ini much at all. All the measure groups and plugins stay exactly the same - the group numbers havent changed. All that is needed is the regular expression to be updated and the API key defined in the webparser plugin config.

```
[getData]
Measure=Plugin
Plugin=WebParser
Header=X-Auth-Token: <api_key_here>
URL=http://api.football-data.org/v2/teams/67/matches?status=SCHEDULED
RegExp=(?siU)utcDate":"(.*)-(.*)-(.*)T(.*)name":"(.*)"}(.*)name":"(.*)"}(.*)utcDate":"(.*)-(.*)-(.*)T(.*)name":"(.*)"}(.*)name":"(.*)"}
```

And just like that, the rainmeter skin is back to its former glory.
![](/images/fixed_skin.png)
