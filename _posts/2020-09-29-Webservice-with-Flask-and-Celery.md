---
layout: post
title: A minimal webservice with FLASK and Backgroundjobs from CELERY
tag: Python Flask Microservice Backgroundjobs Celeris Redis
---

# Following Allong
> It is hard to outline how simple this all is yet brilliant this is


The main concept is the following we have a algorithm that repsresents some simulation and the main point is, that it is **slow**.  We want to built an **api** that takes values and returns the simulation. The **Problem** is that since the Simulation takes time, one request would block the whole api until it is done. **Solution:** we create **jobs** that can run at the background and our api takes the up when the results are ready.

## The Backend
The basisc structure is the following. We have an algorithm that takes inputs and returns the output of the simulation.
The api takes care of handling the inputs and asking for the simulations. Celery is a **job queue** that schedules how the jobs should be processed and returns the final outcome of the job. The api takes back the final results and returns them to the user. Redis is used as a **broker** betwenn the api and the queue and manages the asynchronous communication. (no clock cyle jsut tell me when you are ready and ill go on.)
The whole filesystem my look like rocketscience but it is just python and we can inspect each single step in the interpreter.

### The algorithm 
is rather boring, it does what is does and provides a result.
In our case a rather poor approximation for an intergral. it takes a function as a string e.g. "sqrt(4-xs**2)" and a,b,c,d the x and y coordinates limiting the are that should be inspected (Not important at all do not try to fully understand this)

### The flask api 
takes care of the incomming requests. we have:
* a PUT request that takes the arguments for new calculation and creates a new job for that
* a GET requests that lists all jobs
* a GET that displays the result of a specific job

### The celery part 
is basically just a small wrapper for the algorithm. It creates tasks/jobs that do whatever you want. (In our case doing the approximation of the Integral). 
```python
app = Celery(__name__, backend='rpc://', broker='redis://localhost:6379/')

## easier, if you don't care about exceptions:
# integrate = app.task(approx)

@app.task
def integrate(*args, **kwargs):
    try:
        return approx(*args, **kwargs)
    except Exception:
        return

```

By calling the `delay()` we tell `celery` that it is okay to schedule the job and not handle it directly. `Celery` then creates a `AsyncResult` object that has some attributes and methods. Some of them are `ready(), status, sucessfull()` which are helper classes to tell the current status of the task. After it is ready for pickup `ready()` returns `True`. Most importantly we want to get the result of the calculations, those can be accessed with `get()`. `get()` actually waits until the job is ready and then returns the result.
 Lets look at it in the interpreter (Note filename is worker.py)
```
from worker import integrate
task = integrate.delay(<args for integrate>)
task.read()
task.status
task.get()
```
**And this is all the magic about backgound jobs.**

### The redis part
runs completly in the background no files are related to this we just start the server with `redis-server` once and take the `PORT` it runs on to celery.

### Testing HTML Request with ease
Here I recommend the python package `httpie` which helps to test requests way more intuitive. Just type e.g.
`http GET localhost/port/url a:=0 b="hello"` in the command line and it will process a GET request with the respective arguments.
**Note:** if we want to transmit numbers then we use `:=` and for strings `=`. in addition whitespaces can cause errors `a := 0 BAD a:=0 GOOD`.

PUT and POST requests follow the same logic and the responses are nicely printed to the commandline.

### Back to the API
we initialize a `TASKS` Dictionary in which we store all our jobs. **Note:** By just defining a variable, all values will be lost on restart but this is intended for this project.

```python
from flask import Flask, request, jsonify
from worker import integrate
app = Flask(__name__)

TASKS={}
```
To command a new simulation we first collect and unpack the arguments. (Those together within the request.) Afterwards we take a look how many elements are currently in the `TASKS` and define our id consecutive and ask `celery` to start a new job for us. We then finish the request by returning the task_id as a response. This is important since the caller can use this id to find the results after they are done.
```python
@app.route("/", methods=["PUT"])
def put_task():
    #unpack all values
    f = request.json["f"]
    a = request.json["a"]
    b = request.json["b"]
    c = request.json["c"]
    d = request.json["d"]
    #id size is missing then use 100 as default
    size = request.json.get("size",100)

    task_id = len(TASKS)
    # create a job for the simulation
    TASKS[task_id] =  integrate.delay(f,a,b,c,d,size)
    response ={"result":task_id}

    return jsonify(response)
```

then we define a route to display all available requests, together with their current status.
```python
@app.route("/", methods = ["GET"])
def list_tasks():
    tasks = {task_id: {'ready': task.ready()}
             for task_id, task in TASKS.items()}
    return jsonify(tasks)
```
After receiving out `id` we can make a GET request with our id. This searches the `AsynResult` object int the `TASKS` list and receives the result. The check for readyness is optional here since the `get()` method would wait until it is finish anyways. After the job is done the results is appened to the response and returned as json object
```python
@app.route('/<int:task_id>', methods=['GET'])
def get_task(task_id):
    response = {'task_id': task_id}

    task = TASKS[task_id]
    if task.ready():
        response['result'] = task.get()
    return jsonify(response)

```
```python
if __name__ == "__main__":
    app.run(debug=True)
```