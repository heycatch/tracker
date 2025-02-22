tracker
-------
A time machine for debugging pesky stateful errors.

Motivation
----------
It's not always clear what lines of code introduce ugly bugs.  By walking through modifications to an object's state step by step, `tracker` makes it simpler to debug where the flaws in your logic lie.

Installation
------------

~~Tracker is conveniently available via pip:~~

```bash
pip install tracker
```

Pip package have errors, install via git clone and setup.py

```bash
git clone git@github.com:heycatch/tracker.git
python setup.py install
```

To ensure Tracker is properly installed, you can run the unittest suite from the project root:

```bash
nosetests -v
```

Usage
-----
Tracker uses class decorators to specify a list of attributes to monitor as a program executes.  Let's walk through an example with a bit of code designed to implement a simple balance sheet and see how `tracker` might help hunt down a bug in some sample code.  See the `examples` directory for runnable code snippets.


```python
from tracker import tracked

@tracked('balance')
class BalanceSheet(object):

    def __init__(self, balance=0):
        self.balance = balance

    def add_expense(self, value):
        self.balance -= value

    def deposit(self, value):
        self.balance += value
```

Now let's add some simple erroneous code to demonstrate the usefulness of `tracker`. Let's imagine for a minute that a second developer working on the balance sheet project misunderstood how the `BalanceSheet` class was intended to work, added their own syntax for direct modification of the `balance` attribute, and also included calls to the BalanceSheet's methods.  In other words, our second developer is doubling the amount of every expense and deposit.

```python
def add_expense(sheet, value):
    sheet.balance -= value
    sheet.add_expense(value)

def deposit(sheet, value):
    sheet.balance += value
    sheet.deposit(value)
```

So when the second developer goes to run a routine deposit, they see that the balance is not what they expected.

```python
from balance_sheet import BalanceSheet, add_expense, deposit

sheet = BalanceSheet()
deposit(sheet, 100)

print(sheet.balance)
```

Using tracker's replay functionality, we can rewind and see what lines of code changed the sheet's balance and help our second developer uncover their simple mistake.


```python
for snapshot in sheet.replay():
    print(snapshot)
```

which outputs:

```
{'balance': 0}
--------------------------------------------------------------------------------
balance_sheet_debugging.py:3
  2       
  3       sheet = BalanceSheet()
  4       deposit(sheet, 100)

/home/m/Projects/Tracker/examples/balance_sheet.py:8
  7           def __init__(self):
  8               self.balance = 0
  9       


{'balance': 100}
--------------------------------------------------------------------------------
balance_sheet_debugging.py:4
  3       sheet = BalanceSheet()
  4       deposit(sheet, 100)
  5       print sheet.balance

balance_sheet.py:23
  22      def deposit(sheet, value):
  23          sheet.balance += value
  24          sheet.deposit(value)


{'balance': 200}
--------------------------------------------------------------------------------
balance_sheet_debugging.py:4
  3       sheet = BalanceSheet()
  4       deposit(sheet, 100)
  5       print sheet.balance

balance_sheet.py:24
  23          sheet.balance += value
  24          sheet.deposit(value)
  25      

balance_sheet.py:14
  13          def deposit(self, value):
  14              self.balance += value
  15      
```

Each `Snapshot` object in `sheet.replay()` has a `state` attribute (i.e. `{'balance': 200}`) and a `stack` attribute (an array of `Frame` objects).  Each `Frame` object has a `file` attribute and a `line` attribute which point to the line of code which triggered the modification.  For dead simple debugging, the `__str__` method of the `Snapshot` object outputs a human readable represenations of those values.

Now we can easily walk through the code that caused the double deposit and follow how the `balance` attribute changed value over time.


How does it work?
-----------------
Tracker patches `__setattr__` and `__setitem__` in order to record changes to variables that you've placed on a watch list.
Whenever a change is caught, the `inspect` library is used to capture the lines of code that caused the modification.  Tracker is barely over 100 lines of code long, so I encourage you to read the source for the gritty details.

Contributions
-------------
Tracker is a work in progress, so pull requests, critical feedback, github issues, and feedback of any kind is welcome.
