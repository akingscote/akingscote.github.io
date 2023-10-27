---
title: "Rainmeter Football Skin"
date: "2017-01-25"
categories: 
  - "play"
  - "rainmeter"
---

# OUTDATED - see the [new post](https://akingscote.co.uk/posts/updated-football-rainmeter-fixtures/) for fixes

This post will show you how to create a dynamic Rainmeter skin that will show the upcoming two fixtures of your favourite team (as long as they are within the next 30 days). The end product looks a little something like this:

![](/images/finished_product.png)

To start, [download & install Rainmeter](https://github.com/rainmeter/rainmeter/releases/download/v4.0.0.2618/Rainmeter-4.0-r2618-beta.exe). The standard installation is fine.

If you look at your desktop, you’ll see something like this:
![](/images/1-default-skin.png)

Find Rainmeter in your taskbar and right click it. Hover over _illustro\\Welcome_ and click **unload skin**
![](/images/2-unload-welcome.png)

You will notice that the welcome message has disappeared.

You can download the improved illustro skin for your team using the links bellow:
- [Arsenal](/assets/rainmeter/old/Arsenal.zip)
- [Aston Villa](/assets/rainmeter/old/Aston-Villa.zip)
- [Bournemouth](/assets/rainmeter/old/Bournemouth.zip)
- [Burnley](/assets/rainmeter/old/Burnley.zip)
- [Chelsea](/assets/rainmeter/old/Chelsea.zip)
- [Crystal Palace](/assets/rainmeter/old/Crystal-Palace.zip)
- [Everton](/assets/rainmeter/old/Everton.zip)
- [Hull](/assets/rainmeter/old/Hull.zip)
- [Leicester](/assets/rainmeter/old/Leicester.zip)
- [Liverpool](/assets/rainmeter/old/Liverpool.zip)
- [Man City](/assets/rainmeter/old/Man-City.zip)
- [Man Utd](/assets/rainmeter/old/Man-Utd.zip)
- [Middlesbrough](/assets/rainmeter/old/Middlesbrough.zip)
- [Newcastle](/assets/rainmeter/old/Newcastle.zip)
- [Norwich](/assets/rainmeter/old/Norwich.zip)
- [Southampton](/assets/rainmeter/old/Southampton.zip)
- [Stoke](/assets/rainmeter/old/Stoke.zip)
- [Sunderland](/assets/rainmeter/old/Sunderland.zip)
- [Swansea](/assets/rainmeter/old/Swansea.zip)
- [Tottenham](/assets/rainmeter/old/Tottenham.zip)
- [Watford](/assets/rainmeter/old/Watford.zip)
- [West Brom](/assets/rainmeter/old/West-Brom.zip)
- [West Ham](/assets/rainmeter/old/West-Ham.zip)

I have added to the stand illustro skin for every team in the premier league plus Newcastle United, Norwich and Aston Villa. **This will only work for team fixtures in the next 30 days**. If you want a skin for a team that isn’t provided, you’ll have to follow the below steps or edit one of the provided skins.

____________________________________________________________________________________________________________

## Creating the Skin

**1 – Copy Default Skin to use as template**

Navigate to my documents to find the _skins_ directory. The install directory will be wherever you set it to be (usually in Profram Files) but Rainmeter will automatically create the a directory for storing skins within My Documents.

Copy the default **illustro** skin and rename the folder Football. Copy the **Clock** directory and rename it to Logo. Rename the Clock.ini file to Logo.ini

![](/images/3-logo-folder.png)

**2 – Download a logo**

Download your clubs logo into the @Resources directory within the **Football** skin. Here are some links to some high quality logos taken from their wikipedia page.

- [Arsenal](https://en.wikipedia.org/wiki/File:Arsenal_FC.svg)
- [Aston Villa](https://en.wikipedia.org/w/index.php?curid=22676697)
- [Bournemouth](https://en.wikipedia.org/wiki/File:AFC_Bournemouth_%282013%29.svg)
- [Burnley](https://en.wikipedia.org/wiki/File:Burnley_FC_badge.png)-
- [Chelsea](https://en.wikipedia.org/wiki/File:Chelsea_FC.svg)
- [Crystal Palace](https://upload.wikimedia.org/wikipedia/en/thumb/0/0c/Crystal_Palace_FC_logo.svg/481px-Crystal_Palace_FC_logo.svg.png)
- [Everton](https://en.wikipedia.org/wiki/File:Everton_FC_logo.svg)
- [Hull](https://en.wikipedia.org/wiki/File:Hull_City_Crest_2014.svg)
- [Leicester](https://en.wikipedia.org/wiki/File:Leicester_City_crest.svg)
- [Liverpool](https://en.wikipedia.org/wiki/File:Liverpool_FC.svg)
- [Man City](https://upload.wikimedia.org/wikipedia/en/thumb/c/cf/Manchester_City.svg/860px-Manchester_City.svg.png)
- [Man Utd](https://en.wikipedia.org/wiki/File:Manchester_United_FC_crest.svg)
- [Middlesbrough](https://en.wikipedia.org/wiki/File:Middlesbrough_crest.png)
- [Newcastle](https://en.wikipedia.org/wiki/File:Newcastle_United_Logo.svg)
- [Norwich](https://en.wikipedia.org/wiki/File:Norwich_City.svg)
- [Southampton](https://en.wikipedia.org/wiki/File:FC_Southampton.svg)
- [Stoke](https://en.wikipedia.org/wiki/Stoke_City_F.C.#/media/File:Stoke_City_FC.svg)
- [Sunderland](https://en.wikipedia.org/wiki/File:Logo_Sunderland.svg)
- [Swansea](https://en.wikipedia.org/wiki/Swansea_City_A.F.C.#/media/File:Swansea-City-Logo.png)
- [Tottenham](https://en.wikipedia.org/wiki/File:Tottenham_Hotspur.svg)
- [Watford](https://en.wikipedia.org/wiki/File:Watford.svg)
- [West Brom](https://en.wikipedia.org/wiki/File:West_Bromwich_Albion.svg)
- [West Ham](https://en.wikipedia.org/wiki/West_Ham_United_F.C.#/media/File:West_Ham_United_FC_logo.svg)

Download the file into the @Resources folder and rename it "logo" (this should make it PNG format). The full path should be something like `C:\\Users\\<username>\\Documents\\Rainmeter\\Skins\\Football\\@Resources`

**3 – Create Logo Plugin**

By copying the Clock folder and renaming it Logo, we have actually created a new Rainmeter plugin. The configuration hasn’t changed yet so it is a replica of the existing clock plugin. Right click on the Rainmeter icon from the taskbar on the bottom right and select manage. Click refresh all at the bottom left and you should now see the Football appear in the window. Expand the Football folder and expand the Logo folder. Right click on Logo.ini select load. If you look at your desktop, you’ll now have two sets of the Clock plugin (although we are going to change one to display the logo)

![](/images/4-load-clock-duplicate.png)

Back in my Documents, Navigate to `Rainmeter\\Skins\\Football\\Logo` and open the Logo.ini file.  Each bit of configuration underneath a `[section]` is called a meter. Delete all the `_measures_` (measureTime, measureDate and measureDay). Scroll down to the styling section and remove the styleSeperator meter.

Add a new meter by copying the following (change logo.png to whatever your image is called within the @Resources folder)

```
[MeterLogo]
Meter=Image
ImageName=#@#logo.png
W=128
H=128
X=41r
Y=34R
```

After the MeterLogo meter, add in the following meter which will display the team name (obviously substitute the team name as needed)
```
[meterTitle]
Meter=String
MeterStyle=styleTitle ; Using MeterStyle=styleTitle will basically "copy" the ; contents of the [styleTitle] section here during runtime.
X=100
Y=12
W=190
H=18
Text=Newcastle United
```

3 – Create Fixtures Plugin To create the fixtures plugin, we are going to use a similar method to before. We will copy an existing plugin so we have the styling, then we will edit the configuration to provide the fixture information using a data feed. You will need internet connection for this to work. Back at the Football directory. Copy the Logo folder and rename to Fixtures. Rename the logo.ini file to fixtures.ini. Delete the Logo meters at the bottom of the page.

![](/images/7-copy-clock-rename-to-Fixtures.png) The plugin should be pretty empty except the Rainmeter, metadata, variables and styling configuration. We need to find the fixture data that we want to be displayed. We can use the football-data.org API to get this information. Un-authenticated requests are capped at 100 a day, but that is more than enough for what we need. To save you some time, here are the URL’s for the fixture data for some popular premier league and a few championship sides for fixtures starting in the next 30 days: 
- Arsenal – http://api.football-data.org/v1/teams/57/fixtures?timeFrame=n30
- Aston Villa – http://api.football-data.org/v1/teams/58/fixtures?timeFrame=n30
- Bournemouth – http://api.football-data.org/v1/teams/1044/fixtures?timeFrame=n30
- Burnley – http://api.football-data.org/v1/teams/328/fixtures?timeFrame=n30
- Chelsea – http://api.football-data.org/v1/teams/61/fixtures?timeFrame=n30
- Crystal Palace – http://api.football-data.org/v1/teams/354/fixtures?timeFrame=n30
- Everton – http://api.football-data.org/v1/teams/62/fixtures?timeFrame=n30
- Hull – http://api.football-data.org/v1/teams/322/fixtures?timeFrame=n30
- Leicester – http://api.football-data.org/v1/teams/338/fixtures?timeFrame=n30
- Liverpool – http://api.football-data.org/v1/teams/64/fixtures?timeFrame=n30
- Man City – http://api.football-data.org/v1/teams/65/fixtures?timeFrame=n30
- Man Utd – http://api.football-data.org/v1/teams/66/fixtures?timeFrame=n30
- Middlesbrough – http://api.football-data.org/v1/teams/343/fixtures?timeFrame=n30
- Newcastle – http://api.football-data.org/v1/teams/67/fixtures?timeFrame=n30
- Norwich – http://api.football-data.org/v1/teams/68/fixtures?timeFrame=n30
- Southampton – http://api.football-data.org/v1/teams/340/fixtures?timeFrame=n30
- Stoke – http://api.football-data.org/v1/teams/70/fixtures?timeFrame=n30
- Sunderland – http://api.football-data.org/v1/teams/71/fixtures?timeFrame=n30
- Swansea – http://api.football-data.org/v1/teams/72/fixtures?timeFrame=n30
- Tottenham – http://api.football-data.org/v1/teams/73/fixtures?timeFrame=n30
- Watford – http://api.football-data.org/v1/teams/346/fixtures?timeFrame=n30
- West Brom – http://api.football-data.org/v1/teams/74/fixtures?timeFrame=n30
- West Ham – http://api.football-data.org/v1/teams/563/fixtures?timeFrame=n30

I’m using fixtures that start in the next 30 days because i don’t want a massive plugin that takes up half the page. Three or four fixtures are more than enough. If you cant find the team you are after you can:

Firstly, find the ID for the competition you are after by looking at http://api.football-data.org/v1/competitions/ 

Then, paste the ID in this link http://api.football-data.org/v1/competitions//leagueTable e.g. http://api.football-data.org/v1/competitions/427/leagueTable then press CTRL-F and search for your team (e.g. Wolverhampton) then find their Team ID:
```
{
    "\_links": {"team": {"href": "http://api.football-data.org/v1/teams/76"}},
    "position": 1,
    "teamName": "Wolverhampton Wanderers FC",
    "crestURI": "",
    "playedGames": 0,
    "points": 0,
    "goals": 0,
    "goalsAgainst": 0,
    "goalDifference": 0,
    "wins": 0,
    "draws": 0,
    "losses": 0,
    "home": {"goals": 0, "goalsAgainst": 0, "wins": 0, "draws": 0, "losses": 0},
    "away": {"goals": 0, "goalsAgainst": 0, "wins": 0, "draws": 0, "losses": 0},
}
```
Team ID is here is 76. Use that ID in the following URL to get the fixtures in the next 30 days. E.g. `http://api.football-data.org/v1/teams//fixtures?timeFrame=n30`

___________________________________________________

Now that we have the data we want, we need to feed this into the skin. We need to create a measure to get the data then create a meter to display it. The way this works is we will pass all that data into a measure. Then we will process all that text with a regular expression that will "extract" the dates and team name. There is a RainMeter regular expression builder tool that really simplifies building Regular Expressions but you can just use the regular expression that I have created as it should be the same. Here is the complete fixtures.ini file. You should only have to change the API link (highlighted) to change the team. You can download the .txt file here but you will have to change the extension to .ini and place the file in `Rainmeter\\Skins\\Football\\Fixtures` in order for it to work.

```
[Rainmeter]
Update=1000
Background=#@#Background.png
; #@# is equal to Rainmeter\\Skins\\Football\\@Resources
BackgroundMode=3
BackgroundMargins=0,34,0,14

[Metadata]
; Contains basic information of the skin.
Name=Logo
Author=poiru – original design so deserves the credit
Information=Displays football team logo
License=Creative Commons BY-NC-SA 3.0
Version=1.0.0

[Variables]
; Variables declared here can be used later on between two # characters (e.g. #MyVariable#).
fontName=Trebuchet MS
textSize=8
colorBar=235,170,0,255
colorText=255,255,255,205
; ———————————-
; STYLES are used to "centralize" options
; ———————————-

[styleTitle]
StringAlign=Center
StringCase=Upper
StringStyle=Bold
StringEffect=Shadow
FontEffectColor=0,0,0,50
FontColor=#colorText#
FontFace=#fontName#
FontSize=10
AntiAlias=1
ClipString=1

[styleLeftText]
StringAlign=Left
; Meters using styleLeftText will be left-aligned.
StringCase=None
StringStyle=Bold
StringEffect=Shadow
FontEffectColor=0,0,0,20
FontColor=#colorText#
FontFace=#fontName#
FontSize=#textSize#
AntiAlias=1
ClipString=1

[styleRightText]
StringAlign=Right
StringCase=None
StringStyle=Bold
StringEffect=Shadow
FontEffectColor=0,0,0,20
FontColor=#colorText#
FontFace=#fontName#
FontSize=#textSize#
AntiAlias=1
ClipString=1

[styleBar]
BarColor=#colorBar#
BarOrientation=HORIZONTAL
SolidColor=255,255,255,15
; ———————————-
; Meters are used to display something
; ———————————-
;;;;;;GET HOME TEAM INFO;;;;;

[getData]
Measure=Plugin
Plugin=WebParser
URL=http://api.football-data.org/v1/teams/67/fixtures?timeFrame=n30
RegExp=(?siU)date":"(.\*)-(.\*)-(.\*)T(.\*)homeTeamName":"(.\*)"(.\*)":"(.\*)"(.\*)date":"(.\*)-(.\*)-(.\*)T(.\*)homeTeamName":"(.\*)"(.\*)":"(.\*)"
;;home team name

[process1]
Measure=Plugin
Plugin=WebParser
URL=[getData]
StringIndex=5
;;fixture day

[process2]
Measure=Plugin
Plugin=WebParser
URL=[getData]
StringIndex=3
;;fixture month

[process3]
Measure=Plugin
Plugin=WebParser
URL=[getData]
StringIndex=2
;;fixture year

[process4]
Measure=Plugin
Plugin=WebParser
URL=[getData]
StringIndex=1
;;away team

[process5]
Measure=Plugin
Plugin=WebParser
URL=[getData]
StringIndex=7
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;repeat for second fixture;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;home team name

[process6]
Measure=Plugin
Plugin=WebParser
URL=[getData]
StringIndex=13
;;fixture day

[process7]
Measure=Plugin
Plugin=WebParser
URL=[getData]
StringIndex=11
;;fixture month

[process8]
Measure=Plugin
Plugin=WebParser
URL=[getData]
StringIndex=10
;;fixture year

[process9]
Measure=Plugin
Plugin=WebParser
URL=[getData]
StringIndex=9
;;away team

[process10]
Measure=Plugin
Plugin=WebParser
URL=[getData]
StringIndex=15
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

[meterTitle]
Meter=String
MeterStyle=styleTitle
X=100
Y=12
W=190
H=18
Text=Fixtures

[HomeTeamNameMeter]
Meter=String
MeterStyle=styleLeftText
MeasureName=process1
X=10
Y=40
W=190
H=14

[AwayTeamNameMeter]
Meter=String
MeterStyle=styleLeftText
MeasureName=process5
X=10
Y=56
W=190
H=14

[MeterDateDay]
Meter=String
MeterStyle=styleRightText
MeasureName=process2
X=152
Y=49
W=190
H=14

[meterDateDay.]
Meter=String
MeterStyle=styleRightText
X=5r
Y=49
W=190
H=14
Text=.

[MeterDateMonth]
Meter=String
MeterStyle=styleRightText
MeasureName=process3
X=13r
Y=49
W=190
H=14

[meterDateMonth.]
Meter=String
MeterStyle=styleRightText
X=5r
Y=49
W=190
H=14
Text=.
[MeterDateYear]
Meter=String
MeterStyle=styleRightText
MeasureName=process4
X=25r
Y=49
W=190
H=14
[seperatorMeter]
Meter=Bar
MeterStyle=styleBar
X=10
Y=72
W=190
H=2
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;Repeat for second fixture;;;;;;;;;;;;;;;;;;;;;;;;
[HomeTeamNameMeter2]
Meter=String
MeterStyle=styleLeftText
MeasureName=process6
X=10
Y=80
W=190
H=14

[AwayTeamNameMeter2]
Meter=String
MeterStyle=styleLeftText
MeasureName=process10
X=10
Y=96
W=190
H=14

[MeterDateDay2]
Meter=String
MeterStyle=styleRightText
MeasureName=process7
X=152
Y=89
W=190
H=14

[meterDateDay.2]
Meter=String
MeterStyle=styleRightText
X=5r
Y=89
W=190
H=14
Text=.

[MeterDateMonth2]
Meter=String
MeterStyle=styleRightText
MeasureName=process8
X=13r
Y=89
W=190
H=14

[meterDateMonth.2]
Meter=String
MeterStyle=styleRightText
X=5r
Y=89
W=190
H=14
Text=.

[MeterDateYear2]
Meter=String
MeterStyle=styleRightText
MeasureName=process9
X=25r
Y=89
W=190
H=14
```

The fixtures plugin will look a little something like this: ![](/images/8-fixtures.png) 

So how does this work? Basically, we create a Measure that grabs ALL the data from the API. Then we use a bunch of other sort of sub-measures to process that data. Then we call these sub-measures through a load of Meters which display the information nice and pretty on the screen.
```
;;;;;;GET HOME TEAM INFO;;;;;
[getData]
Measure=Plugin
Plugin=WebParser
URL=**http://api.football-data.org/v1/teams/67/fixtures?timeFrame=n30**
RegExp=(?siU)date":"(.\*)-(.\*)-(.\*)T(.\*)homeTeamName":"(.\*)"(.\*)":"(.\*)"(.\*)date":"(.\*)-(.\*)-(.\*)T(.\*)homeTeamName":"(.\*)"(.\*)":"(.\*)"
```

This is the most importatnt measure which i have called `getData`. It is using the WebParser plugin to process the API. Using the provided link, it will grab all that data and process it with the provided regular expression. If you want to change the team, all you need to do is update that URL corresponding to your team. You should need to change the regular expression unless you want to display more than two fixtures or the timeframe of the fixtures. Lets use the Rainmeter Regular Expression tool to help explain how we have processed this information. Using the API link that shows fixtures within the next 30 days (you can extend this by changing the ?timeFrame=n30 part at the end. E.g. =n40 is next 40 days, p20 is previous 20 days etc…). By navigating to the URL in the browser, we see a wall of text but within this is the information we want. We will use regular expressions to process the data into sections (string index’s) that we will then reference from various rainmeter plugins.

![](/images/9-rain-reg-expression.png)

Open the RainRegExp tool and paste in your API link into the bar on the top left and click Connect. Click Wrap on the top right so that the data is easier to read. Looking at the data, the first information that we are interested in is the date of the first fixture within this next 30 day timeframe.

If you look carefully, the way we can identify this bit of date is that it is preceded by the word date followed by the ":" characters. The date we want is hidden within this text "date":"2016-08-05T18:45:00Z

Using a regular expression, we can search the text from the beginning of all the data until we find exactly date":" and we can store the value after that (2016). We can then search from that last point until we find a dash and store the value after that (08 in this example). We can then search until we get to the first capital T after that and store the value in between to give us the day (05). Try this regular expression and press Parse to process:

`(?siU)date":"(.\*)-(.\*)-(.\*)T(.\*)`

Looking at the print screen, you can see the values we want stored. We will call these by using the StringIndex option within a Measure.

```
;;fixture day
[process2]
Measure=Plugin
Plugin=WebParser
URL=[getData]
StringIndex=3
;;fixture month

[process3]
Measure=Plugin
Plugin=WebParser
URL=[getData]
StringIndex=2
;;fixture year

[process4]
Measure=Plugin
Plugin=WebParser
URL=[getData]
StringIndex=1
```

Here i created 3 measures that are poorly named process2, 3 & 4. They reference the getData measure (which actually makes the request) via the URL followed by the measure name. Then they have a StringIndex value which corresponds to those values stored by the RegularExpression. So my process4 measure has the StringIndex 1 which would correspond to 2016 in this example and Process 3 has a StringIndex of 2 which corresponds to 08 and so on…. This concept is how the whole thing works. The data we want is extracted from the URL, then displayed with a measure. In between each date value (day, month year) I created a measure that literally just displays a dot. This is positioned between each value to make it read DD.MM.YYYY. I didnt need to do this, I would have just extracted the date as it readily available 2016-08-05 but i dont like the format. The home team name is extracted and so is the away team. They are both called via a meter and displayed on the screen. The whole process is repeated (another set of measures, another set of meters) and the second fixture is displayed. I could extend the regular expression even further like so Three Fixtures: `(?siU)date":"(.\*)-(.\*)-(.\*)T(.\*)homeTeamName":"(.\*)"(.\*)":"(.\*)"(.\*)date":"(.\*)-(.\*)-(.\*)T(.\*)homeTeamName":"(.\*)"(.\*)":"(.\*)"(.\*)date":"(.\*)-(.\*)-(.\*)T(.\*)homeTeamName":"(.\*)"(.\*)":"(.\*)"(.\*)` This would process the data for three fixtures, but they wouldnt be displayed until additional measures and meters are created. This can be done by copying and pasting the existing ones and updating the names. If the skin were extended any more than this, the API URL would have to be updated to show fixtures within the next 60 days – timeFrame=n60. Although this hasnt been a complete guide, it should give you a basic understanding of how this was build. Let me know if you want some help extending the functionality or understanding in more detail.
