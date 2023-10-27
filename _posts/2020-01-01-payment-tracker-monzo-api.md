---
title: "Payment Tracker - Monzo API"
date: "2020-01-01"
categories: 
  - "development"
  - "python"
  - "web-development"
tags: 
  - "flask"
  - "http"
  - "monzo"
  - "python"
  - "rest"
  - "vuejs"
  - "web"
---

Where i currently work, i run company wide 5-a-side football sessions. The pitch hire costs around £45 and I charge £5 per person. Unless people pay me promptly, i quickly start to run out of pocket. Currently, I have a gross spreadsheet which i manually update with who has sent me money. As an engineer, this really rubs me up the wrong way and ive had an itch to automate the process for a long time.

I use Monzo as my bank as its brilliant. I can quickly manage my finances and recieve instant notifications on my phone when money has entered my account. This has been especially useful for updating my spreadsheet. Monzo seems like a great company and employs some [modern engineering practices](https://monzo.com/blog/2016/09/19/building-a-modern-bank-backend) and i can only see their company expanding. Monzo have an API so you can integrate your own projects into their system. You cant make transactions, but you can get all your own data really easily. Go to https://developers.monzo.com/ and sign in with your account. Their API playground means you can take a look at the API without having to touch any code.

The plan is to create a web app which i can use to create & manage fixtures and players. When i say create fixtures, i dont mean actually book them, i mean just manually create the fixtures when i have booked them on the phone. When i receive payment, i want my web app to automatically figure out who the payment is from and mark them as not owing me any money.

I normally cancel a fixture if i get any less than 8 people accept. Once i have 12 players, i dont accept any more players. Normally we average around 9. So a fixture can have a variable number of players. A fixture also has a date, a time, a cost and a location. I could use a relational database, but as this is just a bit of fun, i think itll be much easier to use a document driven database. In the past, ive used CouchDB which i love, but im keen to play around with MongoDB as its extremly popular. In hindsight, i really should done some more research - CouchDB would have been much easier to use.

To receive the push notifications, i can use the Monzo webhooks to send a POST message containing the data. To register a webhook, you just define a URL on developers.monzo.com under register webhooks. When a transaction appears, Monzo will attempt to send a POST to the provided URL containing the information. I obviously didnt want to develop on the internet and i didnt want to open up ports on my local PC to the internet so monzo could reach them. What i did was i created a Python Flask endpoint, registered it as a webhook on monzo, ran it on my online server (this site) and then captured the data. I did two requests, one from another monzo account and one from a normal bank to see if the messages looked any different. I captured the two responses and saved them as requests in postman. This way i can simulate the requests and continue development locally.

![](/images/postman_example-1024x435.png)

My flask function looked like this:

```
@app.route('/monzo_webhook', methods=["POST"])
def monzo_notification():
    transaction_date = request.json["data"]["created"]
    #for monzo, description is the sender. for bank accounts its the reference
    description = request.json["data"]["description"]
    #amount is in pence
    amount = request.json["data"]["amount"]
    #for non-monzo, notes are the reference (same is description)
    notes = request.json["data"]["notes"]
    sender = request.json["data"]["counterparty"]["name"]
    result = get_user_record(sender)
    if result[0] == False:
        result = get_user_record(description)
        if result[0] == False:
            print(result[1])
            return jsonify(result[1])
    record = result[1]
    record["total_paid"] = int(record["total_paid"])
    record["total_paid"] += amount
    print("total paid is {}".format(record["total_paid"]))
 
    #update the record
    mydatabase = client["wednesdayfootball"]
    filter = {
        "_id" : record["_id"]
    }
 
    update = {
        '$set': {            
            "total_paid" :  record["total_paid"]
        }
    }
    return update_player(mydatabase, filter, update)
```
The post function isnt very clean, but its fine for a proof of concept. I found that if I received some money from a monzo account, then the description field was the sender name. If i received money from a non-monzo account, then the sender field contain the sender name.

In my web app, im using pymongo to connect to a mongoDB server which runs on the same server. In the Flask server, i create numerous API endpoints which when triggered, use pymongo to connect to the database and return the data (if it exists). This is because MongoDB dosent include a REST api by default (couchdb does though!). Its ok though, because i need a custom server to create my Monzo webhook endpoint anyway. I created the following endpoints and added simulated data into my Postman client.

- Create Fixture
- Create Player
- Get Player
- Get Players
- Get Fixture
- Get Fixtures
- Delete Player
- Delete Fixture

For the single requests, such as get fixture or delete fixture, i provide the ID of the record i want to delete in the URL request. This makes the sever API very simple.

\[code language="python"\] from bson.objectid import ObjectId from bson.errors import InvalidId ... @app.route('/api/{}/delete\_fixture'.format(api\_version), methods=\["DELETE"\]) def deleteFixture(): id = request.args.get("id") if id == None: return jsonify("request must contain id in url. e.g localhost:5000/api/v1/delete\_fixture?id=12345678") mydatabase = client\["wednesdayfootball"\] try: query = { '\_id' : ObjectId(id) } except InvalidId: return jsonify("provided ID is invalid - {}".format(id)) record = mydatabase.games.delete\_one(query) if record.deleted\_count == 1: return jsonify("record deleted") return jsonify("ID - {} not deleted in mongo db wednesdayfootball fixtures db".format(id)) \[/code\]

Then, in my Javascript, all i need to do is call HTTP DELETE to the delete fixture url suffixed with "?id=xxxxxx"

For the frontend, i wanted to use VueJS. Its a great framework and i have experience with it, so thought i could knock something up really quickly. I use vue-bootstrap to help with the styling. The trouble is, i wanted to develop locally but not use the VueJS node development server. My solution was pretty sneaky, instead of running the development server, i would build my solution properly using "npm run build". I just configured my flask server to look for a couple of directories in its immediate child. When i was doing a live deployment, i would copy these files over from my VueJS dist directory, but for development, i just used a symbolic link (something new to me in Windows).

So my directory structure looked like this:

![](/images/symbolic-links.png)

The folders/files highlighted in red are actually symbolic links to the folders/files in blue

This way, each time i did a "npm run build" the flask server would be serving the latest build. Then when i did an actual deployment to my online sever, i just had to copy the actual folders/files (js, css and index.html) over to the flask server. If youre curious, here are the commands i used the create the sym links on windows (img isnt used).

![](/images/symbolic-links-screenshot-1.png)

So now the project is taking shape, i have a bunch of API endpoints on my flask server, i have test requests in Postman, i have a VueJS frontend with numerous pages (i used vue-router). On the frontend, i use Axios to complete my HTTP requests. I can create users, create fixtures, update fixtures, update users and delete users/fixtures.

However, something that is missing that is trivial to implement in CouchDB but difficult to implement in MongoDB is push notifications. When i CRUD a record, i want my frontend to know about it and to automatically display the updated data. In Couchdb, thats as simple as opening up a Server Sent Event subscriber to the provided Changes API, but in MongoDB its more complicated. With Mongo, my frontend dosent talk to it directly, but rather through the API endpoints that Flask is providing. So how can i get push notifications using the MongoDB changes? Short answer is painfully. You cant get change streams on standard mongodb collections, it has to be performed on a replica set which is used to provide replication and failover on mongodb data. Id then need to have Flask open connections to Mongo and create SSE endpoints for my frontend to connect to. Ive instantly regretted using mongo and not couch. However, for my purposes it dosent matter, i can just manually refresh the page for now.

On my Players record, i added a field called "accepted\_bank\_names" and this is an array of strings which show what transaction names i should associate with a user. On my frontend, you can create/update users to update this field to get it right. When i receive a transaction from monzo, i search couchdb and try to find a match. If there is a match, the transaction amount is added to the total for that player.

![](/images/create_player.png)

Here you can see i already have a player called ashley with a couple of entries in the accepted bank names

https://www.youtube.com/watch?v=I-hZoPGGzcQ&feature=youtu.be

The full code can be found here - [https://gitlab.com/Kingscote/football-monzo](https://gitlab.com/Kingscote/football-monzo)

Its not pretty or completely finished, i may re-make it properly.
