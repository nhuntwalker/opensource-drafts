# Getting Past Issues with CORS

Every once in a while in the lifetime of a web application developer you need to work in a system where the frontend (where the client-facing part of the application) exists entirely separately from the backend (where the data goes for processing).
You may set the two up to talk to each other over [AJAX requests](https://developer.mozilla.org/en-US/docs/Web/Guide/AJAX/Getting_Started), with the frontend requesting some data and the backend providing that data.

Or at least that's how it's supposed to work in your head.

Your backend might look something like...

```python
from flask import Flask, Response
from datetime import datetime
import json

app = Flask(__name__)

@app.route("/api/v1/donuts/<id>")
def hello(id: int) -> Response:
    body = {
        "name": "jelly donut",
        "qty": 22,
        "timestamp": datetime.utcnow().strftime("%b %d, %Y %H:%M:%S")
    }
    headers = {"Content-type": "application/json"}
    return Response(json.dumps(body), headers=headers)
```

It's in [Flask](http://flask.pocoo.org/) because it's quick and easy to get something going without much overhead.

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

@app.route("/api/v1/donuts/<id>")
def hello(id: int) -> Response:
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