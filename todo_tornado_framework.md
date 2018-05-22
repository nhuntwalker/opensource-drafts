# A Web App in Four Frameworks - Tornado

Thus far we've managed to cover the [Pyramid](https://opensource.com/article/18/5/pyramid-framework) and [Flask](https://opensource.com/article/18/4/flask) web frameworks.
We've built the same app twice, and we've seen the similarities and differences between a complete DIY framework and a framework with a few more batteries included.

Now let's look at something somewhat different: [the Tornado framework](https://tornado.readthedocs.io/en/stable/).
Tornado is, for the most part, as bare bones as Flask with a major difference--Tornado is built specifically to handle asynchronous processes.
That special sauce isn't _**terribly**_ useful in this particular context, but we'll see a place where we can find a use for it and how it works in a more general situation.

Let's continue in the habit of what we've done thus far and first tackle the setup and config.

## Tornado - Intro, Startup, and Configuration

We've set up two installable Python distributions thus far, so what we do first shouldn't come as much of a surprise.

```
$ mkdir tornado_todo
$ cd tornado_todo
$ pipenv install --python 3.6
$ pipenv shell
(tornado-someHash) $ pipenv install tornado
```

Create a `setup.py` for installing our application:

```
(tornado-someHash) $ touch setup.py
```

```python
# setup.py
from setuptools import setup, find_packages

requires = [
    'tornado',
    'tornado-sqlalchemy',
    'psycopg2',
]

setup(
    name='tornado_todo',
    version='0.0',
    description='A To-Do List built with Tornado',
    author='<Your name>',
    author_email='<Your email>',
    keywords='web tornado',
    packages=find_packages(),
    install_requires=requires,
    entry_points={
        'console_scripts': [
            'serve_app = todo:main',
        ],
    },
)
```

Since Tornado doesn't require any external configuration we can dive right in to writing the Python code that'll run our application.
Let's make our inner `todo` directory and fill it with the first few files we'll need.

```
todo/
    __init__.py
    models.py
    views.py
```

Like Flask and Pyramid, Tornado has some central configuration which will go in `__init__.py`.
From `tornado.web`, we'll be importing the `Application` object.
This will handle the hookups for routing and views, for including our database (when we get there), and any extra settings that are needed to run our Tornado app.

```python
# __init__.py
from tornado.web import Application

def main():
    """Construct and serve the tornado application."""
    app = Application()
```

Like Flask, Tornado is a mostly DIY framework.
While building this app we have to set up the application instance ourselves.
Because Tornado serves the application with its own HTTP server, we also have to set up how this application gets served.
First, we define a port to listen on with `tornado.options.define`.
Then we'll instantiate Tornado's `HTTPServer`, passing the instance of the `Application` object as its argument.

```python
# __init__.py
from tornado.httpserver import HTTPServer
from tornado.options import define, options
from tornado.web import Application

define('port', default=8888, help='port to listen on')

def main():
    """Construct and serve the tornado application."""
    app = Application()
    http_server = HTTPServer(app)
    http_server.listen(options.port)
```

When we use the `define` function, what we end up doing is creating attributes on the `options` object.
Anything that goes in the position of the first argument will be the attribute's name, and what's assigned to the `default` keyword argument will be the value of that attribute.
As an example, if we named the attribute `potato` instead of `port`, then we can access its value via `options.potato`.

Calling `listen` on the `HTTPServer` doesn't start the server yet.
We must do one more step to have a working application that can listen for requests and return responses.
We need an input-output loop.
Thankfully, Tornado comes with that out of the box in the form of `tornado.ioloop.IOLoop`.

```python
# __init__.py
from tornado.httpserver import HTTPServer
from tornado.ioloop import IOLoop
from tornado.options import define, options
from tornado.web import Application

define('port', default=8888, help='port to listen on')

def main():
    """Construct and serve the tornado application."""
    app = Application()
    http_server = HTTPServer(app)
    http_server.listen(options.port)
    print('Listening on http://localhost:%d' % options.port)
    IOLoop.current().start()
```

I personally like a `print` statement of some kind somewhere that tells me when I'm serving my application, but that's me.
You could do without the `print` line if you so chose.

We begin our IOLoop with `IOLoop.current().start()`.
Let's talk a little more about input, output, and asynchronicity.

### A Detour into the Basics of Async in Python and the I/O Loop

Allow me to preface my words by saying that I am absolutely, positively, surely and securely not an expert in asynchronous programming.
As with all things that I write, what follows stems from the limits of my understanding of the concept.
As I am human, it may be deeply, deeply flawed.

The main concerns of an asynchronous program are:

- how is data coming in?
- how is data going out?
- when can some procedure be left to run without consuming my full attention?

Due to the [Global Interpreter Lock](https://realpython.com/python-gil/), Python is--by design--a [single-threaded](https://en.wikipedia.org/wiki/Thread_(computing)) language.
For every task that a Python program must execute, the full attention of its thread of execution is on that task for the duration of that task.
Our HTTP server is written in Python.
Thus, when data (i.e. an HTTP request) is received the server's SOLE FOCUS is that incoming data.
This means that in most cases whatever procedures need to run in handling and processing that data will completely consume the thread of execution of your server, blocking other potential data from being received, until your server finishes whatever it needed to do.

In many cases this isn't too problematic; a typical web request-response cycle will take only fractions of a second.
For a low- to intermediate-traffic site, a fraction of a second isn't that big of a deal, and you can use multiple deployed instances along with a load-balancer like [nginx](https://www.nginx.com/) to distribute traffic for the larger request loads.

However, what if your average response time takes more than a fraction of a second?
What if you use data from the incoming request to start some long-running process like a machine learning algorithm or some massive database query?
Now, your single-threaded web server starts to accumulate a backlog of requests, some of which will get dropped due to simply timing out.
This is not an option, especially if you want your service to be seen as reliable on a regular basis.

In comes the asynchronous Python program.
It's important to keep in mind that because it's still written in Python, the program is still a single-threaded process.
Anything that would block execution in a synchronous program, unless specifically flagged, will still block execution in an asynchronous one.

However, when structured correctly your asynchronous Python program can "shelve" long-running tasks whenever you designate that a certain function should have the ability to do so.
Your async controller can then be alerted when the shelved tasks are complete and ready to resume, managing their execution only when needed without completely blocking the handling of new input.
That was somewhat jargony, so let's demonstrate with a human example.

I often find myself trying to get multiple chores done at home with little time to do them.
On a given day, that backlog of chores may look like so:

- cook a meal (20 mins prep, 40 mins cook)
- wash dishes (60 mins)
- wash and dry laundry (30 mins wash, 90 mins dry per load)
- vacuum floors (30 mins)

If I were acting as a traditional synchronous program, then I'd be doing each task myself, by hand.
Each task would require my full attention to complete before I could consider handling anything else, as without my active attention nothing would get done.
So my sequence of execution might look like:

1. fully focus on preparing and cooking the meal, including waiting around for food to just...cook (60 mins elapsed)
2. transfer dirty dishes to sink (65 mins elapsed)
3. wash all of the dishes (125 mins elapsed)
4. start laundry with my full focus on that, including waiting around for the washing machine to finish, then transferring to the dryer and waiting for the dryer to finish (250 mins elapsed)
5. vacuum the floors (280 mins elapsed)

That's 4 hours and 40 minutes to complete my chores from end-to-end.

Instead of working hard, I should work smart like an asynchronous program.
My home is full of machines that can do my work for me without my continued effort.
Meanwhile, I can switch my attention to what may actively need it **right now**.
My sequence of execution might instead look like:

1. load clothes into and start the washing machine (5 mins)
2. while the washing machine is running, prep food (25 mins elapsed)
3. after prepping food, start cooking food (30 mins elapsed)
4. while the food is cooking, move clothes from the washing machine into the dryer & start dryer (35 mins elapsed)
5. while dryer is running and food is still cooking, vacuum the floors
6. after vacuuming the floors (65 mins elapsed), take food off the stove and load the dishwasher (70 mins elaped)
7. run the dishwasher (130 mins when done)

And now I'm down to 2 hours and 10 minutes.
Even if I allow for more time for switching between jobs (10-20 more minutes total), I'm still down to about half the time I would've spent if I'd waited to perform each task in sequential order.
This is the power of structuring your program to be asynchronous.

So where does the I/O loop come in?
The way that an asynchronous Python program works is by taking in data from some external source (input) and, should the process require it, offloading that data to some external worker (output) for processing.
When that external process later finishes, the main Python program is alerted.
The program then picks up the result of that external processing (input), and continues on its merry way.

Whenever that data isn't actively in the hands of the main Python program, that main program is freed to work on just about anything else.
This includes awaiting completely new inputs (e.g. HTTP requests) and handling the results of long-running processes (e.g. results of Machine Learning algorithms, long running database queries).
The main program, while still single-threaded, becomes event-driven, triggered into action for specific occurrences handled by the program.
The main worker that listens for those events and dictates how they should be handled is the I/O loop.

We travelled a long road to get to this nugget of an explanation, I know, but what I'm hoping to motivate here is that it's not magic, nor is it some type of complex parallel processing or multithreaded work.
The GIL is still in place; any long-running process within the main program will still block anything else from happening.
The program is also still single-threaded.
However, by externalizing tedious work we conserve the attention of that thread toward only that which it needs to be attentive to.

This is kind of like my asynchronous chores above.
When my attention is fully-necessary for prepping food, that's all I'm doing.
However, when I can get the stove to do work for me by cooking my food, and the dishwasher to wash my dishes, and the washing machine/dryer to handle my laundry, my attention is freed to work on other things.
When I am alerted that one of my long-running tasks is finished and is ready to be handled once again, if my attention is free I can pick up the results of that task and do whatever needs to be done with it next.

## Tornado - Routes and Views

Despite having gone through all the trouble of talking about async in Python, we're going to hold off on using it for a bit and first write ourselves a basic Tornado view.

Unlike the _**function-based views**_ we've seen in the Flask and Pyramid implementations, Tornado's views are all **class-based**.
What this means for us is that we'll no longer use individual, stand-alone functions to dictate how requests are handled.
Instead, the incoming HTTP request will be caught and assigned to be an attribute of our defined class.
Its methods will then handle the corresponding request types.

Let's start with a basic view that prints "Hello World" to the screen.
Every class-based view that we construct for our Tornado app **must** inherit from the `RequestHandler` object found in `tornado.web`.
This will set up all the ground-level logic that we'll need (but don't want to write) to take in a request and construct a properly-formatted HTTP response.

```python
from tornado.web import RequestHandler

class HelloWorld(RequestHandler):
    """Print 'Hello, world!' as the response body."""

    def get(self):
        """Handle a GET request for saying Hello World!."""
        self.write("Hello, world!")
```

Because we're looking to handle a `GET` request, we declare (really override) the `get` method.
Instead of returning anything, we provide text or a JSON-serializable object to be written to the response body with `self.write`.
After that, we let the `RequestHandler` take on the rest of the work that must be done before a response can be sent.

As it stands, this view has no actual connection to the Tornado application itself.
We have to go back into `__init__.py` and update the `main` function a bit.
Here's the new hotness:

```python
# __init__.py
from tornado.httpserver import HTTPServer
from tornado.ioloop import IOLoop
from tornado.options import define, options
from tornado.web import Application
from todo.views import HelloWorld

define('port', default=8888, help='port to listen on')

def main():
    """Construct and serve the tornado application."""
    app = Application([
        ('/', HelloWorld)
    ])
    http_server = HTTPServer(app)
    http_server.listen(options.port)
    print('Listening on http://localhost:%d' % options.port)
    IOLoop.current().start()
```

What'd we do?
We imported the `HelloWorld` view from the `views.py` file into `__init__.py` at the top of the script.
Then we added a list of route-view pairs as the first argument to the instantiation to `Application`.
Whenever we want to declare a route in our application, it **MUST** be tied to a view.
You could use the same view for multiple routes if you so chose, but there must always be a view for every route.

We can make sure that this all works by running our app with the `serve_app` command we enabled in the `setup.py`.
Check `http://localhost:8888/` and see that it says "Hello, world!"

Of course there's more we can and will do in this space, but let's leave it here for now and move on to models.

## Tornado - Connecting the Database

As we've seen thus far, if we want to hold on to data we need to connect a database.
Like with Flask, we'll be using a framework-specific variant of SQLAlchemy called [tornado-sqlalchemy](https://tornado-sqlalchemy.readthedocs.io/en/latest/).

Why use this instead of just the bare [SQLAlchemy](https://www.sqlalchemy.org/)?
Well, `tornado-sqlalchemy` has all the goodness of straightforward SQLAlchemy, so we can still declare models with a common `Base` as well as use all the column data types and relationships to which we've grown accustomed.
Alongside what we already know from habit, `tornado-sqlalchemy` provides an accessible async pattern for its database-querying functionality specifically to work with Tornado's existing I/O loop.

We set the stage by adding `tornado-sqlalchemy` and `psycopg2` to `setup.py` to the list of required packages and reinstall the package.
In `models.py` we declare our models.
This step looks pretty much exactly like what we've already seen in Flask and Pyramid, so I'm going to skip the full class declarations and just put up the necessaries of the `Task` model.

```python
# this is not the complete models.py, but enough to see the differences
from tornado_sqlalchemy import declarative_base

Base = declarative_base

class Task(Base):
    # and so on, because literally everything's the same...
```

We still have to connect `tornado-sqlalchemy` to the actual application.
In `__init__.py` we'll be defining the database and integrating it into the application.

```python
# __init__.py
from tornado.httpserver import HTTPServer
from tornado.ioloop import IOLoop
from tornado.options import define, options
from tornado.web import Application
from todo.views import HelloWorld

# add these
import os
from tornado_sqlalchemy import make_session_factory

define('port', default=8888, help='port to listen on')
factory = make_session_factory(os.environ.get('DATABASE_URL', ''))

def main():
    """Construct and serve the tornado application."""
    app = Application([
        ('/', HelloWorld)
    ],
        session_factory=factory
    )
    http_server = HTTPServer(app)
    http_server.listen(options.port)
    print('Listening on http://localhost:%d' % options.port)
    IOLoop.current().start()
```

Much like the session factory we passed around in Pyramid, we can use `make_session_factory` to take in a database URL and produce an object whose sole purpose is to provide connections to the database for our views.
We then tie it in to our application by passing the newly-created `factory` into the `Application` object with the `session_factory` keyword argument.

Finally, initializing and managing the database will look exactly the same as it had for Flask and Pyramid (i.e. separate db management script, working with respect to the `Base` object, etc.).
It'll look so similar that I'm not going to reproduce it here.

## Tornado - Views Revisited

Hello World is always nice for instruction on the basics but we need some real, application-specific views.

Let's start with the info view.

```python
# views.py
import json
from tornado.web import RequestHandler

class InfoView(RequestHandler):
    """Only allow GET requests."""
    SUPPORTED_METHODS = ["GET"]

    def set_default_headers(self):
        """Set the default response header to be JSON."""
        self.set_header("Content-Type", 'application/json; charset="utf-8"')

    def get(self):
        """List of routes for this API."""
        routes = {
            'info': 'GET /api/v1',
            'register': 'POST /api/v1/accounts',
            'single profile detail': 'GET /api/v1/accounts/<username>',
            'edit profile': 'PUT /api/v1/accounts/<username>',
            'delete profile': 'DELETE /api/v1/accounts/<username>',
            'login': 'POST /api/v1/accounts/login',
            'logout': 'GET /api/v1/accounts/logout',
            "user's tasks": 'GET /api/v1/accounts/<username>/tasks',
            "create task": 'POST /api/v1/accounts/<username>/tasks',
            "task detail": 'GET /api/v1/accounts/<username>/tasks/<id>',
            "task update": 'PUT /api/v1/accounts/<username>/tasks/<id>',
            "delete task": 'DELETE /api/v1/accounts/<username>/tasks/<id>'
        }
        self.write(json.dumps(routes))
```

So what changed? Let's go from the top-down.

The `SUPPORTED_METHODS` class attribute was added.
This will be an iterable of only the request methods that are accepted by this view.
Any other method will return a 405 status code.
When we made the `HelloWorld` view we didn't specify this, mostly out of laziness.
Without this class attribute, this view would respond to any request trying to access the route that this view is tied to.

The `set_default_headers` method is declared, which...sets....the.....default......headers of the outgoing HTTP response.
We declare this here to ensure that any response we send back has a `"Content-Type"` of `"application/json"`.

We added `json.dumps(some_object)` to the argument of `self.write` because it makes it easy to construct the content for the body of the outgoing response.

So that's done, and we can go ahead and connect that to the home route in `__init__.py`.

```python
# __init__.py
from tornado.httpserver import HTTPServer
from tornado.ioloop import IOLoop
from tornado.options import define, options
from tornado.web import Application
from todo.views import InfoView

# add these
import os
from tornado_sqlalchemy import make_session_factory

define('port', default=8888, help='port to listen on')
factory = make_session_factory(os.environ.get('DATABASE_URL', ''))

def main():
    """Construct and serve the tornado application."""
    app = Application([
        ('/', InfoView)
    ],
        session_factory=factory
    )
    http_server = HTTPServer(app)
    http_server.listen(options.port)
    print('Listening on http://localhost:%d' % options.port)
    IOLoop.current().start()
```

As we know, there will be more views and routes that need writing.
Each one of them will get dropped into the `Application` route listing as needed.

Each one will also need a `set_default_headers` method.
On top of that we'll create our own `send_response` method, whose job it will be to package our response along with any custom status codes we want to set for a given response.
Since each one will need both of these, we can create a base class that each of our views can inherit from that contains these methods.
That way, we only have to write them once.

```python
# views.py
import json
from tornado.web import RequestHandler

class BaseView(RequestHandler):
    """Base view for this application."""

    def set_default_headers(self):
        """Set the default response header to be JSON."""
        self.set_header("Content-Type", 'application/json; charset="utf-8"')

    def send_response(self, data, status=200):
        """Construct and send a JSON response with appropriate status code."""
        self.set_status(status)
        self.write(json.dumps(data))
```

For a view like the `TaskListView` we'll soon write, we'll also need a connection to the database.
We'll need `tornado_sqlalchemy`'s `SessionMixin` to add a database session within every view class.
We can fold that into the `BaseView` so that by default every view inheriting from it has access to a database session.

```python
# views.py
import json
from tornado_sqlalchemy import SessionMixin
from tornado.web import RequestHandler

class BaseView(RequestHandler, SessionMixin):
    """Base view for this application."""

    def set_default_headers(self):
        """Set the default response header to be JSON."""
        self.set_header("Content-Type", 'application/json; charset="utf-8"')

    def send_response(self, data, status=200):
        """Construct and send a JSON response with appropriate status code."""
        self.set_status(status)
        self.write(json.dumps(data))
```

As long as we're modifying this `BaseView` object, we should address a quirk that will come up when we consider data being posted to this API.

When Tornado (as of v. 4.5) consumes data from a client and organizes it for use in the application, it keeps all of the incoming data as bytestrings.
However, all the code written here is assuming Python 3, so the only strings that we want to work with are unicode strings.
We can add another method to this `BaseView` class whose job it will be to convert the incoming data to unicode prior to it being used anywhere else in the view.

If we want to convert this data before we ever have to use it in a proper view method, we can override the view class's native `prepare` method.
Its job is to run before the view method itself actually runs.
If we override the `prepare` method, we can set some logic to run that'll do the bytestring -> unicode conversion whenever a request is received.

```python
# views.py
import json
from tornado_sqlalchemy import SessionMixin
from tornado.web import RequestHandler

class BaseView(RequestHandler, SessionMixin):
    """Base view for this application."""

    def prepare(self):
        self.form_data = {
            key: [val.decode('utf8') for val in val_list]
            for key, val_list in self.request.arguments.items()
        }

    def set_default_headers(self):
        """Set the default response header to be JSON."""
        self.set_header("Content-Type", 'application/json; charset="utf-8"')

    def send_response(self, data, status=200):
        """Construct and send a JSON response with appropriate status code."""
        self.set_status(status)
        self.write(json.dumps(data))
```

If there's any data coming in, it'll be found within the `self.request.arguments` dictionary.
We can access that data by key and convert its contents (always a list) to unicode.
Because this is a class-based view instead of a function-based view, we can store this modified data as an instance attribute to be used later.
I called it `form_data` here, but it can just as easily be called `potato`.
The point is that we can store data that has been submitted to the application.

### Tornado - Asynchronous View Methods

Now that we've built our `BaseView` we can build the `TaskListView` that will inherit from it.

As you can probably tell from the section heading, this is where all that talk about asynchronicity comes in.
The `TaskListView` will handle `GET` requests for returning a list of tasks, and `POST` requests for creating new tasks given some form data.
Let's first look at the code to handle the `GET` request.

```python
# all the previous imports
import datetime
from tornado.gen import coroutine
from tornado_sqlalchemy import as_future
from todo.models import Profile, Task

# the BaseView is above here
class TaskListView(BaseView):
    """View for reading and adding new tasks."""
    SUPPORTED_METHODS = ("GET", "POST",)

    @coroutine
    def get(self, username):
        """Get all tasks for an existing user."""
        with self.make_session() as session:
            profile = yield as_future(session.query(Profile).filter(Profile.username == username).first)
            if profile:
                tasks = [task.to_dict() for task in profile.tasks]
                self.send_response({
                    'username': profile.username,
                    'tasks': tasks
                })
```

The first major piece here is the `@coroutine` decorator, imported from `tornado.gen`.
Any Python callable that has a portion that acts out-of-sync with the normal flow of the call stack is effectively a "co-routine"; a routine that can run alongside other routines.
In the example of my household chores, pretty much every chore was itself a co-routine.
Some were blocking routines (e.g. vacuuming the floor) but that routine simply blocked my ability to start or attend to anything else.
It didn't block any of the other routines that had already been set in motion from continuing.

Tornado provides for a number of ways to build an app that takes advantage of co-routines, including allowing us to set locks on function calls, conditions for synchronizing asynchronous routines, and a system for manually modifying the events that control the I/O loop.

_**The only way**_ that the `@coroutine` decorator is being used here is to allow the `get` method to farm out the SQL query as a background process and resume once the query is complete, while not blocking the Tornado I/O loop from handling other sources of incoming data.
That is all that's "asynchronous" about this implementation; out-of-band database queries.
Clearly if we wanted to showcase the magic and wonder of an async web app, a To Do list isn't the way.

But hey, that's what we're building so let's see how our method takes advantage of that `@coroutine` decorator.
The `SessionMixin` that was...mixed-in...to the `BaseView` declaration added two handy, database-aware attributes to our view class: `session` and `make_session`.
They're similarly named, and at the end of the day accomplish fairly similar goals.

The `self.session` attribute is a session with an eye on the database.
At the end of the requeset-response cycle, just before the view sends a response back to the client, any changes that have been made to the database are committed, and the session is closed.

`self.make_session`, is a context manager and generator, building and returning a brand new session object on the fly.
That first `self.session` object still exists; `make_session` creates a new one anyway.
The `make_session` generator also has baked into itself the logic for committing and closing the session it creates as soon as its context (i.e. indentation level) ends.

If you inspect the source code, there is **no difference** between the type of object assigned to `self.session` and the type of object generated by `self.make_session`.
The difference is in how they're managed.

With the `make_session` context manager, the generated session belongs only to the context, beginning and ending within that context.
You can open, modify, commit, and close multiple database sessions within the same view with the `make_session` context manager.

`self.session` is much simpler, with the session having already been opened by the time you get to your view method, and committing before the response is sent back to the client.

Despite [the readthedocs snippet](https://tornado-sqlalchemy.readthedocs.io/en/latest/#usage) and the [the pypi example](https://pypi.org/project/tornado-sqlalchemy/#description) both specifying the use of the context manager, there's nothing about either the `self.session` object or the `session` generated by `self.make_session` that is inherently asynchronous.
The point where we start thinking about the async behavior built into `tornado-sqlalchemy` comes when we initiate a query.

The `tornado-sqlalchemy` package provides us with the `as_future` function.
The job of `as_future` is to wrap the query constructed by the `tornado-sqlalchemy` session and yield its return value.
If the view method is decorated with `@coroutine`, then using this `yield as_future(query)` pattern will now make your wrapped query an asynchronous background process.
The I/O loop takes over, awaiting the return value of the query and the resolution of the `future` object created by `as_future`.

To have access to the result from `as_future(query)`, you must `yield` from it.
Otherwise, you only get an unresolved generator object and can do nothing with the query.

Everything else in this view method is pretty much par for the course, mirroring what we've already seen in Flask and Pyramid.

The `post` method will look fairly similar.
For the sake of consistency, let's see how the `post` method looks and how it handles the `self.form_data` that was constructed with the `BaseView`.

```python
@coroutine
def post(self, username):
    """Create a new task."""
    with self.make_session() as session:
        profile = yield as_future(session.query(Profile).filter(Profile.username == username).first)
        if profile:
            due_date = self.form_data['due_date'][0]
            task = Task(
                name=self.form_data['name'][0],
                note=self.form_data['note'][0],
                creation_date=datetime.now(),
                due_date=datetime.strptime(due_date, '%d/%m/%Y %H:%M:%S') if due_date else None,
                completed=self.form_data['completed'][0],
                profile_id=profile.id,
                profile=profile
            )
            session.add(task)
            self.send_response({'msg': 'posted'}, status=201)
```

As I said, it's about what we'd expect:

- The same query pattern as we saw with the `get` method.
- The construction of an instance of a new `Task` object, populated with data from `form_data`
- The adding (but not committing because it's handled by the context manager!) of the new `Task` object to the database session
- The sending of a response back to the client

And thus we have the basis for our Tornado web app.
Everything else (e.g. database management and more views for a more complete app) is effectively the same as what we've already seen in the Flask and Pyramid apps.

## Thoughts About Using the Right Tool for the Right Job

What we're starting to see as we continue to move through these web frameworks is that they can all effectively handle the same problems.
For something like this To Do list, any framework can do the job.
However, some web frameworks are more appropriate for certain jobs than other ones, depending on what "more appropriate" means for you and your needs.

While Tornado is clearly capable of handling the same job that Pyramid or Flask can handle, to use it like we have here is effectively a waste.
It's like using a car to travel one block from home.
Yes it can do the job of "travel", but short trips aren't really why you choose to use a car over a bike or just your feet.

Per the documentation, Tornado is billed as "a Python web framework and asynchronous networking library".
There are few like it in the Python web framework ecosystem.
If the job you're trying to accomplish requires (or would benefit significantly from) asynchronicity in any way, shape, or form, use Tornado.
If your application needs to handle multiple long-lived connections while not sacrificing much in performance, choose Tornado.
If your application is many applications in one and needs to be thread-aware for the accurate handling of data, reach for Tornado.
That's where it works best.
Use your car to do "car things".
Use other modes of transport to do everything else.

## Going Forward and a Little Perspective Check

Speaking of using the right tool for the right job, keep in mind the scope and scale, both present and future, of your application when choosing your framework.
Up to this point we've only really looked at frameworks meant for small to medium-sized web applications.
The next and final installment of this series will cover one of the most popular Python frameworks, Django, meant for big applications that might grow bigger.
Again, while it technically can and will handle the "To Do List" problem, keep in mind that it's not really what the framework is for.
We'll still put it through its paces to show how an application can be built with it, but we have to keep in mind the intent of the framework and how that's reflected in its architecture:

- Flask: meant for small, simple projects; makes it easy for us to construct views and connect them to routes quickly; can be encapsulated in a single file without much fuss
- Pyramid: meant for projects that may grow; contains a fair bit of configuration to get up and running, and separate realms of application components can easily be divided and built out to arbitrary depth without losing sight of the central application
- Tornado: meant for projects benefitting from precise and deliberate I/O control; allows for coroutines and easily exposes methods that can control how requests are received/responses are sent and when those operations occur
- Django (as we'll see): meant for big things that may get bigger; large ecosystem of add-ons and mods; very opinionated in its configuration and management in order to keep all the disparate parts in line

Whether you've been reading since the first post in this series or joined a little later on, thanks for reading!
Please feel free to leave questions or comments, and I'll see you next time with hands full of Django.

## Huge Shoutout to the Python BDFL

I must give credit where credit is due.
MASSIVE thanks are owed to [Guido van Rossum](https://www.twitter.com/gvanrossum) for more than just creating my favorite programming language.

During [Pycascades 2018](https://www.pycascades.com) I was fortunate enough to give the talk that this blog series is based on and be invited to the dinner of the speakers.
It was there that I got to sit next to Guido the whole night and just pepper him with questions.

One of those questions was how in the hell async worked in Python and he, without a bit of fuss, spent time explaining it to me in a way that I could start to grasp the concept.
He later [tweeted to me](https://twitter.com/gvanrossum/status/956186585493458944) a spectacular resource for learning async with Python that I subsequently read 3 times over 3 months, then wrote this post.
You're an awesome guy, Guido!