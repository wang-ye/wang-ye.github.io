I used to set up Anaconda for a Python and Pip environment. Recently I learned that starting from Python3.5, Python has builtin support for virtual environment with package [venv](https://docs.python.org/3/library/venv.html)!

Here is how to set up the venv:

```shell
python3 -m venv /path/to/new/virtual/environment
source /path/to/new/virtual/environment/bin/activate
```

Then, you can call pip to install all your packages!

```shell
pip install -r requirements.txt
```
