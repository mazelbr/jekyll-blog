## Add new app
To add a new app go to the project directory and create a new app with
```python manage.py startapp appname```
This sets up the basic file strucutre for your new app. Next you should **register** your app at the project by adding the appname to the `project/setup.py` `INSTALLED_APPS` list.
This can be done by simply adding the appname at the end of the list e.g. `INSTALLED_APPS = [...,'appname,']`

Afterwards one would like to redirect rounts that aim for the new app to the `urls.py` file of the application.
Create a `urls.py` file at the app directory and at the `project.urls.py` file add somewhat like this to the urlpatterns:
```
urlpatterns = [..., include("appname/", "appname.urls")] 
```
It is good convention to name your app by defining a `app_name` variable in the `urls.py` file that equals the appname. this helps 
us later especially with easy links and anchors.

A boilerplate for the urls.py file can be found here:
```
from django.urls import path
from . import views

app_name ="some_name"
urlpatterns = [
    path("", views.index, name ="index"),
]
```
## Adding view templates
To make the rendering of html responses easier we can use html templates that can dynamically be filled with variables and stuff.
For that we create inside the app directoy folders and file such that `templates/appname/methodname.html` is accesible.

After that you can ue the `render(request, template_path,context)` to access and render your views. the template path should be like "appname/methodname-html"
and the context is a dictionary, containing all variables as `{varname: value}`

NOTE: Your app has to be registered at the project for django to find your view. otherwise it raises a "Template not Found" error 

Boilerplate for HTML
```
<!DOCTYPE html>
<html lang="en">
    <head>
        <title>Hello World</title>
    </head>
    <body>

    </body>
</html>


```
## Template inheritance
Inheriting from templates helps you to not redo the samte html code over and over and it is super simple.
Just create a `addname/templates/appname/layouts.html` file and type the basic strucutre of your html document.
If you want to replace the body with the child templates insert into the body of your layout document:
```
<body>
{%block blockname%}
{%endblock%}
<\body>
```
to fill the block with life let your child templates inherit from this with:
```
{%extends "appname/layout.html"}

{%block blockname%}
//Content
{%endblock%}
```

## Adding new functionalities
1. Adding new functions in the view file:
    to create a new function we create a new function in the route file that takes a `request` object.
    we can basically do whatever we want inside the function and would afterwards want in most cases to return a HTTPresponse by rendering a template.
    
    ```
    def index(request):
        return render(request,"tasks/index.html", {'t': tasks})
        return HttpResponse("Hello World")
    ```
## Adding Links and Anchors
Adding explizit links in python is super easy and nice since then you do not have to correct all links manually when something changes.
Even though we have to prepare it a little bit. 
First of all we have to give our app a name this can be done in the `app/url.py` file. Here we just define a variable called `app_name`.
Since we already named our `routes` with the `name` attribute we can easily access all routes with the pattern `appname.routename`.
```
% url.py
...
app_name = "somename" #here we are defining the appname
urlpatterns=[
    path("", views.index, name="index") #here we are defining the routename

]
% templates/appname/add.html
...
<a href={%url 'app_name:routename' %}> Click me <\a>
...
```
## Adding forms that actually do something
To use forms we first of all have to create one. This can be done in plain html ot more preferably with django forms. 
```
<form action="">
    <input type="text", name="task">
    <input type="submit">
</form>
```
The key part is the `action` attribute of the form. Here we can define **where to form should post to**. Here we use the same logic as for links.
We just tell django which route and therefor which method we should be the receiver of the form. Furthermore we set the `method` attribute to `POST` since it is a POST-Request. 
```
<form action="{% url 'appname:routename' %}" method="post">
    <input type="text", name="task">
    <input type="submit">
</form>
```
By now after submitting the form the defined route (e.g. add) receives the `request` object. The complete object can be instpected in the django docs. The most important attributes for now are the `request.method` and the `request.POST`. Since we chose the `method='post'` it is easy to guess thath the `request.method == 'post'`. Then we might me interested how we can access the elements of the form. Luckily the `request` object stores all of them as a dictionary `{name : value}`in the `POST` attribute. Here `request.POST = {'name': 'submitted name'}`.
Note: a safe way of looking up elements in dictionaries is `dict.get(key,"")` which returns either tha value of key or an empty string if the Key is not in the dict. But it Does not raise an error.

An example of how Post attributes can be accessed can be found here
```
def add(request):
    context = {'value': ""}
    if request.method == "POST":
        context['value'] = request.POST.get("task","") 
    return HttpResponse(context)
```

## Simplify forms with Django Forms
Coding HTML forms can be ugly, fortunately django privides a built in class for that.
First import if with `from django import forms` now inside the forms create a `class` that inherits from `forms.forms`
Django provides plenty of predefined field types such as `CharField(), DateField(), FileField(), ImageField(),...`.
Those are predefined classes and can take additional arguments that simplify the validations. e.g. `CharField(required=True,max_length=8)`.
The fields are assigned to variables that than can be accessed in the request object.

If we want to actually use this form we have to create a new instance insinde the `view` methods. Lets look at an example

```
% app/views.py
from django import forms

class NewForm(forms.Form):
    task = forms.CharField(label="New Task")

...
def add(request):
    return render(request, "some/view.html", {"form" : NewForm()})
```
Here we make a form instance available in the view. If we want to render it we got to the view and load it into a html form.
```
<form action="{% url 'task:add'" %}" method = 'post>
    {% csrf_token %}
    {{form}} //load the form
    <input type="submit">
<\form>
```

## How to Process Forms in the views.
A basic usecase is often to check if the form is valide and if not ask the user to correct his inputs. If the inputs are correct we want to process the data for e.g. creating/updating/deleting data.
**Reminder**: You have access tp the `request.method` and the `request.POST` attributes.
First of all we want to check for the type of request. If the request is get we do not have data to process, we then want to display the plain form. if the request is POST we want to check if it is valid and either reject or process the data.

```
def add(request):
    if request.method == 'POST':
        data = request.POST #access POST dictionariy
        form = NewForm(data) #construct a form filled with the submitted data
        if form.isvalid(): #check server side validation
            task = form.cleaned_data["task"] #the "task" comes from the variable name in the form class
            tasks.append(task)
            return HttpResponseRedirect(reverse("tasks:index"))
        else:
            return render(request, "task/add.html", {"form": form}) #render add view with the form and preserve the user input

    else:
        # if GET then render new plain form
        return render(request, "tasks/add.html", {"form": NewForm()})
```
**Note:** We used `reverse` and `HttpResponseRedirect` and therfore have to import them as well.

One major benefit of django forms is that they provide both clientside and serverside validations.

## How to access local files - SIMPLE
A simple way of using local files is to store them in the base directory or a folder in the base directory.
Since Django provides a variable called `BASE_DIR` in the `settings.py` it is especially easy to access those files.
Just read them like you would usually reas files in python. Inside the `views.py`
```
from django.conf import settings
import os

filepath = os.path.join(BASE_DIR, "filename.txt")
with open(filepath, 'r') as file:
    data = file.read().replace("\n","")
```
And that is literally all the magic. You now can access the the data however you want, e.g. in a method:
```
def index(request):
    return HttpResponse(data)
```