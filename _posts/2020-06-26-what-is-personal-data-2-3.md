---
title: "What is Personal Data? (2/3)"
date: "2020-06-26"
categories: 
  - "development"
  - "golang"
  - "miscellaneous"
tags: 
  - "1984"
  - "data"
  - "open"
  - "postgis"
  - "public"
coverImage: "Big_Brother_2007_UK_logo.png"
---

Continuing on from my first post, this post focuses on sourcing and massaging my data into a usable format ready for cross referencing and some analysis. As detailed in my first post, i received 38,742 records of names and addreses across 52 different files of people on the open register in Reading.

The columns in the data set are: "Elector Number, Prefix, Elector Number,Elector Number Suffix,Elector Markers,Elector DOB,Elector Name,PostCode,Address1,Address2,Address3,Address4,Address5,Address6". The DOB field is empty for nearly all the records.

Im increasingly using Golang, so i thought it would be a good idea to process this data to consoliate into a single table in a sqlite database. Ill dump some of the code here, but it was just strung together quickly. This may not work entirely as ive cut and pasted from multiple files.
```
package main
 
import (
    "fmt"
    "io/ioutil"
    "log"
    "database/sql"
    "io"
    "github.com/360EntSecGroup-Skylar/excelize"
)
 
 
type ElectoralRow struct {
    Area                string
    ElectorNumberPrefix string
    ElectorNumber       int
    ElectorNumberSuffix int
    ElectorMaker        string
    ElectorDOB          string
    ElectorName         string
    PostCode            string
    Address1            string
    Address2            string
}
 
func main() {
    database := createTables()
    addElectoral(database)
}
 
 
createTables() (sql.DB){
    db, _ := sql.Open("sqlite3", "./database.sqlite")
    statement, _ := db.Prepare(`CREATE TABLE IF NOT EXISTS openReg (id INTEGER PRIMARY KEY,
         area TEXT, electorNumberPrefix TEXT, electorNumber INTEGER, electorNumberSuffix INTEGER,
         electorMaker TEXT, electorDOB TEXT, electorName TEXT, postCode TEXT, address1 TEXT, address2 TEXT)`)
    statement.Exec()
   return *db
}
 
func addElectoral(database sql.DB){
    var openRegDir string = "C:\\Users\\ashkingie\\Desktop\\goLearn\\src\\openReg\\"
    files, err := ioutil.ReadDir(openRegDir)
    if err != nil {
        log.Fatal(err)
    }
    for _, file := range files {
        var fullPath string = fmt.Sprintf("%s%s", openRegDir, file.Name())
        fmt.Println("Opening ", fullPath)
 
        if fullPath[len(fullPath)-4:] != "xlsx" {
            fmt.Println("BREAKING - ", fullPath)
            break
        }
 
        f, err := excelize.OpenFile(fullPath)
        if err != nil {
            fmt.Println("ERROR opening file", fullPath, err)
            return
        }
 
        var sheetName string = "Sheet1"
        rows, err := f.GetRows(sheetName)
         
        if err != nil {
            fmt.Println("error getting sheet", sheetName, fullPath)
        }
         
        for _, excelRow := range rows {
            data := electoralRowToStruct(file.Name(), excelRow)
            _ = electoralRowToStruct(file.Name(), excelRow)
            insertOpenReg(database, data)
        }
         
    }
}
 
 
func electoralRowToStruct(fileName string, excelRow[] string) (ElectoralRow){
    electorNumber, err := strconv.Atoi(excelRow[1])
    if err != nil {
        electorNumber = 0
    }
 
    electorNumberSuffix, err := strconv.Atoi(excelRow[2])
    if err != nil {
        electorNumberSuffix = 0
    }
    fileName = processElecFilename(fileName)
    //make address uppercase for consistency
 
    rowData := ElectoralRow{
        Area:                fileName,
        ElectorNumberPrefix: excelRow[0],
        ElectorNumber:       electorNumber,
        ElectorNumberSuffix: electorNumberSuffix,
        ElectorMaker:        excelRow[3],
        ElectorDOB:          excelRow[4],
        ElectorName:         excelRow[5],
        PostCode:            strings.ToUpper(excelRow[6]),
        Address1:            strings.ToUpper(excelRow[7]),
        Address2:            strings.ToUpper(excelRow[8]),
    }
    return rowData
}
 
func processElecFilename(filename string) (string) {
    if filename[len(filename)-5:] == ".xlsx" {
        filename = filename[:len(filename)-5]
    }
    split := strings.Split(filename, "-")
    filename = split[1]
    return filename
}
 
func insertOpenReg(database sql.DB, row dataparse.ElectoralRow){
    statement, _ := database.Prepare("INSERT INTO openReg (area, electorName, address1, postCode) VALUES (?, ?, ?, ?)")
    statement.Exec(row.Area, row.ElectorName, row.Address1, row.PostCode)
}
```

Once the data is in a single table, it's much easier to manage for processing as its in a consistent format. The next step is the most interesting. How on earth do i plot an address onto a map?

Post GIS uses the concept of a Geometry - [](http://postgis.net/workshops/postgis-intro/geometries.html)So i need to convert an address into a Geometry. The way to do that is to first convert an address into a latitude and longitude. The way I decided to do this was to use Google Geocoding API - you pass it an address and it will return the latitude and longitude.

Google offer a free trial of their API, you get $300 dollars free for 12 months and no autocharge after the trail ends.

![](/images/geocoding-requests.png)

Looking at their pricing sheet, it only costs a couple of dollars for 40K requests, so even if i exceed the free trial i shouldnt get a huge bill. Sign up for an API key, it must be type **Server key**. I wont post the full code here, but in a nutshell, i signed up and got an API key. Using Golang, i iterate through every record and make sure that i have house number, road and postcode information. If i have that information, i just use the golang HTTP client and send a request to the API. I process the response and store the values in a "latitude" and "longitude" column for that record in the sqlite database.

```
const apiKey = "api key as a string here"
const urlBase = "https://maps.googleapis.com/maps/api/geocode/json?"
 
//if you dont have the address, you can provide the postcode and return the address
//provide a URL and it will get the data using the API key. 
//throttling is implemented so maybe add a delay after each request
func GetLatLong(houseNumber string, road string, postcode string) (GeoCodeResponse){
    //https://maps.googleapis.com/maps/api/geocode/json?address=130%StSaviours Road%20Reading%20UK&key=<key_here>
    base, _ := url.Parse(urlBase)
    params := url.Values{}
 
    if houseNumber == "0" {
        params.Add("address", fmt.Sprintf("address=%s %s", road, postcode))
    } else {
        params.Add("address", fmt.Sprintf("address=%s %s %s", houseNumber, road, postcode))
    }
    params.Add("key", apiKey)
    
    base.RawQuery = params.Encode() 
 
    gourl := base.String()
    fmt.Println("URL IS", gourl)
    resp, err := http.Get(gourl)
    if err != nil {
        log.Error("error getting data from ", gourl, err)
    }
     
    var response GeoCodeResponse
    if resp.StatusCode != 200 {
        log.Error("didnt receive 200 from url", gourl)
        } else {
            defer resp.Body.Close()
            body, error := ioutil.ReadAll(resp.Body)
            if error != nil {
                log.Error("error reading body of data ", error)
            }
            if err := json.Unmarshal(body, &response); err != nil {
                log.Error("error unmarshalling response to struct", err)
            }
             
            // fmt.Printf("%+v\n", response)
        }
     
    return response
    }
```

There is API throttling in place, so I put a small sleep in place after each iteration although its not in this code snippet as it was cut and pasted.

The results are amazing, the geocoding API returns the coordinates with fantastic accuracy. I tested on my own property and it was spot on. So now i have 38K names and addresses in reading, with lat/long coordinates. Im starting to feel creepy - even though this is OPEN data.

My next task was to repeat the process for some extra data sources. There are tonnes of sources out there and i can sign up for infinite free trials, but this really is just a proof of concept. The next data source im going to mine is going to be Companies House.

According to the National Office of Statistics - there are more than 5 million people in the UK who are self employed. Anyone who is a contactor, works for themselves and is self employed. [ONS - Coronavirus and self-employment in the UK](https://www.ons.gov.uk/employmentandlabourmarket/peopleinwork/employmentandemployeetypes/articles/coronavirusandselfemploymentintheuk/2020-04-24) This means that anyone who is self employed or a contractor, will likely have their information on companies house.

So what i did was download the companies house data from their website -[](http://download.companieshouse.gov.uk/en_output.html)

The data format for companies house was excel files. It was HUGE. I got 6 .xlsx files each 400MB. The information that I received was the companies registered address, whether or not the company was active, whether their accounts were up to date, the category of the company and the most crucial bit of information - the URL to their record online.

Similar to with the open register data, i wrote a go programme that opened each file, iterated through each records and added to an sqlite table when the data looked usable (was in Reading and had the relevant fields populated). In the end, i found about 22K records that were usable for Reading.

Now it turns out that Companies House has an API and you can sign up for an API key.[](https://developer.companieshouse.gov.uk/api/docs/company/company_number/readCompanyProfile.html)

If you go to the a companies record (once authenticated) you can add the "/officers" suffix and it will list the name and address of all registered officers for that company. So my 22K was about to explode in size. Each company would surely have at least 2 officers registered and hopefully they are local to the area.

I just registered with companies house, then go to "your applications" and it will list your API key.[](https://developer.companieshouse.gov.uk/developer/applications)

```
curl -uYOUR_APIKEY_FOLLOWED_BY_A_COLON: https://api.companieshouse.gov.uk/company/{company_number}
curl -u xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx: https://api.companieshouse.gov.uk/company/12279804
```

Here is an example resposne for company 12279804
```
{
   "links":{
      "filing_history":"/company/12279804/filing-history",
      "persons_with_significant_control":"/company/12279804/persons-with-significant-control",
      "self":"/company/12279804",
      "officers":"/company/12279804/officers"
},
   "company_name":"MY LANE CATERING LTD",
   "has_insolvency_history":false,
   "confirmation_statement":{
      "next_made_up_to":"2020-10-23",
      "next_due":"2020-11-06",
      "overdue":false
},
   "sic_codes":[
      "56103",
      "56210"
],
   "type":"ltd",
   "registered_office_address":{
      "locality":"Reading",
      "country":"United Kingdom",
      "address_line_1":"375 Oxford Road",
      "postal_code":"RG30 1HA"
    
},
   "etag":"9ce8a74fbcfd6acd13fd6634d3ab18fc09fdd330",
   "jurisdiction":"england-wales",
   "accounts":{
      "next_made_up_to":"2020-10-31",
      "next_due":"2021-07-24",
      "next_accounts":{
         "due_on":"2021-07-24",
         "period_end_on":"2020-10-31",
         "overdue":false,
         "period_start_on":"2019-10-24"
       
},
      "last_accounts":{
         "type":"null"
},
      "accounting_reference_date":{
         "day":"31",
         "month":"10"
},
      "overdue":false
},
   "company_status":"active",
   "undeliverable_registered_office_address":false,
   "registered_office_is_in_dispute":false,
   "company_number":"12279804",
   "has_charges":false,
   "date_of_creation":"2019-10-24",
   "can_file":true
}
```

Appending the /officers shows me much more interesting information.
```
curl -u xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx: https://api.companieshouse.gov.uk/company/12279804/officers
...
{
  "inactive_count": 0,
  "items_per_page": 35,
  "start_index": 0,
  "kind": "officer-list",
  "items": [
    {
      "links": {
        "officer": {
          "appointments": "/officers/BIIB3OKYK2_8edq-Ue_V9k2cxtU/appointments"
        }
      },
      "address": {
        "country": "United Kingdom",
        "premises": "375",
        "locality": "Reading",
        "address_line_1": "Oxford Road",
        "postal_code": "RG30 1HA"
      },
      "name": "KING, Louis",
      "appointed_on": "2019-10-24",
      "officer_role": "secretary"
    },
    {
      "country_of_residence": "United Kingdom",
      "links": {
        "officer": {
          "appointments": "/officers/5k9v8g6jEtW9JcJZARxfaF67hhI/appointments"
        }
      },
      "date_of_birth": {
        "year": 1983,
        "month": 10
      },
      "nationality": "British",
      "officer_role": "director",
      "name": "KING, Louis",
      "appointed_on": "2019-10-24",
      "address": {
        "postal_code": "RG30 1HA",
        "country": "United Kingdom",
        "locality": "Reading",
        "premises": "375",
        "address_line_1": "Oxford Road"
      },
      "occupation": "Director"
    }
  ],
  "resigned_count": 0,
  "active_count": 2,
  "links": {
    "self": "/company/12279804/officers"
  },
  "total_results": 2,
  "etag": "88d52f539f66c5198500f3af3729a02a199f331b"
}
```
Now it may seem a bit tight posting this information here, but remember this is the whole point in this post. Its PUBLIC and OPEN information. Louis King was born in October 1983 and has a business (and potentially lives at) 375 Oxford Road.

So what I did, was i ammended the go programme to go each records URL and grab the officer data for each company. I then added that to my SQLite database. Companies House API also have throttling on their API so i had to put a delay in and run the programme over night. For companies house data alone, i got 112,086 records. This details all the companies registered in Reading and their addresses. It also lists the officers names, date of births and addresses - although some of these are not in Reading.

Ammending my code as i did before, i passed it through the geocode API. Unfortunately I had SO many records that i quickly used up my $300 free trail.
![](/images/geocoding-landreg-max.png)

So eventually i my programme returned access denied when trying to geocode the data. I decided that i would just leave it there as its just about proving a concept and not doing this properly. The geocoding took ages and unfortunately i made a huge mistake. I was writing the data to a CSV rather than to the database directly - purely out of laziness so that i could reuse some code. I opened up the CSV and sorted it and somehow messed up the ordering!! Fortunately i had a smaller subset of the data - about 20K records where i had previously tested the API. I invested a lot of time trying to undo my mistake but unfortunately the data was unusable. I considered signing up for another free trial key and re-running but decided that it wasnt worth it. 20K records will do.

My final data set for this proof of concept (for now) is going to be the land registry.

What i did was go onto the land registry website and download all the data for Reading [](https://landregistry.data.gov.uk/app/ppd)

Interestingly, you can pay and get information on a specific property - around £3 per house. [](https://eservices.landregistry.gov.uk/eservices/FindAProperty/view/QuickEnquiryInit.do)If i had the budget, i would scrape this site and get all the property deeds in the Reading. This would definately list sensitive information such as who owns the house, when they brought it and their land boundries. Again, this is all public and open data, you are just paying an admin fee to access it.

Again, similar with the Open Register and Companies House data, i put it into an sqlite table and processed it through the geocoding API. The land regsitry gives me an address, the price paid and the date of the transaction. Not the most interesting information and many people know about this data source, but my thought is that its just an easy and accessible source. Its just one of many sources that when used in combination may reveal some interesting insights into how much data is PUBLIC.

By the end of this little mining exercise, i have 112,086 records from companies house, 38,741 records from the open register and 158,225 records from the land regsitry. I tried breifly digging into the twitter API but its changed a lot since i last used it (see my making millions post). I managed to get it working, but the legacy data is behind a paywall and i just done have the budget. If i did, i could get the geodata from a tweet and plot that on my map as well.

So ive used three OPEN and FREE (apart from small admin fees) datasets. If i had the budget or wasnt so moral, I reckon i could definately mine some serious data. I imagine there are plenty of companies/state sponsored activitives that do just this.

Thats enough for this post, in the next post i'll show just what this looks like when its all cross referenced.
