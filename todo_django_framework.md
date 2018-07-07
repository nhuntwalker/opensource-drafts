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
- `STATIC_URL` this is part of a larger body of settings for when you're serving static files. In our particular case where we'll be building a REST API we won't need to serve static files, so we can ignore it. In general though, this sets the root path after the domain name for every static file. So, if I had a logo image to serve it'd be `http://<domainname>/<STATIC_URL>/logo.gif`



## Django - Routes and Views

## Django - Models the Django Way

## Django - Managing the Database

## Django - Connecting Models to Views

## Wrapping Up