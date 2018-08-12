# Getting Past Issues with CORS

*Note: the code samples in this post are meant to be runnable, but won't include real URLs. The Python code can probably be run with just a local Flask server (domain: localhost:5000). The Javascript can be run from a console on any website. I used [the Flask website](http://flask.pocoo.org/) when writing the code samples.*

Every once in a while in the lifetime of a web application developer you need to work in a system where the frontend (where the client-facing part of the application) exists entirely separately from the backend (where the data goes for processing).
You may set the two up to talk to each other over [AJAX requests](https://developer.mozilla.org/en-US/docs/Web/Guide/AJAX/Getting_Started), with the frontend requesting/sending some data and the backend providing/receiving that data.

Or at least that's how it's supposed to work in your head.

Your backend might look something like...

```python
from flask import Flask, Response
from datetime import datetime
import json

app = Flask(__name__)

@app.route("/api/v1/donuts/<id>")
def donut_detail(id: int) -> Response:
    """Get the detail for one donut."""
    body = {
        "name": "jelly donut",
        "qty": 22,
        "timestamp": datetime.utcnow().strftime("%b %d, %Y %H:%M:%S")
    }
    headers = {"Content-type": "application/json"}
    return Response(json.dumps(body), headers=headers)
```

It's in [Flask](http://flask.pocoo.org/) because it's quick and easy for me to get something going without much overhead.

So you build out your frontend a bit and test that your backend can send the necessary data to your front end.
You pop open your browser and throw this little ditty in the browser's console.

```javascript
> fetch('http://backend.domain.com/api/v1/donuts/4')
    .then(response => response.json())
    .then(json => console.log(json))
```

You execute the command and see the following come up:

```
Promise {<pending>}
```

```
Failed to load http://backend.domain.com/api/v1/donuts/4: No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'http://frontend.location.org' is therefore not allowed access. If an opaque response serves your needs, set the request's mode to 'no-cors' to fetch the resource with CORS disabled.
```

```
Uncaught (in promise) TypeError: Failed to fetch
```

What gives?
Every time you try to run this command you get the same response back.

## The Problem

You've run into a problem with **CORS**, or **Cross-Origin Resource Sharing**.

Although they may be annoying, rules about CORS exist to protect you, your users, and your application.
They prevent scripts that you may not have expected from making requests for resources from your backend that you didn't intend to serve up.

CORS rules come into effect whenever a request is sent to your backend from a domain, subdomain, port, or protocol that differs from wherever your backend is being served.
In our example above, `http://frontend.location.org` is clearly different from `http://backend.domain.com`.
Therefore, the request that was attempted was going *across origins* to *share resources*.

## A Seemingly-Simple Fix

As with most good error messages, the error text tells you how to handle the error you've received.
When the data was requested and the server received that request, the response was constructed and returned.
But, when the browser saw the response, it noted that it was from a different origin and the appropriate header wasn't sent along with the request.
That header? **Access-Control-Allow-Origin**.

The Access-Control-Allow-Origin header does what its name implies: specifies an origin that's allowed to make requests against the server.
If the origin where the request originated from doesn't match the origin in the server's header (or the header isn't even set), the browser throws an error.

So add in that header!

```python
from flask import Flask, Response
from datetime import datetime
import json

app = Flask(__name__)

@app.route("/api/v1/donuts/<id>", methods=["GET"])
def donut_detail(id: int) -> Response:
    """Get the detail for one donut."""
    body = {
        "name": "jelly donut",
        "qty": 22,
        "timestamp": datetime.utcnow().strftime("%b %d, %Y %H:%M:%S")
    }
    headers = {
        "Content-type": "application/json",
        "Access-Control-Allow-Origin": "http://frontend.location.org"
    }
    return Response(json.dumps(body), headers=headers)
```

Then re-send the request and voila!

```javascript
> fetch('http://backend.domain.com/api/v1/donuts/4')
    .then(response => response.json())
    .then(json => console.log(json))
```

```
Promise {<pending>}

{name: "jelly donut", qty: 22, timestamp: "Aug 07, 2018 05:19:42"}
```

It works!

The `Access-Control-Allow-Origin` header will either take one specific domain (only the protocol and domain name are necessary, including subdomain) or the value `"*"` to mean "any domain is cool!"
If you are providing a specific domain, it must mach exactly the domain of the expected origin, including protocol.

I would highly recommend setting a specific domain, including the right protocol (http vs https).
For the purposes of application security, there should be no reason why just anyone should be able to make requests to your backend unless you're specifically trying to serve data to anyone that might request it.
Maybe you're building an open API for the resources you hold?
Maybe you're serving static content like JavaScript, CSS, images, or video?
If you don't specifically need to serve data to any origin, set a specific origin.

## Requesting with Data

Now your frontend wants to send data to the backend.
Maybe you decided to create a new type of donut, and you want to record this new donut and how many you made.
The type of request that you might be sending to your backend might look like...

```javascript
> fetch('http://backend.domain.com/api/v1/donuts', {
        method: 'POST',
        mode: 'cors',
        headers: {
            'Content-Type': 'application/json; charset=utf-8'
        },
        body: JSON.stringify({
            name: 'plantain donut',
            qty: 40
        })
    })
    .then(response => response.json())
    .then(json => console.log(json))
```

You have, of course, set up your backend with a route to handle POST requests for adding new donuts.
And, keeping what you already encountered with CORS in mind, you make sure that it's including the `Access-Control-Allow-Origin` header in its response.
After all, **on every route that communicates with the client** you need to make sure that the client receives that header allows it access to the data in the response.

That backend route and view might look like so...

```python
# ...other stuff from before...
@app.route("/api/v1/donuts", methods=["POST"])
def new_donut() -> Response:
    """Accept and create a new donut."""
    headers = {
        "Content-type": "application/json",
        "Access-Control-Allow-Origin": "http://frontend.location.org"
    }
    body = json.dumps({"msg":"Congrats on the new donut!"})
    return Response(body, headers=headers)
```

So, if your request from the frontend is successful you would see `{msg: "Congrats on the new donut!"}` written in the browser's console.

You don't see that.
Not at all.

```
Promise {<pending>}
```
```
Failed to load http://backend.domain.com/api/v1/donuts: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'http://frontend.location.org' is therefore not allowed access. If an opaque response serves your needs, set the request's mode to 'no-cors' to fetch the resource with CORS disabled.
```
```
Uncaught (in promise) TypeError: Failed to fetch
```

This looks the same as that first error!
What happened?!
You set up the route and view correctly!
You included the right header with the right value!
What gods must you sacrifice to in order to make a damn request!

## Sending Data with AJAX

Whenever you're trying to send data from the client to the server in any way that isn't directly from a form or a plain text submission, you're going to run into this problem.

If you read the error closely it actually does look a bit different.
The error mentions something called a "preflight" request.
What's actually happened is that when you tried to send off the data, before the `POST` request was packaged and sent, the browser sent a probe to check that the backend can even accept `POST` requests to that route.

That request, **prior to the flight of the POST request**, was an `OPTIONS` request.
What the `OPTIONS` request hopes to get back is effectively a listing of the allowed methods for this route.

    "Hey backend, before I try to even send this data, what am I allowed to even do?" 
    
    "Oh hey Client, you can do GET, POST, and DELETE!"
    
    "Thanks, I've here's a POST with some JSON data."

    "Got it! Your POST has been received!"

In addition to setting that `Access-Control-Allow-Origin` header, you'll need to modify the view handling the `POST` request to handle the `OPTIONS` request.
With Flask, that's a pretty straightforward task.
You can either construct a new view to specifically handle the `OPTIONS` request, or you can add `OPTIONS` to the list of allowed methods for the `new_donut` view.

I like having separate functions for separate purposes, so I'm going to make a second view.
**NOTE: When Flask reads through the route/view combinations in the file it reads from top to bottom and routes the request to the first view that matches.** I'm going to put my "OPTIONS" view above my "POST" view or else it won't work in this implementation.

```python
# ...other stuff from before...
@app.route("/api/v1/donuts", methods=["OPTIONS"])
def new_donut_preflight() -> Response:
    headers = {
        "Content-type": "application/json",
        "Access-Control-Allow-Origin": "http://frontend.location.org",
        "Access-Control-Allow-Methods": "POST, OPTIONS",
    }
    return Response("", headers=headers)
# ...the POST view below here...
```

Woo! Fixed! Send off that request again.

```
Promise {<pending>}
```
```
Failed to load http://backend.domain.com/api/v1/donuts: Request header field Content-Type is not allowed by Access-Control-Allow-Headers in preflight response.
```
```
Uncaught (in promise) TypeError: Failed to fetch
```

Dammit, another error.
A different error though, so progress has been made!

The error is now explicitly telling you what it needs.

```Request header field Content-Type is not allowed by Access-Control-Allow-Headers in preflight response.```

There's a header that could (really, should) be in the response called `Access-Control-Allow-Headers` that specifies what headers the server is willing to accept for what would be the follow-up `POST` request.
The `Content-Type` header, which was included in the request to declare it as incoming JSON, wasn't amongst the allowed headers, so the request was rejected.

That's a simple fix for us though.
Server-side, we can add to the outgoing response that the `Content-Type` header is to be expected on any `POST` request coming in on the `/api/v1/donuts` route.

```python
# ...other stuff from before...
@app.route("/api/v1/donuts", methods=["OPTIONS"])
def new_donut_preflight() -> Response:
    headers = {
        "Content-type": "application/json",
        "Access-Control-Allow-Origin": "http://frontend.location.org",
        "Access-Control-Allow-Methods": "POST, OPTIONS",
        "Access-Control-Allow-Headers": "Content-Type"
    }
    return Response("", headers=headers)
# ...the POST view below here...
```

Sending the request again should show something different.

```
Promise {<pending>}
```

```
{msg: "Congrats on the new donut!"}
```

Hey! The `POST` request went through!

This type of modification is going to be necessary whenever you want to send a POST request with data that isn't coming directly from a form.
It'll also be necessary if you have any custom headers, want to hand cookies back and forth, or want to set a header for caching responses.

While this wasn't an exhaustive look into CORS, hopefully there's enough here to help you knock past the more common roadblocks when handling data going between different origins.
Thanks for reading, and I hope that all your requests are fulfilled (unless they shouldn't be).