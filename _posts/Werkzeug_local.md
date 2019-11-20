In Werkzeug, there are three very interesting class Local, LocalProxy and LocalStack. It is also key to understand Flask's thread model.

Flask uses multithreading for serving. During the code, we can often see statements like

```python
from flask import request
from flask import Flask
app = Flask(__name__)

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        return do_the_login()
    else:
        return show_the_login_form()
```

The `request` contains your request details, including method, url parameters, cookies and other things.

This is achieved with Werkzeug's Local classes.
The local class is very similar to the ThreadLocal, but it adds extra support for greenlets. With local, you can support multithreading without managing the local variables within each thread.

Some code samples
```
local_values = Local()
def task(num):
    local_values.name = num
    import time
    time.sleep(1)
    val = str(local_values.name) + threading.current_thread().name + '\n'
    print(val)


for i in range(20):
    th = threading.Thread(target=task, args=(i,),name=' thread %s' % i)
    th.start()
```

LocalProxy provides a dynamic access to a local object. Why you want to use a proxy object for it? This doc in Chinese provides [better info](https://www.jianshu.com/p/3f38b777a621).

```
class Student:
    def __init__(self):
        self.score = 0

    def set_score(self, s):
        self.score = s

    def get_score(self):
        return self.score

    def __repr__(self):
        return f"score = {self.score}"

l = Local()
# student is a localproxy.
student = l('student')

def assign_student(i):
    s = Student()
    s.set_score(i)
    l.student = s
    # student._get_current_object()
    score = student.get_score()
    print(student)
    print(score)

assign_student(100)
```

