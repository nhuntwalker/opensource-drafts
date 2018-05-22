# A Web App In Four Frameworks - Intro & Flask

For almost two years I was an instructor of web and software development at the Seattle-area code school [Code Fellows](https://www.codefellows.org/).
I taught the beginner- and intermediate-level web development classes, in addition to advanced software development with Python.
As the lead Python instructor I would routinely take students through the Pyramid and Django web frameworks to prepare them for jobs as web developers upon graduation.

Now, most of my students didn't know this until after the course was over, but when I started working at Code Fellows I had never once used Python for web development.
I had only ever used WordPress and a smattering of JavaScript, reserving my Python for data analysis in my Astronomy PhD program.
Anyway, all the Python web dev knowledge that I gained I gathered on the job, so I only had exposure to Pyramid and Django.
I had heard about the Flask framework as being this thing called a "microframework", and I had seen the name "tornado" pop up whenever I installed Jupyter notebook in a new virtual environment, but I had only at best poked at each.

I wanted to gain some understanding of these other frameworks that I had only heard about.
When [PyCascades 2018](https://www.pycascades.com/) was announced, it seemed like the perfect opportunity to build a presentation that would require me to dive into each and present their functionality to the world.
And so, here we are.

## The Task at Hand

The idea here is to show and compare the Flask, Tornado, Pyramid, and Django web frameworks on the same playing field--in the construction of an API for a simple web application.
The concept bringing each of the frameworks together is that of a To Do list application.
The API is itself fairly straightforward:

- New visitors to the site should be able to register new accounts
- Registered users can log in, log out, see information for their profiles, and edit their info
- Registered users can create new task items, see their existing tasks, and edit existing tasks

All this rounds out to a somewhat compact set of API endpoints that each backend would need to implement, along with the allowed HTTP methods:

- `GET /`
- `POST /accounts`
- `POST /accounts/login`
- `GET /accounts/logout`
- `GET, PUT, DELETE /accounts/<str : username>`
- `GET, POST /accounts/<str : username>/tasks`
- `GET, PUT, DELETE /accounts/<str : username>/tasks/<int : id>`

Each framework puts together its routes, models, views, database interaction, and overall application configuration differently.
This will be the first post of 4, focusing on how [Flask](http://flask.pocoo.org/) gets it done.

## Flask - Startup & Configuration

Like most of the widely-used Python libraries, Flask is a Python package installable from the [Python Package Index](https://pypi.python.org).
So, after you've created a directory for yourself to work in (something like `flask_todo` is a fine directory name), install the `flask` package.
You'll also want to install `flask-sqlalchemy` so that your Flask application has a simple way to talk to a SQL database.

I personally prefer to do this type of work within a Python 3 virtual environment, so typing the following in your command line should get us right where we need to be:

```
$ mkdir flask_todo
$ cd flask_todo
$ pipenv install --python 3.6
$ pipenv shell
(flask-someHash) $ pipenv install flask flask-sqlalchemy
```

If we want to turn this into a `git` repository, this would be a good place to run `git init` as well.
It'll be the root of our project, and should we want to export our codebase to a different machine, it'd help to have all of the setup files that are needed here.

A good way to get moving with this is to turn our codebase into an installable Python distribution.
At our project's root, we'll create our `setup.py`, as well as a directory to hold our source code called `todo`.

Our `setup.py` should look something like this:

```python
from setuptools import setup, find_packages

requires = [
    'flask',
    'flask-sqlalchemy',
    'psycopg2',
]

setup(
    name='flask_todo',
    version='0.0',
    description='A To-Do List built with Flask',
    author='<Your actual name here>',
    author_email='<Your actual e-mail address here>',
    keywords='web flask',
    packages=find_packages(),
    include_package_data=True,
    install_requires=requires
)
```

This way, anytime we want to install or deploy our project we have all the necessary packages in our `requires` list.
We'll also have all we need to set up our package and install it in our `site-packages`.
For more information on how to write an installable Python distribution, check out [the docs on setup.py](https://docs.python.org/3/distutils/setupscript.html).

Now, we've already created our `todo` directory, containing our source code.
Within the `todo` directory we're going to create an `app.py` and a blank `__init__.py` file.
The `__init__.py` is there so that we may import from `todo` as if it was an installed package.
The `app.py` file will be our application's root.
This is where all the `Flask` application goodness will go, and we'll create an environment variable that points to that file.
If you're using `pipenv` like I am here, then you can locate your virtual environment with `pipenv --venv` and set up that environment variable in your environment's `activate` script.

```
# in your activate script, probably at the bottom (but anywhere will do)

export FLASK_APP=$VIRTUAL_ENV/../todo/app.py
export DEBUG='True'
```

When we installed `Flask`, we also installed the `flask` command-line script.
Typing `flask run` will prompt our virtual environment's Flask package to run an HTTP server using the `app` object in whatever script the `FLASK_APP` environment variable is pointing to.
Above I've also included an environment variable named `DEBUG` which will be used a bit later.

Let's talk about this `app` object.

In `todo/app.py` we're going to create an `app` object which is an instance of the `Flask` object.
It'll act as the central configuration object for the entire application.
We'll use it to set up any pieces of our application that we may need for extended functionality, e.g. a database connection and help with authentication.

We'll use it far more regularly to set up the routes that will become our application's points of interaction.
This is a lot of words thus far, so let's see the code that it corresponds to.

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    """Print 'Hello, world!' as the response body."""
    return 'Hello, world!'
```

Here we have the most basic complete Flask application.
`app` is an instance of `Flask`, taking in the `__name__` of the script file.
This let's Python know how to import from files relative to this one.
Decorating our first **view** function we have `app.route` decorator, where we can specify one of the routes that can be used to access our application.
We'll see this more later.

Any view that we specify must be decorated by `app.route` to actually be a functional part of the application.
We can have as many functions as you want scattered across our application, but if we want that functionality to be accessible from anything external to the application we must decorate that function and specify a route to make it into a view.

In the example above, when the app is running and accessed at `http://domainname/`, a user will receive `"Hello, World!"` as a response.

## Flask - Connecting the Database

While the code example specified above does represent a complete Flask application, it doesn't do much of anything interesting.
One interesting thing that a web application _can_ do is persist user data, but it needs the help of and connection to a database.

Flask is very much a "Do It Yourself" web framework.
For us this means that there's no built-in database interaction.
To connect a SQL database to our Flask application we'll use the `flask-sqlalchemy` package.
The `flask-sqlalchemy` package will need just one thing in order to connect to a SQL database: the database URL.

Now, we can use a wide variety of SQL database management systems with `flask-sqlalchemy`, as long as that DBMS has an intermediary following the [DBAPI-2 standard](https://www.python.org/dev/peps/pep-0249/).
I personally enjoy PostgreSQL if for no other reason than I've used it a lot, so our intermediary will be the `psycopg2` package.
We won't have to do anything with it ourselves; `flask-sqlalchemy` will recognize from our database URL that we're using Postgres, make sure that `psycopg2` is installed in our environment, and use `psycopg2` to talk to our Postgres database.
We just need to make sure that we install `psycopg2` and include it in our list of required packages in `setup.py`.

Flask needs the database url to be a part of its central configuration through the `SQLALCHEMY_DATABASE_URI` attribute.
If we wanted something quick and dirty, we could just hardcode a database URL right into our application.

```python
# top of app.py
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgres://localhost:5432/flask_todo'
db = SQLAlchemy(app)
```

However, this is not a sustainable solution.
If we change databases or don't want our database URL visible in source control, we'll have to go through extra steps to ensure that our information is appropriate for the appropriate environment.

We can make things simpler for ourselves if we use environment variables, so that no matter what machine our code is on we can always point at the right stuff if that stuff is configured in the running environment.
It'll also ensure that even though we need that information to run our application, it never shows up as a hardcoded value in source control.

In the same place where we declared `FLASK_APP`, declare a `DATABASE_URL` pointing to the location of our Postgres database.
In development one tends to work locally, so we'll point to our local database.

```
# also in your activate script

export DATABASE_URL='postgres://localhost:5432/flask_todo'
```

Now in `app.py`, we'll include the database url into our app configuration.

```python

app.config['SQLALCHEMY_DATABASE_URI'] = os.environ.get('DATABASE_URL', '')
db = SQLAlchemy(app)
```

and just like that, our application now has a database connection!

## Flask - Defining Objects

Having a database to talk to is a good first step.
Now, we'll define some objects with which to fill that database.

In application development, a "model" refers to the data representation of some real or conceptual object.
So, if the application we're building is a car dealership, we may define for ourselves a `Car` model, that encapsulates all of the attributes and behavior that we can think of for a car.

In our particular case, we're dealing with a To Do list with Tasks, each Task belonging to a User.
So before we think too deeply about how they're related to each other, we can start by defining objects for Tasks and Users.

With the `flask-sqlalchemy` package, we're leveraging [SQLAlchemy](https://www.sqlalchemy.org/) to set up and inform the structure of our database.
We'll define a model that will live in our database by inheriting from the `db.Model` object, and we'll define the attributes of those models as `db.Column` instances.
For each column we must specify a data type, so we'll pass that data type into the call to `db.Column` as the first argument.

Because the model definition occupies a different conceptual space than our application configuration, we should make a `models.py` to hold our model definitions separate from our `app.py`.
When we construct our Task model, we expect it to have the following attributes:

- `id`: a value that's a unique identifier that we can pull from the database
- `name`: the name or title of the task that the user will see when the task is listed
- `note`: any extra comments that one might want to leave along with their task
- `creation_date`: the date and time at which the task was created
- `due_date`: the date and time at which the task is due to be completed, if at all
- `completed`: an indication as to whether or not the task has been completed.

Given the above attribute list for Task objects, we can define our application's `Task` object like so:

```python
from .app import db
from datetime import datetime

class Task(db.Model):
    """Tasks for the To Do list."""
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.Unicode, nullable=False)
    note = db.Column(db.Unicode)
    creation_date = db.Column(db.DateTime, nullable=False)
    due_date = db.Column(db.DateTime)
    completed = db.Column(db.Boolean, default=False)

    def __init__(self, *args, **kwargs):
        """On construction, set date of creation."""
        super().__init__(*args, **kwargs)
        self.creation_date = datetime.now()
```

Note the extension of the class constructor method.
At the end of the day, any model that we construct is still a Python object, and so must go through construction in order to be instantiated.
One thing that we want to ensure is that the creation date of our model instance reflects its actual date of creation.
As such, we can explicitly set that relationship by effectively saying "when an instance of this model is constructed, record the date and time and set it as the creation date."

## Flask - Model Relationships

In a given web application we may want to be able to express relationships between objects.
In our example of the To Do list, we conceptually have users that own multiple tasks, with each task being owned by only one user.
This is an example of a "Many-to-One" relationship, also known as a Foreign Key relationship, where the tasks are the "many" and the user owning those tasks is the "one".

With Flask we can specify a many-to-one relationship using the `db.relationship` function.
First, we have to build the User object.

```python
class User(db.Model):
    """The User object that owns tasks."""
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.Unicode, nullable=False)
    email = db.Column(db.Unicode, nullable=False)
    password = db.Column(db.Unicode, nullable=False)
    date_joined = db.Column(db.DateTime, nullable=False)
    token = db.Column(db.Unicode, nullable=False)

    def __init__(self, *args, **kwargs):
        """On construction, set date of creation."""
        super().__init__(*args, **kwargs)
        self.date_joined = datetime.now()
        self.token = secrets.token_urlsafe(64)
```

It looks very similar to our Task object; you'll find that most objects have the same basic format of class attributes as table columns.
Every once in a while you'll run into something a little different, including some multiple-inheritance magic, but what you see above will be more the norm than not.

Now that our `User` model is created, we can set up our Foreign Key relationship.
On the "many", we set fields for the `user_id` of the `User` that owns this task, as well as the `user` object with that ID.
We also make sure to include an keyword argument (`back_populates`) that updates the User model when the task is given a user as an owner.

On the "one", we set a field for the `tasks` that the one particular User owns.
Similar to maintaining the two-way relationship on the Task object, we set a keyword argument on the User's relationship field to update the Task when it is assigned to a user.

```python
# on the Task object
user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
user = db.relationship("User", back_populates="tasks")

# on the User object
tasks = db.relationship("Task", back_populates="user")
```

## Flask - Initializing the Database

Now that we have our models and our model relationships, we can think about setting up our database.
Flask doesn't come with its own database-management utility, so to some degree we'll have to write our own.
We don't have to get fancy with it; just something to recognize what tables are to be created and then some code to create them (or drop them should the need arise).
If you want to do something more complex, like handle updates to database tables (i.e. database migrations), you'll want to look into a tool like [Flask-Migrate](https://flask-migrate.readthedocs.io/en/latest/) or [Flask-Alembic](https://flask-alembic.readthedocs.io/en/stable/).

We'll create a script next to our `setup.py` for managing our database called `initializedb.py`.
Of course, it doesn't need to be called this, but why not give files names that are appropriate to their function?
Within `initializedb.py`, we'll import the `db` object from `app.py` and use it to create or drop our tables.
`initializedb.py` should end up looking something like this:

```python
from todo.app import db
import os

if bool(os.environ.get('DEBUG', '')):
    db.drop_all()
db.create_all()
```

If a `DEBUG` environment variable is set, then drop tables and rebuild.
Otherwise, just create the tables once, and we're good to go.

## Flask - Views & URL Config

The last bits that we need to connect our entire application are the views and routes.
In web development, a "view" in concept is some functionality that runs when a specific access point (a "route") in your application is hit.
These access points appear as URLs; paths to functionality in your application that return some data or handle some data that has been provided.
The views themselves will be some logical structures that handle specific HTTP requests from a given client, and return some HTTP response to that client.

In Flask, our views will appear as functions.
We saw one of these in the `hello_world` view seen above.
For clarity, I'll reproduce it below:

```python
@app.route('/')
def hello_world():
    """Print 'Hello, world!' as the response body."""
    return 'Hello, world!'
```

When the route of `http://domainname/` is accessed, the client will receive a response of "Hello, world!".

With Flask, a function is marked as a view when it is decorated by `app.route`.
In turn, `app.route` adds to the application's central configuration a map from the specified route to the function that runs when that route is accessed.
We can use this to start to build out the rest of our API.

Let's start with a view that only handles `GET` requests, and response with the JSON representing all the routes that will be accessible and the methods that can be used to access them.

```python
from flask import jsonify

@app.route('/api/v1', methods=["GET"])
def info_view():
    """List of routes for this API."""
    output = {
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
    return jsonify(output)
```

We only want our view to handle one _specific_ type of HTTP request, so we use `app.route` to add that restriction.
The `methods` keyword argument will take a list of strings as a value, with each string being a type of possible HTTP method.
In practice, you can use `app.route` to restrict to one or more types of HTTP request, or accept any by leaving the `methods` keyword argument alone.

Whatever we intend to return from our view function _**must**_ be a string or an object that Flask turns into a string when constructing a properly-formatted HTTP response.
The exceptions to this rule will be when you're trying to handle redirects and Exceptions thrown by your application.
What this means for you, the developer, is that you need to be able to encapsulate whatever response you're trying to send back to the client into something that can be interpreted as a string.

A good structure that contains complexity but can still be stringified is a Python dictionary.
So, I recommend that whenever you want to send some data to the client you choose a Python `dict` with whatever key-value pairs you need to convey information.
To turn that dictionary into a properly-formatted JSON response, headers and all, pass it as an argument to Flask's `jsonify` function (`from flask import jsonify`).

The view function above takes what is effectively a listing of every route that this API intends to handle and sends it to the client whenever the `http://domainname/api/v1` route is accessed.
Note that on its own Flask only supports routing to exactly-matching URIs, so accessing that same route with a trailing `/` would be raise a 404 error.
If we wanted to handle both with the same view function, we'd have to stack decorators like so:

```python
@app.route('/api/v1', methods=["GET"])
@app.route('/api/v1/', methods=["GET"])
def info_view():
    # blah blah blah more code
```

An interesting case here is that if the route we had defined had a trailing slash to start with, and the client asked for the route without the slash, we wouldn't need to double up on decorators.
Flask would redirect the client's request appropriately.
Odd that it doesn't work both ways, but that's the tool we've chosen.

## Flask - Requests and the DB

At base, a web framework's job is to handle incoming HTTP requests and return HTTP responses.
The previously-written view doesn't really have much to do with HTTP requests aside from the URI that was accessed.
It doesn't actually process any data.
Let's now consider how Flask behaves when data needs handling.

THe first thing to know is that Flask doesn't provide a separate `request` object to each view function.
Flask has _**one**_ global request object that every view function can use, and that object is conveniently named `request` and importable from the Flask package.

The next thing is that Flask's route patterns can have a bit more nuance to them. 
One scenario is a hard-coded route that must be matched perfectly in order to activate a view function.
Another scenario is a route pattern that can handle a range of routes, all mapping to one view by allowing for a part of that route to be variable.
If the route in question does have a variable, the corresponding value can be accessed from the same-named variable in the view's parameter list.

```python
@app.route('/a/sample/<variable>/route)
def some_view(variable):
    # some code blah blah blah
```

If we want to communicate with the database within a view, we must use the `db` object that was populated toward the top of the script.
Its `session` attribute is our connection to the database when we want make changes.
If we just want to query for objects, the objects built from `db.Model` have their own database interaction layer through the `query` attribute.

Finally, any response that we want from a view that's more complex than just a string must be built deliberately.
Previously we built a response using a "jsonified" dictionary, but in doing so certain assumptions were made (e.g. 200 status code, status message "OK", Content-Type of "text/plain").
Any special sauce that we want in your HTTP response must be added deliberately.

Knowing these facts about working with Flask views allows us to construct a view whose job it is to create new `Task` objects.
Let's see the code below, then address it piece by piece.

```python
from datetime import datetime
from flask import request, Response
from flask_sqlalchemy import SQLAlchemy
import json

from .models import Task, User

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = os.environ.get('DATABASE_URL', '')
db = SQLAlchemy(app)

INCOMING_DATE_FMT = '%d/%m/%Y %H:%M:%S'

@app.route('/api/v1/accounts/<username>/tasks', methods=['POST'])
def create_task(username):
    """Create a new task for a user."""
    user = User.query.filter_by(username=username).first()
    if user:
        due_date = request.form['due_date']
        task = Task(
            name=request.form['name'],
            note=request.form['note'],
            due_date=datetime.strptime(due_date, INCOMING_DATE_FMT) if due_date else None,
            completed=bool(request.form['completed']),
            user_id=user.id,
        )
        db.session.add(task)
        db.session.commit()
        output = {'msg': 'posted'}
        response = Response(
            mimetype="application/json",
            response=json.dumps(output),
            status=201
        )
        return response
```

Let's start with the `@app.route` decorator.
The route we see is `'/api/v1/accounts/<username>/tasks'`, where `<username>` is a route variable.
As seen previously, we surround with angle brackets any part of our route that we want to be variable.
We then include that part of the route on the next line in the parameter list **with the same name**.
The only parameters that should be in our parameter list should be the variables in our route.

Next comes the query:

```python
user = User.query.filter_by(username=username).first()
```

We want to look for one user by username.
Conceptually, to do this we need to look at all of the User objects that are stored in our database and find the users with the username matching the one that was requested.
With Flask we can ask the `User` object directly through the `query` attribute for the instance matching our criteria.
This type of query would provide us with a list of objects (even if it's only one object or none at all), so to get the one object that we want we just call `first()`.

```python
task = Task(
    name=request.form['name'],
    note=request.form['note'],
    due_date=datetime.strptime(due_date, INCOMING_DATE_FMT) if due_date else None,
    completed=bool(request.form['completed']),
    user_id=user.id,
)
```

Whenever data is sent to our application, regardless of the HTTP method used, that data is stored on the `form` attribute of the `request` object.
Whatever the name was of the field on the front-end will be the name of the key mapped to that data in the `form` dictionary.
It'll always come in the form of a string, so if we want our data to be a specific data type, we'll have to make it explicit by casting it as the appropriate type.

The other thing to note here is the assignment of the current user's ID to the newly-instantiated `Task`.
This is how we maintain that foreign key relationship.

```python
db.session.add(task)
db.session.commit()
```

Creating a new `Task` instance is great, but its construction has no inherent connection to tables in the database.
In order to actually insert a new row into the corresponding SQL table, we must use the `session` attached to the `db` object.
`db.session.add(task)` stages the new `Task` instance for being added to the table, but doesn't add it yet.
Here it's done once, but we can add as many things as we want before committing.
`db.session.commit()` takes all of our staged changes, or "commits", and applies them to the corresponding tables in our database.

```python
output = {'msg': 'posted'}
response = Response(
    mimetype="application/json",
    response=json.dumps(output),
    status=201
)
```

The response that gets constructed is an actual instance of a `Response` object, with its `mimetype`, body, and `status` set deliberately.
The goal for this particular view is to alert the user that they created something new.
Seeing as how this particular view is supposed to be part of a backend API that sends and receives JSON, the response body must be JSON serializable.
A dictionary with a simple string message should suffice.
We ensure that it's ready for transmission by calling `json.dumps` on our dictionary, which will turn our Python object into valid JSON.
Note, we use this instead of `jsonify`, as `jsonify` constructs an actual response object using its input as the response body.
`json.dumps` in contrast just takes a given Python object and converts it into a valid JSON string if possible.

By default, the status code of any response we send with Flask will be `200`.
That will work for most circumstances where we're not trying to send back a specific redirection-level or error-level message.
In this particular case we want to explicitly let the front-end know when a new item has been created.
As such, we set the status code to be 201, corresponding to having created a new thing.

And that's it!
That's a basic view for creating a new `Task` object in Flask given the current set up of our To Do List application.
Similar views could be constructed for listing, editing, and deleting tasks, but that at least gives us an idea of how it could be done.

## The Bigger Picture

There is of course more that goes into an application than one view for creating new things.
While I haven't discussed anything about authorization/authentication systems, testing, database migration management, cross-origin resource sharing, etc., the details shown above should give you more than enough to start digging into building your own Flask applications.

What's next?
Well, the goal of this whole endeavor was to compare and contrast how different web frameworks handle the same problems, so next up will be the next framework.
Since we've already seen the major functionality and will be familiar with what types of operations that a framework addressing our specific problem are supposed to handle, the next framework's discussion shouldn't be quite as long.
And, on the positive side, it should provide an interesting canvas against which to compare the way that Flask does it.

Either way, thanks for reading.
Looking forward to throwing up the next iteration of this series soon.