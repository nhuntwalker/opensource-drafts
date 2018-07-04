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

The directory that it created, in my case `django_todo`, represents **the configuration root for your project**.
Let's dig into that now.

## Django - Configuration

## Django - Routes and Views

## Django - Models the Django Way

## Django - Managing the Database

## Django - Connecting Models to Views

## Wrapping Up