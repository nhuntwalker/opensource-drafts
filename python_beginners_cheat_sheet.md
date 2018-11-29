# Opensource.com: Python 3.7 Beginner's Cheat Sheet

Python (v3.7.0) is a powerful programming language with a wide variety of uses.
But even with it's large community and diverse menu of extensions, there's plenty built into the language itself that even the most novice programmer can find useful.
Here's a few of those built-in pieces:

---

Print a string, set of strings, or object representation(s) to the console.

```python
>>> print('any string')
any string
>>> print('string one', 'string two')
string one string two
>>> print([1, 2.0, 'three', ['f','o','u','r']])
[1, 2.0, 'three', ['f', 'o', 'u', 'r']]
```

---

Return the number of items contained within a string or object.
If the item doesn't keep track of what it holds, you'll get an error.

```python
>>> my_list = [41, 33, 48, 71, 60, 26]
>>> len(my_list)
6
```

---

View the documentation for any function or object that has it. To leave that view, hit the "q" key

```python
>>> help(len)
Help on built-in function len in module builtins:

len(obj, /)
    Return the number of items in a container.
(END)
```

---

Create an iterable of integers.
You can either get sequential numbers starting at 0 up to (but not including) some higher number by providing one integer as an argument, set the bounds of your numbers by providing two integers, or set the bounds and the space between numbers with three integers.

```python
>>> range(4)
range(0, 5)
>>> range(1, 10)
range(1, 10)
>>> range(2, 200, 4)
range(2, 200, 4)
```

You can't access these numbers by index like a list.
You need to loop through the `range` to access them.

---

Another function useful in loops.
If you want to iterate over an iterable object (e.g. a list) and access not just the item but the item's index within the container. use `enumerate`

```python
>>> countries = ['Turks & Caicos', 'Grenada', 'Vanuatu', 'Lebanon', 'Barbados']
>>> for idx, item in enumerate(countries):
...     print(f'{idx} - {item}')
...
0 - Turks & Caicos
1 - Grenada
2 - Vanuatu
3 - Lebanon
4 - Barbados
```

---

Sort any iterable that contains objects that are all of the same type (as long as they can be compared to each other) without mutating the original object.

```python
>>> sorted([64, 99, 69, 26, 70, 53, 11, 42, 99, 5])
[5, 11, 26, 42, 53, 64, 69, 70, 99, 99]
```

---

Open any file as text for reading or writing.
By default the file is opened to read.
If reading, the file must actually exist at the specified absolute or relative path.

```python
>>> with open('some_text.txt') as f:
...     f.read()
...
'Whatever text was in the file'
>>> with open('writeable_file.csv', 'w') as f:
...     f.write('one,two,three')
...
```

---

Find out what type of object anything is with `type`

```python
>>> type([])
<class 'list'>
>>> type('random text')
<class 'str'>
```

---

Check if an object is an instance of some built-in type or user-created `class` with `isinstance`

```python
>>> isinstance('potato', str)
True
>>> isinstance('potato', list)
False
```

---
