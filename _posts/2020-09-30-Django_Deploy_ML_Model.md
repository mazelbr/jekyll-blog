---
layout: post
title:  "How to Deploy a ML Model with Django"
tag: ML Deploying Django Python
---

# Using Django to Deploy a ML-Model
## Setup
This is a quick guide to deploy models to Django.
We asume that you already got a Django project running and start with creating a new app.
inside the `project dir`
```bash
python manage.py startapp ml
```
Dont forget to register you app at the project and set the routing in the `project.urls.py` file to redirect to the local ones
```python
% project.settings.py
#append app name to the installed apps
INSTALLED_APPS = [
    ...
    'ml',
]

% project/urls.py
 urlpatterns = [
     ...
     path("ml/", include("ml.urls"))
 ]
% app/urls.py
from django.urls import path
from . import views

app_name = "ml"
urlpatterns = [
    path("", views.index, name ="index"),
]
```
## Build Form for input
### 1. Create `app/templates/app/index.html` file and folders and fill it with the basic html strucutre

### 2. Create a index method in `app.views` and make sure that it cann access the template
```python
%app/view.py
from django.shortcuts import render
from django.http import HttpResponse

def index(request):
    return render(request, "ml/index.html")
```
### 3. Add a Form to the view and render it at the template
At the views create a Form class that inherits from `django.forms.Form` and add a `CharField`
```python
% app/views.py
from django import forms

#create new form type
class TextForm(forms.Form):
    input = forms.CharField(required = True)

# make a new form instance available at the template
def index(request):
    return render(request, "ml/index.html", {"form": TextForm()})
```
{% raw %}
```html
%ml/index.html
...
<form action= "{% url 'ml:index' %}" method='POST'>
    {% csrf_token %}
    {{form}}
    <input type="submit"> 
</form>
...
```
{% endraw %}

### 4. Handle the POST request of the form
If a the `index` view is accessed via Post we want to check and process the data, if the `method` is GET we want to render a plain form

```python
def index(request):
    #initialize conext
    context = {"form" : TextForm(), "output": ""}
    if request.method == "POST":
        # create a form and fill it with the user input
        form = TextForm(request.POST)
        
        if form.is_valid():#if the form is valid we can start to process the input
            input_string = form.cleaned_data['input']
            input_string = np.array(input_string)
            context["output"] = str(input_string)
            return render(request, "ml/index.html", context)

        else: # if its not valid we want to render the form with the user input
            context["form"] = form
            render(request, "ml/index.html", context)
    #if the Request is GET then just load a plain form
    else:
        return render(request, "ml/index.html",context) 
```

5. Load an import classifier/pipeline and make a prediction
We assume here that the classifier already has been trained and exported with `joblib`
```
from joblib import dump, load
dump(pipe, 'clf.joblib') 
clf2 = load('clf.joblib') 
```
we create a `model` folder in the base directory of the `project` and inside there we store the serialized file.
Luckily Django provides a `BASE_DIR` variable, which can be imported with `from django.config import settings` and accessed with `settings.BASE_DIR`. The further procedure is basically the same as in plain python. We create a path that points to the file, load it and use it.
```
%views.py
import os
from joblib import dump, load

path_name = os.join(settings.BASE_DIR, "models", "clf.joblib")
clf = load(path_name)
```
By now the classifier is loaded and can be used for e.g. predictions. Note that the input is expected to a `numpy.array` and a common error at this stage was `cannot iterate over 0d-array` this was cause by storing the input string directly in `numpy.array("Input string")` but the correct way would be to put the string inside a list and therfore `numpy.array(["Input string"])`

**Note:** for predicting with the classifier it is NOT necesarry to import sklearn or any other moduls it works straight out of the box.

The final code could look then somewhat like this:
```python
lass TextForm(forms.Form):
    input = forms.CharField(required=True)

path_name =  os.path.join(settings.BASE_DIR, 'models','clf.joblib')
clf2 = load(path_name) 
#with open(path_name, 'r') as file:
#    data = file.read().replace('\n', '')

def index(request):
    #initialize conext
    
    context = {"form" : TextForm(), "output": "", "prediction":{}}
    if request.method == "POST":
        # create a form and fill it with the user input
        form = TextForm(request.POST)
        
        if form.is_valid():#if the form is valid we can start to process the input
            input_string = form.cleaned_data['input']
            input_string = np.array([input_string])
            prediction = clf2.predict(input_string)

            context["prediction"]["class"] = prediction[0]

            i = list(clf2["NB"].classes_).index(prediction[0])
            context["prediction"]["proba"] = clf2.predict_proba(np.array(["Hi my name is"]))[0][i]

            context["form"] = form
            return render(request, "ml/index.html", context)

        else: # if its not valid we want to render the form with the user input
            context["form"] = form
            render(request, "ml/index.html", context)
    #if the Request is GET then just load a plain form
    else:
        return render(request, "ml/index.html",context)
```
And in the views:
{% raw %}
```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <title>Hello World</title>
    </head>
    <body>
        <h1>Hi</h1>
        <form action= "{% url 'ml:index' %}" method='post'>
            {% csrf_token %}
            {{form}}
            <input type="submit"> 
        </form>
        {% if prediction %}
        <p>Prediction: {{prediction.class}} Probability: {{prediction.proba}}</p>
        {% else %}
        <p>No predictions yet</p>
        {% endif %}


        <p>{{output}}</p>
    </body>
</html>

```
{% raw %}




