# CeleryWithDjango
A tutorial for setup the asynchronous task in the django server using celery with redis as task manager 

In this tutorial I am setting up an empty project and app (as this is not the django tutorial I am not explaining django here).

The project structure is like below
```
│   manage.py
│
├───AppWithAsyncTask
│   │   admin.py
│   │   apps.py
│   │   models.py
│   │   tests.py
│   │   views.py
|   |   urls.py
│   │   __init__.py
│   │
│   └───migrations
│           __init__.py
│
└───CeleryAsyncTaskForDjango
    │   settings.py
    │   urls.py
    │   wsgi.py
    │   __init__.py
```
I have followed [celery documentation](https://docs.celeryproject.org/en/latest/django/first-steps-with-django.html) for setting up the project.

## Step1: Add the required files

### 1. create celery.py
create `celery.py` file inside the project folder, here it is `CeleryAsyncTaskForDjango` and paste the below content
```python
from __future__ import absolute_import
import os
from celery import Celery
from django.conf import settings

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'CeleryAsyncTaskForDjango.settings')
#https://stackoverflow.com/questions/45744992/celery-raises-valueerror-not-enough-values-to-unpack
#os.environ.setdefault('FORKED_BY_MULTIPROCESSING', '1')
app = Celery('CeleryAsyncTaskForDjango')

# Using a string here means the worker will not have to pickle the object when using Windows.
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks(settings.INSTALLED_APPS)


@app.task(bind=True)
def debug_task(self):
    print('Request: {0!r}'.format(self.request))
```

**Note**: For the machines which doesn't support fork(), we need to add the line `os.environ.setdefault('FORKED_BY_MULTIPROCESSING', '1')` to celery.py 

### 2. add celery app to __init__.py

open the `__init__.py` file in `CeleryAsyncTaskForDjango` and paste below content

```python
from __future__ import absolute_import

# This will make sure the app is always imported when Django starts so that shared_task will use this app.
from .celery import app as celery_app
__all__ = ('celery_app',)
```
### 3. create tasks.py
The tasks files can be created inside the application, here I am creating in `AppWithAsyncTask`. We can place the task in the `tasks.py` file.

## Step2: Verify program execution time without celery
For making the code simple, I have added sleep to the view and response is taken as timestamp like below
```python
class SampleAPI(APIView):

    def get(self, request):
        requets_time = timezone.localtime()
        sleep(10)
        return Response({"Request time": requets_time,"Response time": timezone.localtime()})
```
``` json
{
    "Request time": "2020-08-10T18:30:39.362125Z",
    "Response time": "2020-08-10T18:30:49.362436Z"
}
```

Now lets make this larger/time consuming process as asynchronous, so it will run as asynchronous process

## Step3: Adding celery config

Since we haven't started the broker with the celery like `app = Celery('CeleryAsyncTaskForDjango', broker='redis://127.0.0.1:6379/')`, we need to configure the settings. we can adjust these setting in settings.py file

**Note** : We have used the `namespace='CELERY'` for the config, so we need to define all the config using `CELERY_`
```python
CELERY_BROKER_URL = "redis://localhost:6379"
CELERY_RESULT_BACKEND = "redis://localhost:6379"
CELERY_ACCEPT_CONTENT = ['application/json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
```

Add below code to `tasks.py` file
```python
from django.utils import timezone
from celery.decorators import task
from time import sleep

@task(name="async_model_task")
def async_model_task():
    with open("output.txt",'w') as f:
        f.write("\ntask started: {}\n".format(timezone.localtime()))
        sleep(10)
        f.write("task finished: {}".format(timezone.localtime()))
    return ""
```

Now change `views.py` to call the delay method like below
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from AppWithAsyncTask.tasks import async_model_task
from django.utils import timezone
from time import sleep

# Create your views here.

class SampleAPI(APIView):

    def get(self, request):
        requets_time = timezone.localtime()
        async_model_task.delay()
        return Response({"Request time": requets_time,"Response time": timezone.localtime()})
```

Now start the celery worker in new command prompt and hit the same api.
```
celery -A CeleryAsyncTaskForDjango worker -l info
```
once the worker starts, we will see the output like below
```
- ** ---------- [config]
- ** ---------- .> app:         CeleryAsyncTaskForDjango:0x21d59b84358
- ** ---------- .> transport:   redis://localhost:6379//
- ** ---------- .> results:     redis://localhost:6379/
- *** --- * --- .> concurrency: 4 (prefork)
-- ******* ---- .> task events: OFF (enable -E to monitor tasks in this worker)
--- ***** -----
 -------------- [queues]
                .> celery           exchange=celery(direct) key=celery


[tasks]
  . CeleryAsyncTaskForDjango.celery.debug_task
  . async_model_task
```

## Let's check the execution
 Response from api
 ```json
{
    "Request time": "2020-08-10T19:03:18.852483Z",
    "Response time": "2020-08-10T19:03:25.318374Z"
}
 ```
Log in the output.txt
```
task started: 2020-08-10 19:03:25.326371+00:00
task finished: 2020-08-10 19:03:35.334847+00:00
```

So we achieved the asynchronous process in Django using celery with redis as backend

Finally project tree is like below
```
│   manage.py
│   output.txt
│
├───AppWithAsyncTask
│   │   admin.py
│   │   apps.py
│   │   models.py
│   │   tasks.py
│   │   tests.py
│   │   urls.py
│   │   views.py
│   │   __init__.py
│   │
│   ├───migrations
│   │       __init__.py
│
└───CeleryAsyncTaskForDjango
    │   celery.py
    │   settings.py
    │   urls.py
    │   wsgi.py
    │   __init__.py

```