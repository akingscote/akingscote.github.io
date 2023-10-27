---
title: "SOAP Web Services in Python with Nested ComplexTypes and Lists"
date: "2017-06-23"
categories: 
  - "development"
  - "python"
tags: 
  - "python"
  - "soap"
  - "spyne"
  - "web-services"
  - "zeep"
---

Simple Object Access Protocol (SOAP) is a dated web service protocol that refuses to die. In the RESTful world of the web, SOAP still persists in legacy systems that refuse to update. This post is intended as a quick way to get up and running with SOAP with some fiddely functionality thats made simple by Python.

A SOAP service publishes a Web Service Description Language (WSDL) that defines the message format that will be accepted to the service. For example, i could say that my SOAP service accepts an element called name and the value must be a string -

Or i could say that i have an element called person, and each person must have a name which must be a string and an age which must be an integer.

    <name>akingscote</name>
    <age>100</age>

In WSDL, an element that contains multiple different types is called a ComplexType. So in the previous example, person would be dictated as a ComplexType. These are trivial examples, but it can get horrifically convoluted and complicated as a WSDL could dictate elements with multiple sub elements that each have multiple sub-elements all of different types.

For this example, i am using two different Python libraries. The first is called [zeep](http://docs.python-zeep.org/en/master/) and it is a SOAP client meaning it does not host the service, but connects to it. The second library is called [spyne](http://spyne.io/#inprot=Soap11&outprot=Soap11&s=rpc&tpt=WsgiApplication&validator=true) and is used in this example to host the SOAP service.

# Creating and Hosting the Service

In this example, im creating a number of ComplexTypes to demonstrate how to create the service with nested complex type element - a common business scenario. `class Player(ComplexModel): name = String age = Integer position = String class League(ComplexModel): name = String total_teams = Integer tier = Integer  class Team(ComplexModel): team_name = String player = Array(Player, wrapped=False) year_formed = Integer country = String league = League`

The player class here dictates a name which must be a string, an age which must be an integer and a position which must be a string. It is a ComplexType because it contains multiple and different types (String and Integer). In the Team class, I have declared that there must be multiple players. This was achieved by making the player call an Array (list) of the player class. The result is something like

And so on.... Still Within the team class, I have said that the league must be the League ComplexType which means that the league will look something like this
```
<league>
    <name></name>
    <total_teams></total_teams>
    <tier></tier>
</league>
```
 

So an example of the Team class is expected to look something like this:
```
<league>
<team>
    <team_name>STRING VALUE</team_name>
    <player>
        <name>STRING VALUE</name>
        <age>INTEGER VALUE</age>
        <position>STRING VALUE</position>
    </player>
    ...
    <player>
        <name>STRING VALUE</name>
        <age>INTEGER VALUE</age>
        <position>STRING VALUE</position>
    </player>
    <league>
        <name>STRING VALUE</name>
        <total_teams>INTEGER VALUE</total_teams>
        <tier>INTEGER VALUE</tier>
    </league>
</team>
```
 

At this stage, the relationships are defined but they are not published on the service. The service im creating just has one method and it just displays the data its been sent and then returns it back.
```
class MyService(ServiceBase):
    @srpc(Team, _returns=Team)
    def display_team_info(received_value):
        print 'Received {}'.format(received_value)
        return received_value
```
This service uses the ServiceBase subclass to do the actual work. Using the @srpc (stateless remote procedure call) spyne decorator, ive said that this method must accept a **Team** complex type. This could be changed to accept an individual type (String or Integer) instead of a ComplexType, but that would make things too easy. I have said that this method returns (via the _returns assignment) type **Team** (again i could return string or an integer). The actual method name is called display_team_info and just accepts one parameter (which must be of type Team as declared by the @srpc(Team) section).

Everything is then tied together and hosted on a WSGI server (using the localhost in this example). Full Service Code:
```
"""
SOAP service hosted on a WSGI server
Defines a Complex object
- demonstrates nested instances of classes
- demonstrates arrays in SOAP
"""

import logging
from spyne import Application, Unicode, ComplexModel, Array

from spyne.service import ServiceBase
from spyne.decorator import srpc

from spyne.protocol.soap import Soap11
from spyne.model.complex import Iterable
from spyne.model.primitive import Integer
from spyne.model.primitive import String
from spyne.model.primitive import UnsignedInteger
from spyne.model.primitive import String

from spyne.server.wsgi import WsgiApplication

from wsgiref.simple_server import make_server

class Player(ComplexModel):
    name = String
    age = Integer
    position = String

class League(ComplexModel):
    name = String
    total_teams = Integer
    tier = Integer

class Team(ComplexModel):
    team_name = String
    player = Array(Player, wrapped=False)
    year_formed = Integer
    country = String
    league = League

class MyService(ServiceBase):
    @srpc(Team, _returns=Team)
    def display_team_info(received_value):
        print 'Received {}'.format(received_value)
        return received_value

if __name__ == '__main__':
    logging.basicConfig(level=logging.INFO)
    logging.getLogger('spyne.protocol.xml').setLevel(logging.INFO)

    app = Application([MyService], 'spyne.examples.hello.http',
    in_protocol=Soap11(validator='lxml'),
    out_protocol=Soap11(),
    )

    server = make_server('127.0.0.1', 8000, WsgiApplication(app))

    logging.info("listening to http://127.0.0.1:8000")
    logging.info("wsdl is at: http://localhost:8000/?wsdl")

    server.serve_forever()
```

Navigating to `http://localhost:8000/?wsdl` you will be able to see the WSDL. (Firefox had some issues with rendering the XML so use Chrome if you cant see anything) ![WSDL Screenshot](/images/example-WSDl.png)

# Creating the Client

The client must connect to the services WSDL to understand the services & methods formats (called SOAP messages). Using the Zeep factory functionality, this is pretty painless.

```
from zeep import Client  #connect to the service and get the WSDL
client = Client("http://127.0.0.1:8000/?wsdl")  #as we need to create multiple types, use a factory
factory = client.type_factory("ns0")
```
The "ns0" will refer to all types of the WSDL rather than a specific few, this is fine as we need to get all of the nested class definitions as well. We know that we need to create some players and a league and then pass them into the team object. You can either pass the data directly into the class instantiation and consquently into the classes __init__, or you could instantiate the class and then assign each attribute a value. The first method is more efficient, but the second helps to write more readable code when dealing with these nested classes.

```
#instantiate and pass variables to init
player1 = factory.Player(name="akingscote", age=100, position="defender")
player2 = factory.Player(name="aqueenscote", age=101, position="forward")
player3 = factory.Player(name="aprincecote", age=5, position="midfield")  #or instantiate, then assign the attribute variables. Makes the code easy to read team = factory.Team()
team.team_name = "Newcastle United"
team.year_formed = 1892
team.country = "United Kingdom" #instance within an instance
team.league = factory.League(name="Premier League", total_teams=20, tier=1)
team.player = [player1, player2, player3]
```

The objects are created from the factory class. This is how the client gets the class definitions. Notice how simple it was to create the multiple players, the instances are just placed into a list. On the server side, the Team class does all the work with the

`player = Array(Player, wrapped=False)`

Finally, we send the data to the service. In this example, the service is returning the data it receives. So on the client im just printing the response to see the SOAP message. An alternative to this is to change the logging level to debug on the server.
```
with client.options(raw_response=True):
    response = client.service.display_team_info(team)
    print response.content
```

Client Code: 
```
"""SOAP Client that sends a basic data type to a service"""
from zeep import Client  #connect to the service and get the WSDL
client = Client("http://127.0.0.1:8000/?wsdl") #as we need to create multiple types, use a factory
factory = client.type_factory("ns0")  #instantiate and pass variables to init
player1 = factory.Player(name="akingscote", age=100, position="defender")
player2 = factory.Player(name="aqueenscote", age=101, position="forward")
player3 = factory.Player(name="aprincecote", age=5, position="midfield")  #or instantiate, then assign the attribute variables. Makes the code easy to read team = factory.Team()
team.team_name = "Newcastle United"
team.year_formed = 1892
team.country = "United Kingdom" #instance within an instance
team.league = factory.League(name="Premier League", total_teams=20, tier=1)
team.player = [player1, player2, player3] 
with client.options(raw_response=True):
    response = client.service.display_team_info(team)
    print response.content
```

So start up the server first, check you can access the WSDL (again, firefox is weird and dosent like displaying XML so use chrome) and then run the client. Using a wireshark capture on the local interface, from the client console or from the debug output on the server, you will see a SOAP response that looks like this:
```
<?xml version='1.0' encoding='UTF-8'?>
<soap11env:Envelope
	xmlns:soap11env="http://schemas.xmlsoap.org/soap/envelope/"
	xmlns:tns="spyne.examples.hello.http">
	<soap11env:Body>
		<tns:display_team_infoResponse>
			<tns:display_team_infoResult>
				<tns:league>
					<tns:name>Premier League</tns:name>
					<tns:total_teams>20</tns:total_teams>
					<tns:tier>1</tns:tier>
				</tns:league>
				<tns:country>United Kingdom</tns:country>
				<tns:year_formed>1892</tns:year_formed>
				<tns:player>
					<tns:name>akingscote</tns:name>
					<tns:age>100</tns:age>
					<tns:position>defender</tns:position>
				</tns:player>
				<tns:player>
					<tns:name>aqueenscote</tns:name>
					<tns:age>101</tns:age>
					<tns:position>forward</tns:position>
				</tns:player>
				<tns:player>
					<tns:name>aprincecote</tns:name>
					<tns:age>5</tns:age>
					<tns:position>midfield</tns:position>
				</tns:player>
				<tns:team_name>Newcastle United</tns:team_name>
			</tns:display_team_infoResult>
		</tns:display_team_infoResponse>
	</soap11env:Body>
</soap11env:Envelope>
```
