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

Not created for you by `manage.py` is a `urls.py` file for this app, however you can just make that yourself.

## Django - Managing the Database

## Django - Connecting Models to Views

## Wrapping Up