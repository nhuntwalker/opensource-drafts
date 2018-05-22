# A Web App in Four Frameworks - Pyramid

In my last post I introduced the idea of a web application built in four different Python web frameworks in order to show how to get started in each and showcase the differences between them.
I then went on to build the basics of a To Do list in the [Flask](http://flask.pocoo.org/) web framework.

Let's continue onto the next web framework and dive more into the differences between them where they exist.
Let's get into it with [Pyramid](https://trypyramid.com/).

## Pyramid - Intro, Startup, and Configuration

Self-described as "the start small, finish big, stay finished framework", the Pyramid web framework is much like Flask in that it takes very little get up and running.
In fact, you'll recognize many of the same patterns as w build out this application.
The major difference between the two, however, is that Pyramid comes with several useful utilities included, which we'll see in a minute.
Before we get to that point though, let's create a virtual environment and install the package.

```
$ mkdir pyramid_todo
$ cd pyramid_todo
$ pipenv install --python 3.6
$ pipenv shell
(pyramid-someHash) $ pipenv install pyramid
```

Much like we did with Flask, we should create for ourselves a `setup.py` file to make what we're about to build into an easily-installable Python distribution.

```python
# setup.py
from setuptools import setup, find_packages

requires = [
    'pyramid',
    'plaster_pastedeploy',
    'pyramid-ipython',
    'waitress'
]

setup(
    name='pyramid_todo',
    version='0.0',
    description='A To-Do List build with Pyramid',
    author='<Your name>',
    author_email='<Your email>',
    keywords='web pyramid pylons',
    packages=find_packages(),
    include_package_data=True,
    install_requires=requires,
    entry_points={
        'paste.app_factory': [
            'main = todo:main',
        ]
    }
)
```

The last bit there, `entry_points`, sets up...entry points into our application that other services can use.
In particular we're allowing for the `plaster_pastedeploy` package to access what will be the `main` function in our application for building an application object and serving it.
We'll circle back to all this in a bit.

When we installed `pyramid`, we also gained a few Pyramid-specific shell commands.
We'll be concerned chiefly with `pserve` and `pshell`.
`pserve` will take an INI-style configuration file that we specify as an argument and serve our application locally.
`pshell` will also take a configuration file as an argument, but instead of serving our application it'll open up for us a Python shell that is aware of our application and its internal configuration.

The aforementioned configuration file is pretty important, so it might be worth investigating a bit more.
Pyramid can take its configuration both from environment variables and from a configuration file.
To avoid too much confusion between what is where, we'll write most of our configuration in the configuration file, with only a select few sensitive configuration parameters set in the virtual environment.

Let's create a file called `config.ini`

```ini
[app:main]
use = egg:todo
pyramid.default_locale_name = en

[server:main]
use = egg:waitress#main
listen = localhost:6543
```

Here we're saying a couple of things:

- For where the actual application is coming from, use the `main` function located in the `todo` package installed in the environment
- To serve this app, use the `waitress` package installed in the environment, and serve on localhost port 6543

When serving an application and working in development, it helps to have some logging set up so that you can see what's going on.
Let's add some configuration to handle logging for our application:

```ini
# continuing on...
[loggers]
keys = root, todo

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = INFO
handlers = console

[logger_todo]
level = DEBUG
handlers =
qualname = todo

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(asctime)s %(levelname)-5.5s [%(name)s:%(lineno)s][%(threadName)s] %(message)s
```

In short, we're asking to log everything to do with the application in general to the console.
If we want less output, we can set the logging level to `WARN`, where message will only fire if there's a problem.

Because Pyramid is meant for an application that grows, we should plan out a file structure that could support such an application.
Web applications can, of course, be built however you want.
But, generally, the conceptual blocks you're going to want to cover will contain

- **models** for containing the code and logic for dealing with data representations
- **views** for code and logic pertaining to the request-response cycle
- **routes** for the paths for access to the functionality of your application
- **scripts** for any code that might be used in configuration or management of the application itself

Given the above, we can have a file structure like so:

```
setup.py
config.ini
todo/
    __init__.py
    models.py
    routes.py
    views.py
    scripts/
```

Much like Flask's `app` object, Pyramid will have its own central configuration.
It comes from its `config` module and is known as the `Configurator` object.
We'll be using this object to handle everything from our route configuration, to pointing to where our models and views exist.
We'll be doing all of this in an inner directory called `todo`, within an `__init__.py` file.

```python
# todo/__init__.py

from pyramid.config import Configurator

def main(global_config, **settings):
    """Returns a Pyramid WSGI application."""
    config = Configurator(settings=settings)
    config.scan()
    return config.make_wsgi_app()
```

The `main` function here will look for some global configuration from your environment, as well as any settings that came through the particular configuration file that you provide when you run the application.
It'll then take those settings and use them to build an instance of the `Configurator` object, which for all intents and purposes is the factory for your application.
Finally, `config.scan()` will look for any views that we'd like to attach to our application that are marked as Pyramid views.

Wow, that was a lot to configure.

## Pyramid - Routes and Views

Now that we've gotten a chunk of the configuration under our belts, we can actually start to add functionality to our application.
As we've seen before, functionality comes in the form of URL routes that external clients can hit, which then map to functions that Python can run for you.

With Pyramid, all functionality must be added to the `Configurator` in some way, shape, or form.
For example, let's say that we want to build the same simple `hello_world` view that we built with Flask, mapping to the route of `/`.
With Pyramid, we can register the `/` route with the `Configurator` using the `.add_route()` method.
This method takes as arguments the name for the route that you want to add, as well as the actual pattern that must be matched to access that route.
For this particular case, we'll want to add to our `Configurator` like so:

```python
config.add_route('home', '/')
```

And until we create a view and attach it to that route, that path into our application sits open and alone.
When we add the view, we have to make sure to include the `request` object in the parameter list.
Every Pyramid view _must_ have the `request` object as its first parameter, as that's what's being passed as the first argument to the view when it's being called by Pyramid.

One similarity that Pyramid views will share with Flask is that you can mark a function as a view with a decorator.
Specifically, the `@view_config` decorator from `pyramid.view`.

In our own `views.py`, let's build the view that we want to see in the world.

```python
from pyramid.view import view_config

@view_config(route_name="hello", renderer="string")
def hello_world(request):
    """Print 'Hello, world!' as the response body."""
    return 'Hello, world!'
```

With the `@view_config` decorator, we have to at least specify the name of the route that will map to this particular view.
You can stack `view_config` decorators on top of one another to map to multiple routes if you want, but you have to have at least one to connect view the view at all and each one must include the name of a route.

The other argument, `renderer`, is optional but _not really_.
If we hadn't specified a renderer, we would've had to deliberately construct the HTTP response we wanted to send back to the client using the `Response` object from `pyramid.response`.
By specifying the `renderer` as a string, Pyramid knows to take whatever is returned by this function and wrap it in that same `Response` object with the MIME type of `text/plain`.
By default, Pyramid allows you to use `string` and `json` as renderers.
If you've attached a templating engine to your application because you want to have Pyramid generate your HTML as well, then you can point directly to your HTML template as your renderer.

Alright, so that first view is done.
Let's just see what `__init__.py` looks like now with the attached route.

```python
# in __init__.py
from pyramid.config import Configurator

def main(global_config, **settings):
    """Returns a Pyramid WSGI application."""
    config = Configurator(settings=settings)
    config.add_route('hello', '/')
    config.scan()
    return config.make_wsgi_app()
```

Spectacular.
Getting here was no easy feat, but now that we're set up we can add functionality with significantly less difficulty.

## Pyramid - Smoothing a Rough Edge

Right now our application only has one route, but one can imagine that an application of some larger size can have many dozens or even *hundreds* of routes.
Containing them all in the same `main` function that contains our central configuration as well isn't really the best idea; the space becomes cluttered!
Thankfully we can include routes fairly easily with a couple tweaks to our application.

**One**: in that `routes.py` file create a function called `includeme` (yes, it must actually be named this) that takes a configurator object as an argument.

```python
# in routes.py
def includeme(config):
    """Include these routes within the application."""
```

**Two**: move that `config.add_route` method call from `__init__.py` into that `includeme` function:

```python
def includeme(config):
    """Include these routes within the application."""
    config.add_route('hello', '/')
```

**Three**: alert the Configurator that you need to include this `routes.py` file as a part of its configuration.
Because it's in the same directory as `__init__.py` we can get away with specifying the import path to this file as `.routes`.

```python
# in __init__.py
from pyramid.config import Configurator

def main(global_config, **settings):
    """Returns a Pyramid WSGI application."""
    config = Configurator(settings=settings)
    config.include('.routes')
    config.scan()
    return config.make_wsgi_app()
```

## Pyramid - Connecting the Database

As we had done with Flask, we're going to want to persist data by connecting a database.
Pyramid will leverage [SQLAlchemy](https://www.sqlalchemy.org/) directly instead of using a specially-tailored package.

Let's first get the easy part out of the way.
We will require `psycopg2` and `sqlalchemy` to talk to our Postgres database and manage our models, so we add them to our `setup.py`.

```python
# in setup.py
requires = [
    'pyramid',
    'pyramid-ipython',
    'waitress',
    'sqlalchemy',
    'psycopg2'
]
# blah blah other code
```

Now, we have a decision to make regarding the inclusion of our database's URL.
There's no wrong answer here; what you do will depend on the application you're building and how public your codebase needs to be.

THe first option will have us keep as much of our configuration in one place as possible by hard-coding the database URL into the `config.ini` file.
One drawback is that we create a security risk for applications with a public codebase.
Anyone that can view the codebase will be able to see the full database URL, including username, password, database name and port.
Another is maintainability, where if we need to change environments, or for whatever reason need to change which database our application should point at, we have to then modify the `config.ini` file directly.
Either that or maintain one configuration file for each new environment, adding potential for discontinuity and errors in the application.
**If this is the option chosen**, then we would modify the `config.ini` file under the `[app:main]` heading to include this key-value pair:

```ini
sqlalchemy.url = postgres://localhost:5432/pyramid_todo
```

The second option will have us specify the location of the database URL when we create the `Configurator`, pointing to an environment variable whose value can be set depending on what environment we're working in.
One drawback here is that we're now further splintering our configuration, having some in the `config.ini` file and some in the Python codebase directly.
Another drawback is that when we need to use that database URL anywhere else in our application, like in a database management script, we need to code in a second reference to that same environment variable (or set up the variable in one place and import from that location).
**If this is the option chosen**, then we'll add like so

```python
# in __init__.py
import os
from pyramid.config import Configurator

SQLALCHEMY_URL = os.environ.get('DATABASE_URL', '')

def main(global_config, **settings):
    """Returns a Pyramid WSGI application."""
    settings['sqlalchemy.url'] = SQLALCHEMY_URL # <-- important!
    config = Configurator(settings=settings)
    config.include('.routes')
    config.scan()
    return config.make_wsgi_app()
```

## Pyramid - Defining Objects

Ok, so now we have a database.
We now need `Task` and `User` objects.

Because we're using SQLAlchemy directly, we'll have a somewhat significant departure from Flask with how we built objects, but nothing too far away.
First, every object that we wish to construct must inherit from SQLAlchemy's [declarative base class](http://docs.sqlalchemy.org/en/latest/orm/extensions/declarative/api.html#api-reference).
It'll keep track of everything that inherits from it, allowing for simpler management of the database.

```python
# in models.py
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class Task(Base):
    pass

class User(Base):
    pass
```

The columns, data types for those columns, and model relationships will be declared in much the same way as with Flask, though we'll import them directly from SQLAlchemy instead of some pre-constructed `db` object.
All else is the same.

```python
# in models.py
from datetime import datetime
import secrets

from sqlalchemy import (
    Column, Unicode, Integer, DateTime, Boolean, relationship
)
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class Task(Base):
    """Tasks for the To Do list."""
    id = Column(Integer, primary_key=True)
    name = Column(Unicode, nullable=False)
    note = Column(Unicode)
    creation_date = Column(DateTime, nullable=False)
    due_date = Column(DateTime)
    completed = Column(Boolean, default=False)
    user_id = Column(Integer, ForeignKey('user.id'), nullable=False)
    user = relationship("user", back_populates="tasks")

    def __init__(self, *args, **kwargs):
        """On construction, set date of creation."""
        super().__init__(*args, **kwargs)
        self.creation_date = datetime.now()

class User(Base):
    """The User object that owns tasks."""
    id = Column(Integer, primary_key=True)
    username = Column(Unicode, nullable=False)
    email = Column(Unicode, nullable=False)
    password = Column(Unicode, nullable=False)
    date_joined = Column(DateTime, nullable=False)
    token = Column(Unicode, nullable=False)
    tasks = relationship("Task", back_populates="user")

    def __init__(self, *args, **kwargs):
        """On construction, set date of creation."""
        super().__init__(*args, **kwargs)
        self.date_joined = datetime.now()
        self.token = secrets.token_urlsafe(64)
```

Note that we're not adding a `config.include` line for `models.py` anywhere because we don't need to.
We only add a `config.include` line if there's some part of the application's configuration that needs changing.
All that we've done here is created two objects, inheriting from some `Base` class that SQLAlchemy gave us.

## Pyramid - Initializing the Database

So we have our models.
Let's write ourselves a script whose job it will be to talk to and initialize the database.
In the `scripts` directory, we'll create two files: `__init__.py` and `initializedb.py`.
The first will simply be there to turn the `scripts` directory into a Python package.
The second will be the actual script we need for our database management.

Within `initializedb.py` we'll need a function whose job it will be to set up the necessary tables in our database.
Similar to what we did with Flask, we'll need this script to be aware of the `Base` object, whose metadata keeps track of every class that inherits from it.
We'll also need our database URL so that we can point to and modify its tables.

As such, we can get away with this as our database initialization script:

```python
# initializedb.py
from sqlalchemy import engine_from_config
from todo import SQLALCHEMY_URL
from todo.models import Base

def main():
    settings = {'sqlalchemy.url': SQLALCHEMY_URL}
    engine = engine_from_config(settings, prefix='sqlalchemy.')
    if bool(os.environ.get('DEBUG', '')):
        Base.metadata.drop_all(engine)
    Base.metadata.create_all(engine)
```

**Important Note: this will only work if we go with the second option of including the database url as an environment variable in `todo/__init__.py`**.
If instead the database URL was stored in the configuration file, we'd have to include a few lines to read that file.
It'd look something like this:

```python
# alternate initializedb.py
from pyramid.paster import get_appsettings
from pyramid.scripts.common import parse_vars
from sqlalchemy import engine_from_config
import sys
from todo.models import Base

def main():
    config_uri = sys.argv[1]
    options = parse_vars(sys.argv[2:])
    settings = get_appsettings(config_uri, options=options)
    engine = engine_from_config(settings, prefix='sqlalchemy.')
    if bool(os.environ.get('DEBUG', '')):
        Base.metadata.drop_all(engine)
    Base.metadata.create_all(engine)
```

Either way, in our `setup.py` we can add a console script that will access and run this function.

```python
# bottom of setup.py
setup(
    # ... other stuff
    entry_points={
        'paste.app_factory': [
            'main = todo:main',
        ],
        'console_scripts': [
            'initdb = todo.scripts.initializedb:main',
        ],
    }
)
```

When this package is then installed, we'll have access to a new console script called `initdb`, which will construct the tables in our database.
If the database URL is stored in the configuration file, we'll have to include the path to that file when we invoke the command.
It'd look like `$ initdb /path/to/config.ini`.

## Pyramid - Requests and the DB

Ok, here's where it gets a little deep.
Let's talk about **transactions**.
A "transaction", in an abstract sense, is any change made to an existing database.
As we saw with Flask, transactions are persisted no sooner than when they are committed.
If changes have been made that haven't yet been committed, and we want those changes to not actually occur (maybe an error is thrown in the process or something?), we can **rollback** a transaction and abort those changes.

In Python, the [transaction package](http://zodb.readthedocs.io/en/latest/transactions.html) allows us to interact with transactions as objects, which can roll together multiple changes into one single commit.
`transaction` provides us with **transaction managers**, which allow our applications a straightforward, thread-aware way of handling those transactions so that all we need to think about is what to change.
The `pyramid_tm` package will take the transaction manager from `transaction` and wire it up in a way that's appropriate for Pyramid's request-response cycle, attaching a transaction manager to every incoming request.

Now, normally with Pyramid the `request` object is populated when the route mapping to a view is accessed and the view function is called.
**Every view function will have a `request` object to work with.**
However, Pyramid allows us to modify its configuration to add whatever we might need to the `request` object.
We can use the transaction manager that we'll be adding to the `request` to create a session for us with every request and add that session _to_ the request.

Yay, so why is this important?

By attaching a _**transaction-managed session**_ to the `request` object, we're setting ourselves up so that **whenever the request is done being processed by the view, any changes we made to the database session will be committed without us explicitly committing.**
Let's see what all these concepts look like in code.

```python
# __init__.py
import os
from pyramid.config import Configurator
from sqlalchemy import engine_from_config
from sqlalchemy.orm import sessionmaker
import zope.sqlalchemy

SQLALCHEMY_URL = os.environ.get('DATABASE_URL', '')

def get_session_factory(engine):
    """Return a generator of database session objects."""
    factory = sessionmaker()
    factory.configure(bind=engine)
    return factory

def get_tm_session(session_factory, transaction_manager):
    """Build a session and register it as a transaction-managed session."""
    dbsession = session_factory()
    zope.sqlalchemy.register(dbsession, transaction_manager=transaction_manager)
    return dbsession

def main(global_config, **settings):
    """Returns a Pyramid WSGI application."""
    settings['sqlalchemy.url'] = SQLALCHEMY_URL
    settings['tm.manager_hook'] = 'pyramid_tm.explicit_manager'
    config = Configurator(settings=settings)
    config.include('.routes')
    config.include('pyramid_tm')
    session_factory = get_session_factory(engine_from_config(settings, prefix='sqlalchemy.'))
    config.registry['dbsession_factory'] = session_factory
    config.add_request_method(
        lambda request: get_tm_session(session_factory, request.tm),
        'dbsession',
        reify=True
    )

    config.scan()
    return config.make_wsgi_app()
```

That looks like a lot but all that we did was what was said above, plus adding an attribute to the `request` object called `request.dbsession`.

We're asking for a few new packages to be included here, so let's update `setup.py` with those packages.

```python
# in setup.py
requires = [
    'pyramid',
    'pyramid-ipython',
    'waitress',
    'sqlalchemy',
    'psycopg2',
    'pyramid_tm',
    'transaction',
    'zope.sqlalchemy'
]
# blah blah other stuff
```

### Routes and Views Revisited

We need to make some real views that handle the data within our database, and the routes that map to them.

Let's start with the routes.
Previously we created the `routes.py` file to handle all of our routes but didn't do much beyond the basic `/` route.
Let's fix that.

```python
# routes.py
def includeme(config):
    config.add_route('info', '/api/v1/')
    config.add_route('register', '/api/v1/accounts')
    config.add_route('profile_detail', '/api/v1/accounts/{username}')
    config.add_route('login', '/api/v1/accounts/login')
    config.add_route('logout', '/api/v1/accounts/logout')
    config.add_route('tasks', '/api/v1/accounts/{username}/tasks')
    config.add_route('task_detail', '/api/v1/accounts/{username}/tasks/{id}')
```

Now we not only have static URLs like `/api/v1/accounts`, but are able to handle some variable URLs like `/api/v1/accounts/{username}/tasks/{id}` where any variable in a URL will be surrounded by curly braces.

We can create the same type of view that we saw before with Flask, the view to create an individual task.
We can use the `@view_config` decorator to ensure that it only takes incoming `POST` requests, and check out how Pyramid handles data from the client.
Let's write it out then check out how it differs from Flask's version.

```python
# in views.py
from datetime import datetime
from pyramid.view import view_config
from todo.models import Task, User

INCOMING_DATE_FMT = '%d/%m/%Y %H:%M:%S'

@view_config(route_name="tasks", request_method="POST", renderer='json')
def create_task(request):
    """Create a task for one user."""
    response = request.response
    response.headers.extend({'Content-Type': 'application/json'})
    user = request.dbsession.query(User).filter_by(username=request.matchdict['username']).first()
    if user:
        due_date = request.json['due_date']
        task = Task(
            name=request.json['name'],
            note=request.json['note'],
            due_date=datetime.strptime(due_date, INCOMING_DATE_FMT) if due_date else None,
            completed=bool(request.json['completed']),
            user_id=user.id
        )
        request.dbsession.add(task)
        response.status_code = 201
        return {'msg': 'posted'}
```

To start, we note on the `@view_config` decorator that the only type of request we're looking for this view to handle is a "POST" request.
If we want to specify one type of request or one set of requests, we either provide the string noting the request or a tuple/list of such strings.

```python
response = request.response
response.headers.extend({'Content-Type': 'application/json'})
# ...other code...
response.status_code = 201
```

The HTTP response that will be sent to client will be generated based on `request.response`.
Normally, we wouldn't have to worry about that object.
It'd just produce a properly-formatted HTTP response and we'd never know the difference.
However, because we want to do something specific like modify the response's status code and headers, we need to access that response and its methods/attributes.

Unlike Flask, we don't need to modify the view function parameter list just because we have variables in the route URL.
Instead, anytime a variable exists in the route URL it is collected in the `matchdict` attribute of the `request`.
It'll exist there as a key-value pair, where the key will be the variable (e.g. "username") and the value will be whatever value was specified in the route (e.g. "bobdobson").
Regardless of what value is passed in through the route URL, it'll always show up as a string in the matchdict.
So, when we want to pull the username from the incoming request URL, we access it with `request.matchdict['username']`

```python
user = request.dbsession.query(User).filter_by(username=request.matchdict['username']).first()
```

The way that we're querying for our objects when using `sqlalchemy` directly will differ significantly from what the `flask-sqlalchemy` package allows.
Recall that when we used `flask-sqlalchemy` to build our models, we had our models inherit from the `db.Model` object.
That `db` object **already contained a connection to the database**, so using that connection we could perform a straightforward operation like `User.query.all()`.

That simple interface isn't present here, as the models in our Pyramid app inherit from `Base`, which is generated from `declarative_base()`, coming directly from the `sqlalchemy` package.
There is no direct awareness of the database it'll be accessing.
That awareness was attached to the `request` object via our app's central configuration as the `dbsession` attribute.
Here's the code that did that for us from above:

```python
config.add_request_method(
    lambda request: get_tm_session(session_factory, request.tm),
    'dbsession',
    reify=True
)
```

With all that being said, **whenever we want to query OR modify the database, we must work through `request.dbsession`**.
In our particular case, we want to query our "users" table for a specific user, with their username as our identifier.
As such, we provide our `User` object as an argument to the `.query` method, and then do our normal SQLAlchemy operations from there.

An interesting thing about this way of querying the database is that you can query for more than just one object or list of one type of object.
You can query for:

 - object attributes on their own. e.g. `request.dbsession.query(User.username)` would just query for usernames
 - tuples of object attributes. e.g. `request.dbsession.query(User.username, User.date_joined)`
 - tuples of multiple objects. e.g. `request.dbsession.query(User, Task)`

The data sent along with the incoming request will be found within the `request.json` dictionary.

The last major difference is that because we went through all of the machinations necessary to attach the committing of a session's activity to Pyramid's request-response cycle, we don't have to call `request.dbsession.commit()` at the end of our view.
Convenient, but there is one thing that we need to be aware of moving forward.
If instead of a new add to the database, we wanted to edit a pre-existing object in the database, we couldn't use `request.dbsession.commit()`.
Pyramid will throw an error, saying something along the lines of commit behavior being handled by the transaction manager, so you can't call it on your own.
And if you don't do something that resembles committing your changes, then your changes won't actually stick.

The solution here is to use `request.dbsession.flush()`.
The job of `.flush()` is to signal to the database that some changes have been made and need to be included with the next commit.

## For the Future

At this point, we've set up most of the important parts of the Pyramid analog to what we were able to construct with Flask in part one.
As it was before, there's more that goes into an application, but much of the meat is handled here.
Other view functions will follow similar formatting to what we've seen above, and of course there's always the question of security (which Pyramid has built in!).

One of the major differences that I see going through the setup of a Pyramid application is that there's a much more intense configuration step than there is with Flask.
I actually broke this down into those configuration steps so that we could understand more about exactly what's going on when a Pyramid application is constructed.
However, it'd be disingenuous of me to act like I've known all of this since I first started programming.
When I first learned the Pyramid framework, I learned it with Pyramid 1.7 and its scaffolding system of `pcreate`, which builds out most of the necessary configuration so that all you need to do is think about the functionality you want to build.

As of Pyramid 1.8, `pcreate` has been deprecated in favor of [cookiecutter](https://cookiecutter.readthedocs.io/en/latest/), which effectively does the same thing.
The difference is that it's maintained by someone else, and there are cookiecutter templates for more than just Pyramid projects.
Now that we've gone through the components of a Pyramid project, **I'd fully endorse never building a Pyramid project from scratch again and instead using a cookiecutter template**.
Why do the hard work if you don't have to?
For a template that'd accomplish much of the same of what we've written here (and a little bit more), check out the [pyramid-cookiecutter-alchemy](https://github.com/Pylons/pyramid-cookiecutter-alchemy) template.
That's actually similar to the `pcreate` scaffold that I first learned Pyramid with.

Thanks for reading, and see you next time when I'll dive into the Tornado web framework!