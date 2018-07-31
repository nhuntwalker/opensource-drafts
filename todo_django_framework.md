# A Web App in Four Frameworks - Django

After the past three installments looking into [Flask](https://opensource.com/article/18/4/flask), [Pyramid](https://opensource.com/article/18/5/pyramid-framework), and [Tornado](https://opensource.com/article/18/6/tornado-framework), we've finally made our way to Django!
Django is, by and large, the major web framework for Python developers these days and it's not too hard to see why.
Django excels in hiding a lot of the configuration logic and letting you focus on being able to build big and build fast.

That being said, when it comes to small projects such as our everpresent To Do list, using Django can become a bit like bringing a firehose to a watergun fight.
Let's see how it all comes together below.

## Django - Intro and Startup

[Django](https://www.djangoproject.com/) styles itself as "a high-level Python Web framework that encourages rapid development and clean, pragmatic design.
Built by experienced developers, it takes care of much of the hassle of Web development, so you can focus on writing your app without needing to reinvent the wheel."
And they really mean it!
This massive web framework comes with so many batteries included that oftentimes during development it can be a mystery as to how everything manages to work together.

Not only that, the Django community is absolutely massive.
It's so massive, in fact, that there's [a whole website](https://djangopackages.org/) devoted to the third party packages people have designed to plug into Django to do a whole variety of things.
This includes everything from Authentication and Authorization, to full-on Django-powered content management systems, to e-commerce add-ons, integrations with Stripe.
Talk about not re-inventing the wheel; chances are that if you want something done with Django someone has already done it and you can just pull it into your project.

For our purposes, we're going to want to build a REST API with Django, so we're going to leverage the always popular [Django REST framework](http://www.django-rest-framework.org/).
Its job is to turn the Django framework, which was built to serve fully-rendered HTML pages built from templates, into a framework that's specifically geared toward effectively handling REST interactions.
Let's get going with that.

```
$ mkdir django_todo
$ cd django_todo
$ pipenv install --python 3.6
$ pipenv shell
(django-someHash) $ pipenv install django djangorestframework
```

For reference I'm working with `django-2.0.7` and `djangorestframework-3.8.2`.

Unlike Flask, Tornado, and Pyramid, we don't need to write our own `setup.py` file.
We're not making an installable Python package this time.
Believe it or not, Django actually takes care of that for us, in a way.
We'll still need a `requirements.txt` file to track all of our necessary installs.
However, as far as targeting modules within our Django project goes Django will let us list the subdirectories we want to have access to, then will allow us to access those directories as if they're installed packages.

But first we have to have a Django project.

When you installed Django, you installed the command-line script `django-admin`.
It's job is to manage all of the various django-related commands that help put your project together and maintain it as you continue to develop.
Instead of having you build up the entire Django ecosystem from scratch, the `django-admin` will allow you to get started with all the absolutely necessary files you need for a standard Django project.

The syntax for invoking `django-admin`'s startproject command is `django-admin startproject <projectname> <directory where you want the files>`.
Since I want the files to exist right in my current working directory I would type

```
(django-someHash) $ django-admin startproject django_todo .
```

Typing `ls` would show me one new file and one new directory.
`manage.py` is a command line-executable Python file that really ends up just being a wrapper around `django-admin`.
As such, its job is the same: to help you manage your project.
Hence the name.

The directory that it created, in my case `django_todo` inside of `django_todo`, represents **the configuration root for your project**.
Let's dig into that now.

## Django - Configuration

When I call `django_todo` the "configuration root", what I mean is that this directory holds the files necessary for generally configuring your Django project.
Pretty much everything outside of this directory will be focused solely on the "business logic" associated with your project's models, views, routes, etc.
All points that connect the project together will lead here.

Using the `ls` command will show you 4 files in the `django_todo` directory:

```
(django-someHash) $ cd django_todo
(django-someHash) $ ls
__init__.py settings.py urls.py     wsgi.py
```

`__init__.py` is empty, solely existing to turn this directory into an importable Python package.

`settings.py` is where you'll set things like whether or not you're in DEBUG mode, what database you want to use, where Django should look for your files, etc.
It is the "main configuration" part of your configuration root and we'll dig into that momentarily.

`urls.py` is, as the name implies, where you set up your URLs.
While you don't have to write every URL for your application explicitly in this file, you _**do**_ need to make this file aware of any other places where you've declared URLs.
If this file isn't aware, other URLs might as well not exist at all.

`wsgi.py` is solely for serving your application in production.
Just like how Pyramid, Tornado, and Flask exposed some "app" object that was your configured application to be served, Django must also expose one.
This is where that "app" is exposed.
You can then target it to be served with something like [gunicorn](http://gunicorn.org/), [waitress](https://docs.pylonsproject.org/projects/waitress/en/latest/), or [uwsgi](https://uwsgi-docs.readthedocs.io/en/latest/).

### Setting the Settings

Taking a look inside of `settings.py` will show you that it's absolutely massive, and these are just the defaults!
This doesn't even include hooks for the database, static files, media files, any AWS integration, or any of the other dozens of ways that a Django project can be configured.
Let's see, top to bottom, what you're given:

- `BASE_DIR` sets the base directory, or the directory where `manage.py` is located. This is useful for locating files.
- `SECRET_KEY` is a key used for cryptographic signing inside your Django project. In practice it's used for things like sessions, cookies, CSRF protection, and auth tokens. As soon as possible, the value for `SECRET_KEY` should be changed and moved into an environment variable. You could then access that value with `os.environ.get['SECRET_KEY']`
- `DEBUG` tells Django whether to run your project in development mode or production mode. In development mode, when an error pops up Django will show you the stack trace that led to the error, as well as all the settings and configuration that were involved in running the project. As you might imagine, this would be a massive security issue if `DEBUG` was set to `True` in production. A simple way to safeguard your project is to set `DEBUG` to an environment variable, like `bool(os.environ.get('DEBUG', ''))`
- `ALLOWED_HOSTS` is the literal list of hostnames that your application is being served from. In development this can be empty, but in production **your Django project will not run if the host that serves the project is not amongst the list of ALLOWED_HOSTS**.
- `INSTALLED_APPS` is the list of Django "apps" (think of them as subdirectories) that your Django project has access to. You're given a few by default to provide:
    - the built-in Django administrative website
    - Django's built-in authentication system
    - Django's one-size-fits-all manager for data models
    - Session management
    - Cookie and session-based messaging
    - Usage of static files inherent to your site, like `css` files, `js` files, any images that are a part of your site's design, etc.
- `MIDDLEWARE` is as it sounds: the list of middleware that help your Django project run. Much of it is for handling various types of security, though you can add others as you need them
- `ROOT_URLCONF` sets the location of your base-level URL configuration file. That `urls.py` that we saw before? By default, Django is pointing to that file to gather all your URLs. If for whatever reason you wanted Django to look elsewhere, you'd set the path to that location here.
- `TEMPLATES` is the list of template engines that Django would use for your site's front-end if you were relying on Django to build your HTML. Since we're not, this isn't as relevant
- `WSGI_APPLICATION` sets the location of your WSGI application&mdash;the thing that gets served when in production. By default it points to an `application` object in `wsgi.py`. You'd rarely, if ever, have to change this
- `DATABASES` sets which databases your Django project will be accessing. The `default` database _**must**_ be set. You can set others by name as long as you provide the `HOST`, `USER`, `PASSWORD`, `PORT`, database `NAME`, and appropriate `ENGINE`. As you might imagine, these are all sensitive pieces of information so it'd be best to hide them away in environment variables. [Check the Django docs](https://docs.djangoproject.com/en/2.0/ref/settings/#databases) for more details
- `AUTH_PASSWORD_VALIDATORS` is effectively a list of functions that will be run to check input passwords. You get a few by default, but if you have other, more complex validation than checking if the password matches a user's attribute, if it exceeds the minimum length, if it's one of the 1,000 most common passwords, or if the password is entirely numeric, you can list it here.
- `LANGUAGE_CODE` will set the language for your site. By default it's for US english, but you can switch it up to be other languages
- `TIME_ZONE` is the timezone for any autogenerated timestamps in your Django project. I cannot stress enough how important it is that you stick to UTC and do any timezone-specific processing elsewhere instead of trying to reconfigure this setting. As [this article] states, it's the common denominator amognst all timezones because there's no offsets to worry about. If offsets are that important to you, calculate them as you need.
- `USE_I18N` will let Django use its own translation services to translate strings for the front-end. I18N = internationalization (18 characters between "i" and "n")
- `USE_L10N` will use the common local formatting of data if set to `True`. A great example is dates: in the US it's MM-DD-YYYY. In Europe, dates tend to be seen as DD-MM-YYYY
- `STATIC_URL` this is part of a larger body of settings for serving static files. In our particular case where we'll be building a REST API we won't need to serve static files, so we can ignore it. In general though, this sets the root path after the domain name for every static file. So, if I had a logo image to serve it'd be `http://<domainname>/<STATIC_URL>/logo.gif`

These settings are pretty much ready to go by default.
One thing that we'll have to change is the database setting.
First we create the database that we'll be using with

```
(django-someHash) $ createdb django_todo
```

We want to use a PostgreSQL database just like we did with Flask, Pyramid, and Tornado.
What that means is that we'll have to change the setting to allow our server to access a PostgreSQL database.
First: the engine.
By default the database engine is `django.db.backends.sqlite3`.
We'll be changing that to `django.db.backends.postgresql`.

For more information about Django's available engines, [check the docs](https://docs.djangoproject.com/en/2.0/ref/settings/#std:setting-DATABASE-ENGINE).
Note that while it is technically possible to incorporate a NoSQL solution into your Django project, out of the box Django is strongly biased toward SQL solutions.

Next we have to specify the key-value pairs for the different parts of the connection parameters.

- `NAME` will be the name that we just created above.
- `USER` will be whatever your individual username is for accessing your postgres database
- `PASSWORD` will also be whatever password you need to access your database
- `HOST` will be the host for your database. `localhost` or `127.0.0.1` will work for us as we're developing locally.
- `PORT` is whatever PORT you've got open for Postgres. If you're following all the rules it's `5432`.

This information is obviously sensitive, but this file is expecting you to include hardcoded strings.
That's not going to work for the responsible developer.
There are several ways to address this problem but I'm going to shortcut it entirely and just set up environment variables.

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME', ''),
        'USER': os.environ.get('DB_USER', ''),
        'PASSWORD': os.environ.get('DB_PASS', ''),
        'HOST': os.environ.get('DB_HOST', ''),
        'PORT': os.environ.get('DB_PORT', ''),
    }
}
```

Before you go forward, make sure that you actually set the environment variables up or else Django will not work.
Also, install `psycopg2` so you can actually talk to your database.

## Django - Routes and Views

Let's make something actually function inside of this project.
We'll be using Django REST Framework to construct our REST API, so we have to make sure we can actually use it by adding `rest_framework` to the end of `INSTALLED_APPS` in `settings.py`.

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework'
]
```

While Django REST Framework doesn't exclusively work with class-based views (like Tornado), it is the preferred method for writing views.
Let's define one for ourselves.

We'll create a file called `views.py` in `django_todo`.
Within `views.py` we'll create our first "Hello, world!" view.

```python
# in django_todo/views.py
from rest_framework.response import Response
from rest_framework.views import APIView

class HelloWorld(APIView):
    def get(self, request, format=None):
        """Print 'Hello, world!' as the response body."""
        return Response("Hello, world!")
```

Every Django REST Framework class-based view inherits either directly or indirectly from `APIView`.
`APIView` handles a ton of stuff, but for our purposes it does these specific things:

- Set up the methods we'll need to direct traffic based on the HTTP method (e.g. GET, POST, PUT, DELETE)
- Populate the `request` object with all of the data and attributes we'll need for parsing and processing any incoming request
- Take the `Response` that every dispatch method (i.e. methods named `get`, `post`, `put`, `delete`) returns and constructs a properly-formatted HTTP response.

Yay, we have a view!
On its own it does nothing.
We need to connect it to a route.

If we hop into `django_todo/urls.py` we reach our default URL configuration file.
As was mentioned earlier: if there's a route in your Django project anywhere and it's not included here, **it doesn't exist**.

The way that you add desired urls is by adding to the given `urlpatterns` list.
By default you're given a whole set of URLs for Django's built-in site administration backend.
We can look into that later.

You're also given some very helpful doc strings that tell you exactly how to add routes to your Django project.
You need to provide a call to `path()` with at least two parameters (really you should have 3 for the default pattern):

- the actual route, as a string
- the view function (only ever a function!) that will handle that route
- the name of the route in your Django project; it makes it so much easier to reference your routes later on

Let's get our `HelloWorld` view back in here and attach it to the home route `"/"`.

```python
# django_todo/urls.py, after the big doc string
from django.contrib import admin
from django.urls import path

from django_todo.views import HelloWorld

urlpatterns = [
    path('', HelloWorld.as_view(), name="hello"),
    path('admin/', admin.site.urls),
]
```

Well this is different.
The route that we specified is just a blank string. Why does that work?
Django is already assuming that every path you're declaring begins with a leading slash.
You'e just specifying routes to resources after the initial domain name.
So, if your route isn't going to a specific resource and is instead just the home page, the route is just `""`.

The `HelloWorld` is being imported from that `views.py` file that we just created.
In order to do the import we're suggesting here, we need to update `settings.py` to include `django_todo` in the list of `INSTALLED_APPS`.
Yeah, it's a bit weird.
Here's one way to think about it.

`INSTALLED_APPS` refers to the list of directories or packages that you want Django to see as importable packages.
It's Django's way of treating its projects like packages without going through a `setup.py`.
We want the `django_todo` directory to be treated like an importable package, so we include that directory in `INSTALLED_APPS`.
As such, now any module within that directory is also importable.
And so we get our view.

The `path` function will ONLY take a view function as that second argument, not just a class-based view on its own.
Luckily, valid Django class-based views all include this `.as_view()` method.
Its job is to package all the goodness of the class-based view up into a view function and return that view function.
So you never have to worry about making that translation.
All you have to do is think about the business logic, letting Django and Django REST Framework handle the rest.

Let's crack this open in the browser!

Django comes packaged with its own local development server, accessible through `manage.py`.
Navigate to the directory containing `manage.py` and type

```
(django-someHash) $ ./manage.py runserver
```

When the `runserver` command is executed, Django does a check to make sure that the project is more or less wired-together correctly.
It's not fool-proof but it does catch some glaring issues.
It also notifies you if your database is out of sync with your code.
Undoubtedly ours is because we haven't done any committed any of application's stuff to our database but that's fine for now.
Visit `localhost:8000` to see the output of the `HelloWorld` view.

Huh.
That's not the completely-plain text data that we saw before in Pyramid, Flask, and Tornado.
When you use Django REST Framework, your HTTP response when viewed in the browser is this sort of rendered HTML, showing your actual JSON response in red.

But don't fret!
If you do a quick `curl` looking at `localhost:8000` in the command line you don't get any of that fancy HTML.
Just the content.

```
(django-someHash) $ curl localhost:8000
"Hello, world!"
```

Bueno!

Django REST Framework wants you to have access to a human-friendly interface when using the browser.
Makes sense; typically if JSON is being looked at in the browser it's because a human wants to check that it looks right, or get a sense of what the JSON response is going to look like as they design some consumer of an API.
It's a lot like what you'd get from a service like [Postman](https://www.getpostman.com/).

Either way, we know that our view is working and we can move on with our lives!

## Django - Models the Django Way

Before we move into thinking about the data model, let's recap what we've done thus far to build out this Django project.

1. Started the project with `django-admin startproject <project name>`
2. Update the `django_todo/settings.py` to use environment variabels for `DEBUG`, `SECRET_KEY`, and values in the `DATABASES` dict
3. Install `Django REST Framework` and add it to the list of `INSTALLED_APPS`
4. Create `django_todo/views.py` to include our first view class to say Hello to the World
5. Update `django_todo/urls.py` with a path to our new home route
6. Update `INSTALLED_APPS` in `django_todo/settings.py` to include the `django_todo` package

Alright, cool.
Let's create our data models now.
It's worth noting that if you want your project to talk to a Postgres database with *any* Python program you need to have `psycopg2` installed in your environment.

A Django project's entire infrastructure is built around data models.
It's written so that each data model can have its own little universe with its own views, its own set of URLs that concern it, even its own tests if you're so inclined to write them.

If you wanted to build a simple Django project you could circumvent this by just writing your own `models.py` file in the `django_todo` directory and importing it into your views.
However, we're trying to write a Django project the "right" way, so we should divide up our models as best we can into their own little packages The Django Way™.

The Django Way™ involves creating what are called Django "apps".
Django "apps" aren't separate applications per se; they don't have their own settings and whatnot.
However, they can have just about everything else you'd think of being in a standalone application:

- Set of self-contained URLs
- Set of self-contained HTML templates if you want to serve HTML
- One or more data models
- Set of self-contained views
- Set of self-contained tests

They are made to be independent so that they can be easily shared like standalone applications.
In fact, Django REST Framework is an example of a Django app.
It comes packaged with its own views and HTML templates for serving up our JSON.
We just leverage that Django App to turn our project into a full-on REST API with less hassle.

To create the Django app for our To Do list items, we'll want to use the `startapp` command with `manage.py`.

```
(django-someHash) $ ./manage.py startapp todo
```

The `startapp` command will succeed silently.
You can check that it did what's supposed to have done using the `ls` command in your command line.

```
(django-someHash) $ ls
Pipfile      Pipfile.lock django_todo  manage.py    todo
```

Look at that, we've got a brand new `todo` directory.
Let's look inside!

```
(django-someHash) $ ls todo
__init__.py admin.py    apps.py     migrations  models.py   tests.py    views.py
```

Here are the files that `manage.py startapp` created for you:

- `__init__.py` - empty; exists just so that this directory can be seen as a valid import path for models, views, etc.
- `admin.py` - not quite empty; used for formatting your models for this app in the Django admin, which we're not getting into in this article
- `apps.py` - not much work to do here either. Helps with formatting models for the Django admin
- `migrations` - this is a directory that'll contain snapshots of your data models, used for updating your database. This is one of the few frameworks that comes with database management built-in, and part of that is allowing you to update your database instead of having to tear it down and rebuild it with an external service
- `models.py` - this is where your data models will go. They don't _**absolutely**_ have to go here, but honestly the only reason why you wouldn't put them here is to give yourself (or others) a hard time when using your app
- `tests.py` - this is where your tests would go if you were to write them
- `views.py` - this is for the views you write that pertain to the models in this app. You don't have to write them here. You could, for example, write all your views in `django_todo/views.py`. It's here, however, so that you can separate your concerns easier. This becomes far more relevant when you have sprawling applications that cover many conceptual spaces.

Not created for you by `manage.py` is a `urls.py` file for this app.
However, you can just make that yourself.

Before moving forward we should do ourselves a favor and add this new Django app to our list of `INSTALLED_APPS` in `django_todo/settings.py`.

```python
# in settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'django_todo',
    'todo' # <--- the line was added
]
```

If we enter `todo/models.py` we'll see that `manage.py` already wrote a bit of code for us to get started.
Diverging from models were created in the Flask, Tornado, and Pyramid implementations, Django doesn't leverage a third party to manage database sessions or the construction of its object instances.
It's all rolled into Django's `django.db.models` submodule.

To create a model in Django you'll need to build a `class` that inherits from `models.Model`.
All of the fields that will apply to instances of that model should appear as class attributes.
Instead of importing columns and field types from SQLAlchemy like we have in the past, all of our fields will come directly from `django.db.models`.

```python
# todo/models.py
from django.db import models

class Task(models.Model):
    """Tasks for the To Do list."""
    name = models.CharField(max_length=256)
    note = models.TextField(blank=True, null=True)
    creation_date = models.DateTimeField(auto_now_add=True)
    due_date = models.DateTimeField(blank=True, null=True)
    completed = models.BooleanField(default=False)
```

While there are some definite differences between what Django needs from you and what SQLAlchemy-based systems needed from you, the overall contents and structure is more or less the same.
Let's point out the differences.

We no longer need to declare a separate field for an auto-incremented ID number for our object instances.
Django builds one for us unless we specify a different field as the primary key. 

Instead of instantiating Column objects that are passed data type objects, we just directly reference the data types as the columns themselves.

The `Unicode` field became either `models.CharField` or `models.TextField`.
`CharField` is for small text fields of a specific maximum length, whereas `TextField` is for any amount of text.

When the `TextField` needed to be able to be blank, it was specified in TWO ways.
`blank=True` says that when an instance of this model is constructed and the data that's being attached to this field is being validated, it's ok for that data to be empty.
This is subtly different from `null=True`, which says that when the table for this model class gets constructed, the column corresponding to `note` will allow for blank or `NULL` entries.
So to sum that all up, `blank=True` controls how data gets added to model instances while `null=True` controls how the database table holding that data got constructed in the first place.

The `DateTime` field grew some muscle and became able to do some work for us instead of us having to modify the `__init__` method for the class.
For the `creation_date` field we specify that `auto_now_add=True`.
What this means in a practical sense is that **when a new model instance is created** Django will **auto**matically record the date and time of **now** as this field's value.
That's handy!

When neither `auto_now_add` or its cousin `auto_now` are set to `True`, `DateTimeField` will expect data like any other field.
It'll need to be fed with a proper `datetime` object to be valid.
Here, `blank` and `null` are both set to `True` so that an item on the To Do list can just be an item to be done at some point in the future, with no defined date or time.

`BooleanField` just ends up being a field that can take one of two values: `True` or `False`.
Here, the default value is set to be `False`.

### Managing the Database

As was mentioned earlier, Django has its own way of doing database management.
Instead of us having to write...really any code at all regarding our database, we leverage the `manage.py` script that Django provided for us on construction.
It'll manage for us not just the construction of the tables for our database, but also any updates we wish to make to those tables _**without**_ having to blow the whole thing away!

Because we've constructed a _new_ model we need to make our database aware of it.
The first thing we need to do is put into code the schema that corresponds to this model.
The `makemigrations` command of `manage.py` will take a snapshot of the model class we built and all of its fields.
It'll take that information and package it into a Python script that'll live in this particular Django app's `migrations` directory.
**You will never have a reason to run these scripts yourself.**
It'll exist solely so that Django can use it as a basis to update your database table, or to inherit information from when you update your model class.

```
(django-someHash) $ ./manage.py makemigrations
Migrations for 'todo':
  todo/migrations/0001_initial.py
    - Create model Task
```

This will look at every app listed in `INSTALLED_APPS` and check for models that exist in those apps.
It'll then check the corresponding `migrations` directory for migration files and compare them to the models in each of those `INSTALLED_APPS` apps.
If a model has been upgraded beyond what the latest migration says should exist, a new migration file will be created that inherits from the most recent one.
It'll be automatically named, and also be given a message that says what changed since the last migration.

If it's been a while since you last worked on your Django project and can't remember if your models were in sync with your migrations, you have no need to fear.
`makemigrations` is an idempotent operation; your `migrations` directory will only have one copy of the current model configuration whether you run `makemigrations` once or twenty times.
Even better than that, if you run `./manage.py runserver` Django can detect that your models are out of sync with your migrations it'll just flat out tell you in colored text so that you can make the appropriate choice.

This next point is something that trips everybody up at least once: **creating a migration file does not immediately affect your database**.
When you ran `makemigrations` you prepared your Django project to define how a given table should be created and end up looking.
It's still on you to apply those changes to your database.
That's what the `migrate` command is for.

```
(django-someHash) $ ./manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions, todo
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying sessions.0001_initial... OK
  Applying todo.0001_initial... OK
```

When you actually apply your migrations, Django first checks to see if the other `INSTALLED_APPS` have migrations to be applied.
It checks them in roughly the order that they're listed.
You want your app to be listed last because you want to make sure that, in case your model depends on any of Django's built-in models, the database updates that are made don't suffer from dependency issues.

We have another model to build: the User model.
However, the game has changed a bit sine we're using Django.
So many applications require some sort of User model that Django's `django.contrib.auth` package just has its own built for you to use.
If it weren't for the authentication token that we require for our users, we could just move on ahead reinventing the wheel.

However, we need that token.
There's a couple ways that we can handle this.

- Inherit from Django's `User` object, making our own object that extends it by adding a `token` field
- Create a brand new object that exists in a one-to-one relationship with Django's `User` object, whose only purpose is to hold a token

Note that I didn't include a method that involved creating a new User object from the ground up or modifying Django's `User` object in any real way.
That's because there are certain parts of Django's infrastructure that involve that `User` object, particularly when it comes to user authentication.

I'm in the habit of building object relationships, so I'll go with the second option.
Let's call it a `Owner` as it basically has a similar connotation as a `User`, which is what we want

Out of sheer laziness I could just include this new `Owner` object in `todo/models.py`, but I'm going to refrain from that.
`Owner` doesn't explicitly have to do with the creation or maintenance of items on the task list.
Conceptually, the `Owner` is simply the *owner* of the task.
There may even come a time where I want to expand this `Owner` to include other data that has absolutely nothing to do with tasks.

Just to be safe, and to show that spinning up new Django apps is a small deal, let's make an `owner` app whose job it is to house and handle this `Owner` object.

```
(django-someHash) $ ./manage.py startapp owner
```

Don't forget to add it to the list of `INSTALLED_APPS` in `settings.py`.

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'django_todo',
    'todo',
    'owner'
]
```

In the interest of full disclosure, when I first wrote this bit I intended to call it "profile" but it conflicted with an existing Python module so here we are.

If we look at the root of our Django project, we now have two Django apps:

```
(django-someHash) $ ls
Pipfile      Pipfile.lock django_todo  manage.py    owner        todo
```

In `owner/models.py` let's build this `Owner` model.
As mentioned earlier, it'll have a one-to-one relationship with Django's built-in `User` object.
We can enforce this relationship with Django's `models.OneToOneField`

```python
# owner/models.py
from django.db import models
from django.contrib.auth.models import User
import secrets

class Owner(models.Model):
    """The object that owns tasks."""
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    token = models.CharField(max_length=256)

    def __init__(self, *args, **kwargs):
        """On construction, set token."""
        self.token = secrets.token_urlsafe(64)
        super().__init__(*args, **kwargs)
```

Here we're saying that the `Owner` object is linked to the `User` object, with one `owner` instance per `user` instance.
`on_delete=models.CASCADE` dictates that if the corresponding `User` gets deleted, the `Owner` instance it's linked to will also get deleted.

Now our `Owner` needs to own some `Task` objects.
It'll be very similar to the `OneToOneField` seen above, except that we'll stick a `ForeignKey` field on the `Task` object pointing to an `Owner`.

```python
# todo/models.py
from django.db import models
from owner.models import Owner

class Task(models.Model):
    """Tasks for the To Do list."""
    name = models.CharField(max_length=256)
    note = models.TextField(blank=True, null=True)
    creation_date = models.DateTimeField(auto_now_add=True)
    due_date = models.DateTimeField(blank=True, null=True)
    completed = models.BooleanField(default=False)
    owner = models.ForeignKey(Owner, on_delete=models.CASCADE)
```

Every To Do list task has exactly one owner, who can multiple tasks.
When that owner is deleted, anything task that they own goes with them.

Let's now run `makemigrations` to take a new snapshot of our data model setup, then `migrate` to apply those changes to our database.

```
(django-someHash) django $ ./manage.py makemigrations
You are trying to add a non-nullable field 'owner' to task without a default; we can't do that (the database needs something to populate existing rows).
Please select a fix:
 1) Provide a one-off default now (will be set on all existing rows with a null value for this column)
 2) Quit, and let me add a default in models.py
```

Oh no! We have a problem! What happened?
Well, when we created the `Owner` object and added it as a `ForeignKey` to `Task`, we basically necessitated that every `Task` requires an `Owner`.
However, the first migration we made for the `Task` object didn't include that requirement.
So, even though there's no data currently existing in our database's table, Django is doing a pre-check on our migrations to make sure that they're compatible and this new migration that we're proposing is not.

There's a couple ways to deal with this sort of problem:

1. Blow away the current migration and build a new one that includes the current model configuration
2. Add a default value to the `owner` field on the `Task` object

Option 2 wouldn't really make too much sense here; we'd be proposing that any `Task` that was created would, by default, be linked to some default owner despite one not even necessarily existing.
The sensible thing to do here would be option 1: destroy and rebuild.

Delete the migrations from the migrations directory, redo `makemigrations`.

```
(django-someHash) django $ ./manage.py makemigrations
Migrations for 'owner':
  owner/migrations/0001_initial.py
    - Create model Owner
Migrations for 'todo':
  todo/migrations/0001_initial.py
    - Create model Task
```

Woo! We have our models!
Welcome to the Django way of declaring objects.

For good measure, let's ensure that whenever a `User` is made, it gets automatically linked with a new `Owner` object.
We can do this using Django's `signals` system. 
Basically we say exactly what we intend: "when you get the signal that a new `User` has been constructed, construct a new `Owner` and set that new `User` as that `Owner`'s `user` field."
In practice that looks like:

```python
# owner/models.py
from django.contrib.auth.models import User
from django.db import models
from django.db.models.signals import post_save
from django.dispatch import receiver

import secrets


class Owner(models.Model):
    """The object that owns tasks."""
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    token = models.CharField(max_length=256)

    def __init__(self, *args, **kwargs):
        """On construction, set token."""
        self.token = secrets.token_urlsafe(64)
        super().__init__(*args, **kwargs)


@receiver(post_save, sender=User)
def link_user_to_owner(sender, **kwargs):
    """If a new User is saved, create a corresponding Owner."""
    if kwargs['created']:
        owner = Owner(user=kwargs['instance'])
        owner.save()
```

We set up a function that listens for signals to be sent from the `User` object built into Django.
Particularly, it's waiting for just after a `User` object has been saved.
This can come from either a new `User` or an update to an existing `User`; we discern between the two scenarios within the listening function.

If the thing sending the signal was a newly-created instance, `kwargs['created']` will have the value of `True`.
We only want to do something if this is `True`.
If it's a new instance, we create a new `Owner`, setting its `user` field to be the new `User` instance that was created.
After that we `save()` the new `Owner`.
This will commit our change to the database if all is well.
It'll fail if the data doesn't validate against the fields we declared.

Now let's talk about how we're going to access their data.

## Django - Accessing Model Data

In the Flask, Pyramid, and Tornado frameworks we accessed model data by running queries against some database session.
Maybe it was attached to a `request` object, maybe it was a standalone session.
Either way, we had to establish a live connection to the database and query on that connection.

This isn't the way that Django works.
Django by default doesn't leverage any third party to converse with the database, instead choosing to allow the model classes to maintain their own conversations with the database.

Every model class that inherits from `django.db.models.Model` will have attached a `objects` object.
This will take the place of the `session` or `dbsession` that we've become so familiar with.
Let's open the special shell that Django gives you and investigate how this `objects` object works.

```
(django-someHash) django $ ./manage.py shell
Python 3.7.0 (default, Jun 29 2018, 20:13:13) 
[Clang 9.1.0 (clang-902.0.39.2)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)
>>>
```

The Django shell is different from your normal Python shell in that it's aware of the Django project that you've been building and can do easy imports of your models, views, settings, etc. without you having to worry about installing a package.
We can access our models with a simple import.

```python
>>> from owner.models import Owner
>>> Owner
<class 'owner.models.Owner'>
```

Currently we have no `Owner` instances.
We can tell by querying for them with `Owner.objects.all()`.

```python
>>> Owner.objects.all()
<QuerySet []>
```

Anytime you run a query method on the `<Model>.objects` object, you'll get a `QuerySet` back.
For our intents and purposes right now it's effectively a `list`, and this `list` is showing us that it's empty.
Let's make an `Owner` by making a `User`.

```python
>>> from django.contrib.auth.models import User
>>> new_user = User(username='kenyattamurphy', email='kenyatta.murphy@gmail.com')
>>> new_user.set_password('wakandaforever')
>>> new_user.save()
```

If we query for all of our `Owner`s now, we should find Kenyatta.

```python
>>> Owner.objects.all()
<QuerySet [<Owner: Owner object (1)>]>
```

Yay we've got data!

## Django - Serializing Models

In the blog posts prior to this one, I didn't focus much on how the data for the objects that we created was going to be displayed.
Admittedly, I should have.

We're going to be doing something with our resources beyond just "Hello World", "here's a listing of routes", and "oh hey, you created a new resource, good for you!"
As such, we're going to want to see some sort of JSON-ified output that represents those resources well.
Taking that object's data and transforming it into a JSON object for submission across HTTP is a version of **data serialization**.
In serializing data, we're taking the data that we currently have and reformatting it to fit some standard, more-easily-digestible form.

If I were doing this with Flask, Pyramid, and Tornado, I'd create a new method on each model that I intended to give the user direct access to called `to_json()`.
The only job of  `to_json()` would be to return a JSON-serializable (i.e. numbers, strings, lists, dicts) dictionary with whatever fields I want displayed for the object in question.

It's probably look something like this for the `Task` object:

```python
class Task(Base):
    ...all the fields...

    def to_json(self):
        """Convert task attributes to a JSON-serializable dict."""
        return {
            'id': self.id,
            'name': self.name,
            'note': self.note,
            'creation_date': self.creation_date.strftime('%m/%d/%Y %H:%M:%S'),
            'due_date': self.due_date.strftime('%m/%d/%Y %H:%M:%S'),
            'completed': self.completed,
            'user': self.user_id
        }
```

It's not fancy, but it does the job.

Django REST Framework, however, provides you with an object that'll not only do that for you, but also validate inputs when you want to create new object instances or update existing ones.
It's called the [ModelSerializer](http://www.django-rest-framework.org/api-guide/serializers/#modelserializer).

Django REST Framework's `ModelSerializer` is effectively documentation for your models.
They don't have lives of their own if there are no models attached.
Their main job is to accurately represent your model and make the conversion to JSON thoughtless when your model's data needs to be serialized and sent over a wire.

Django REST Framework's `ModelSerializer` works best for simple objects.
As an example, imagine that we didn't have that `ForeignKey` on the `Task` object.
We could create a serializer for our `Task` that'd convert its field values to JSON as necessary with the following declaration:

```python
# todo/serializers.py
from rest_framework import serializers
from todo.models import Task

class TaskSerializer(serializers.ModelSerializer):

    class Meta:
        model = Task
        fields = ('id', 'name', 'note', 'creation_date', 'due_date', 'completed')
```

Inside our new `TaskSerializer` we create a `Meta` class.
`Meta`'s job here is just to hold information, or **metadata**, about the thing you're attempting to serialize.
Then, we note the specific fields that we want to show.
If we wanted to show all of the fields, we could just shortcut the process and use `'__all__'`.
We could, alternatively, use the `exclude` keyword instead of `fields` to tell Django REST Framework that we want every field except for a select few.
You can have as many serializers as you like, so maybe you want one for a small subset of fields and one for all the fields? Go wild here.

In our particular case, there is a relation between each `Task` and its owner `Owner` that must be reflected here.
As such, we need to borrow the `serializers.PrimaryKeyRelatedField` object to specify that each `Task` will have an `Owner`.
Its owner will be found from the set of all owners that exists.
We get that set by doing a query for those owners and returning the results that we want to be associated with this serializer: `Owner.objects.all()`.

```python
# todo/serializers.py
from rest_framework import serializers
from todo.models import Task
from owner.models import Owner

class TaskSerializer(serializers.ModelSerializer):
    owner = serializers.PrimaryKeyRelatedField(queryset=Owner.objects.all())

    class Meta:
        model = Task
        fields = ('id', 'name', 'note', 'creation_date', 'due_date', 'completed')
```

## Django - Connecting Models to Views

## Wrapping Up