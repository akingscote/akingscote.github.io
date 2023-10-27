---
title: "VueJS & Golang Server - reCAPTCHA V3 Validation"
date: "2021-08-14"
categories: 
  - "development"
  - "golang"
  - "security"
  - "web-development"
tags: 
  - "availability"
  - "bot"
  - "golang"
  - "security"
  - "spam"
  - "vuejs"
coverImage: "captchav3.png"
---

I recently volunteered to develop a website for a friend, who wanted the ability to send emails directly from the frontend. On the surface, this is a straightforward request, but the reality is that it means *quite* a lot of work. One of the aspects which was new to me, was having a public facing form which could send an email. Knowing how much spam i get on this site, i figured i'd need to some pretty robust anti-spam mechanisms in place. My first thought was to implement honeypot fields - which are form fields which have styling to hide them from the GUI. Bot's will just automatically attempt to fill in the forms and will populate all fields with their garbage. On the backend, i'd catch whether these honeypot fields have values and return an error if they do.

It sounds good, but i feel that a better (more reliable) method would be to implement reCAPCHA - **C**ompletely **A**utomated **P**ublic **T**uring test to tell **C**omputers and **H**umans **A**part. The problem there is then the UX - i really hate clicking the traffic lights whenever i want to use a website. However... I discovered that with reCAPTCHA v3, you never get interruption with users. It automatically tries to figure out whether the interaction is legitimate or not and returned a score from 0.0 - 1.0. 0.0 is very likely a bot, whereas 1.0 is likely a legitimate human interaction, so its up to you to decide your threshold.

The entire codebase can be found at [https://gitlab.com/akingscote-personal/vue-and-go-captcha](https://gitlab.com/akingscote-personal/vue-and-go-captcha). There are README's within the [website](https://gitlab.com/akingscote-personal/vue-and-go-captcha/-/tree/main/website) and [server](https://gitlab.com/akingscote-personal/vue-and-go-captcha/-/tree/main/server) directories which have some more information.

## Design

This is just a noddy proof of concept, but there is a development method that ive started to use which has made my life a lot easier. Basically, ive started to avoid installing npm packages onto my machine and try to containerise the _development_ process as much as possible.

Outside the relatively complicated development process ive conducted, the design for this is pretty simple. Its just a single page application SPA developed in VueJS, which has a web form. The webform sends a HTTP POST request to a specific backend route. To support the route, ive developed a bespoke web server. When the user submits the form, there is a client side reCATPCHA validation. If successful, there is a backend reCAPTCHA validation. If successful, the POST request completes and a valid response is returned to the user.

## Prerequisites

I'm developing on Ubuntu 20.04, but most of the setup is containerised and none of this work is OS depentant.

You will need, `docker`, `docker-compose`and most importantly reCAPTCHA v3 keys.

Go to [https://www.google.com/recaptcha/admin/create](https://www.google.com/recaptcha/admin/create) , and select `reCAPTCHAv3` and register a valid domain. You can use `localhost` and `127.0.0.1`.

You will get two keys - a **site** key and a **secret** key. The **site** key is a client side key used in the frontend application, so a user can easily see it and snoop it out in any outgoing requests. The **secret** key goes into the server and is hidden from the user.

The **site** key is used to invoke reCAPTCHA service. The **secret** key authorizes communication between the backend and the reCAPTCHA server to [verify the user's response](https://developers.google.com/recaptcha/docs/verify).

![](/images/newcaptcha.png)

## Development Process

The entire codebase is available in gitlab [https://gitlab.com/akingscote-personal/vue-and-go-captcha](https://gitlab.com/akingscote-personal/vue-and-go-captcha). There are The site key is used to invoke reCAPTCHA service. The secret key authorizes communication between the backend and the reCAPTCHA server to verify the user's response.some pretty detailed README's in there, which might explain some of the lower level things which are going on.

The short version is this:

On the development machine, you need to install node, the Vue CLI and bootstrap a basic project.

Install Node & add its bin directory to the path
```
sudo mkdir -p /usr/local/lib/nodejs
wget https://nodejs.org/dist/v14.17.5/node-v14.17.5-linux-x64.tar.xz -O /tmp/node.tar.xz
sudo tar -xJvf /tmp/node.tar.xz -C /usr/local/lib/nodejs
echo "export PATH=$PATH:/usr/local/lib/nodejs/node-v14.17.5-linux-x64/bin" &gt;&gt; ~/.zshrc
source ~/.zshrc
```

Install the Vue CLI
```
# Install globally as sudo user, so that non priv accounts can use it
sudo env PATH=$PATH:/usr/local/lib/nodejs/node-v14.17.5-linux-x64/bin npm install -g @vue/cli
```

The site key is used to invoke reCAPTCHA service. The secret key authorizes communication between the backend and the reCAPTCHA server to verify the user's response.

Bootstrap the project

`vue create hello-world -d`

Now we have a webapp base to work from. I created a two _Dockerfiles_ - `dev.Dockerfile` and `prod.Dockerfile`. The `dev` file is for the development - the VueJS `.vue` files are mounted into the container and the server is spun up in development mode - meaning i can change the files on my local machine and they automatically change in the container. The downside is that the first time i run the container, its a bit slow as its installing all the dependencies in the `package.json`, but it means i dont clutter up my local machine.

Additionally, if i want additional packages, i need to install them into the container, then copy ouThe site key is used to invoke reCAPTCHA service. The secret key authorizes communication between the backend and the reCAPTCHA server to verify the user's response.t the `package.json` file back into my local machine. I find its a small price to pay though. The `prod` build dosent run a development server, but builds for production. You can then `docker cp` the built `dist` directory onto the local machine - ready to be served by the golang webserver.

For the webapp development, i used `docker-compose`so i dont have to remember a dozen different commands.

For the server side, ive created a `Dockerfile` so you can build the binary without actually having `go` installed. But the development was done the old school way - by building it on my machine. The server is a Golang app. Im a big fan of the golang [cobra](https://github.com/spf13/cobra) CLI package.  I also loosely conformed to the [standards](https://github.com/golang-standards/project-layout) for project layout. Built as a go module as its just dreamy.

## Frontend

Im not going to detail how to built a form and submit a POST request in VueJS. You can look at the code [here](https://gitlab.com/akingscote-personal/vue-and-go-captcha/-/tree/main/website).

Using the `axios` library, I'm can sending a POST request to the server. Before the POST is sent, i'm using the [vue-recaptcha-v3](https://www.npmjs.com/package/vue-recaptcha-v3) package to connect to Google & invoke the reCAPTCHA service. The **site** key is used in this request and presumably the requesting hostname is checked by google. Returned in the invocation response is a token.

This token is then sent to the server in the POST request. I couldnt see if there was a defined process for this, so i added it in a custom header which ive called `X-Recaptcha-Token`. This token value isn't really sensitive and is verified on the backend.

```
sync recaptcha() {
    // (optional) Wait until recaptcha has been loaded.
    await this.$recaptchaLoaded()
 
    // Execute reCAPTCHA with action "login".
    const token = await this.$recaptcha('login')
 
    if (token != "") {
        // Do stuff with the received token.
        this.sendForm(token)
    } else {
        alert("Captcha Failed, please try again")
        console.log("Token not legit")
    }
 
}
```

This function is called on form submission, if the token is present, the `sendForm` function will send the HTTP POST to the backend, with the token in the headers.

```
endForm(token) {
 
    let formData = new FormData();
 
    //validation here
 
    formData.append("name", this.name);
    formData.append("email", this.email);
    formData.append("message", this.message);
 
    const headers = {
        "Content-Type": "multipart/form-data",
        "X-Recaptcha-Token": token
    };
 
    let host = document.location.hostname;
    let endpoint = "http://" + host + "/myendpoint"
 
    axios({
        method: "post", 
        url : endpoint,
        data: formData,
        headers: headers
    }
    ).then(() => {
        this.name = "";
        this.email = "";
        this.message = "";
        alert("Thank you for your message");
    })
    .catch((error) => {
 
        if (!error.response) {
            alert("Cannot connect to endpoint")
            console.error(error)
        } else {
            // http status code
            const code = error.response.status
            // response data
            const response = error.response.data
            if (code == 412) {
                console.error(error.response.data);
                let err = "Error submitting form - " + error.response.data
                alert(err);
            } else {
                console.log(error)
                alert("Error submitting form")
            }
        }
    });
},
```

Notice that i'm catching `412` responses on the frontend...

##  Backend

The webserver is a go cli application using the cobra package. You can use the `--help` flag to see which options are available. Once the VueJS website is built, you can point the server to the `dist` directory.

I normally `docker cp` from the `pod.Dockerfile` container, to wherever the webserver is running from.

You dont need golang installed to build the binary, you can do that with docker.

```
DOCKER_BUILDKIT=1 docker build -o bin .
```

That will output the webserver binary in the `bin` directory. The `Dockerfile` will build it for you and has all the prereqs required.

`./vue-and-go-captcha --websiteDir ../website/dist --port 80`

```
$ ./vue-and-go-captcha --help
Run the websever which hosts a VueJS application and server side
    captcha validation
 
Usage:
  webserver [flags]
 
Flags:
  -h, --help                help for webserver
      --logLevel string     Log Level (debug, info, warn, error) (default "info")
      --port int            port to listen on (default 8080)
      --websiteDir string   Full path to website files (default "./static")
```

When the POST request is sent, a _preflight_ HTTP OPTIONS request is sent to the server before the POST.

A [preflight](https://developer.mozilla.org/en-US/docs/Glossary/Preflight_request) request is typically sent by web browsers to see if CORS is supported. The golang server lazily handles this requests on the `myendpoint` endpoint.

```
func myEndpoint(w http.ResponseWriter, r *http.Request) {
    log.Debugln("Endpoint request...")
    // check for preflight
    if r.Method == "OPTIONS" {
        log.Infoln("OPTIONS request received, likely preflight request...")
        w.Header().Set("Content-Type", "application/json")
            w.Header().Set("Access-Control-Allow-Origin", "*")
            w.Header().Set("Access-Control-Allow-Methods", "POST, GET, OPTIONS, PUT, DELETE")
            w.Header().Set("Access-Control-Allow-Headers", "*")
        return
    }
}
```

Which essentially means, just allow everything here. Its not something you'd likely allow in production.

Once the sneaky options request is "_handled",_ I look for the `X-Recaptcha-Token` header in the request, if its not present I return a HTTP `412` to the client. `412` means [Precondition failed](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/412), which seems like a sensible response to clients which do not contain the data i expect. This `412` is caught on the client side and an alert it presented to the user saying "captcha validation failed".

If the token is present, i then actually perform the **secret** backend captcha validation...

The process is actually very simple, its just another POST request, but this time from the server directly to Googles captcha verification service. I need the token from the frontend invocation, which is referred to as the "captcha_response". The secret and the frontend token are sent to `www.google.com/recaptcha/api/siteverify` and if its all ok, a result is returned from `0.0` to `1.0`.

```
type CaptchaResponse struct {
    Success     bool      `json:"success"`
    ChallengeTs time.Time `json:"challenge_ts"`
    Hostname    string    `json:"hostname"`
    Score       float64   `json:"score"`
    Action      string    `json:"action"`
}
 
secret := "xxxxxxxxxxxxxxxxxxxxxxxxxxx"
 
//one-line post request/response...
response, err := http.PostForm("https://www.google.com/recaptcha/api/siteverify", url.Values{
    "secret": {secret},
    "response": {captcha_response}})
 
if err != nil {
    log.Errorln("error verifying captcha", err)
}
 
defer response.Body.Close()
body, err := ioutil.ReadAll(response.Body)
 
if err != nil {
    log.Errorln(err)
}
 
var capResponse CaptchaResponse
err = json.Unmarshal(body, &capResponse)
if err != nil {
    log.Errorln("error:", err)
}
log.Debugln(capResponse)
 
captchaScore := 0.75
 
if capResponse.Score < captchaScore {
    log.Errorln("Captcha score less than", captchaScore, capResponse.Score)
}
 
if capResponse.Success == true {
    log.Infoln("Captcha validation successful - ", capResponse.Score)
}
```

Ive set the threshold pretty loose, at 0.75. There is also a `Success` field which i check is set to `true`.

If all is good, the POST request is processed as usual. If there is ever a problem with the reCAPTCHA, a 412 is returned.

## End Result

The end result is a basic SPA VueJS app, which is build for production and hosted in a go web server. The server has a custom endpoint called `myendpoint`, which receives an incoming POST request. reCAPTCHA v3 is invoked from the frontend and a the response is sent to the `myendpoint` endpoint in the server. If everything is gravy, the request is processed and returned to the user. Otherwise, a 412 is returned and the user is alerted that the captcha validation has failed.

Download and watch [this](/assets/video/captcha-working.webm) video to see a full demonstration.

[video width="1478" height="858" webm="https://akingscote.co.uk/wp-content/uploads/2021/08/captcha-working.webm"][/video]

It was an interesting thing to play with. I wasnt sure whether you _needed_ a custom backend to work with captcha, but it turns out you do.
