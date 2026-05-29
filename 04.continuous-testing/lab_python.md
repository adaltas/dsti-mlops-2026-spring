
# Lab

Continuous testing

## Objectives

- Learn how to write and run unit tests for your functions
- Use test-driven development (TDD)

## Before starting

1. Create and activate virtual environment.
2. Install pytest in the newly created virtual environment.
3. Go to the [pytest doc](https://docs.pytest.org/en/stable/getting-started.html). Read how the files and test functions should be named, so pytest can discover them automatically.

## Prepare the environment

During testing you will write python functions and the tests that will test those functions. For this to work, files with tests need to be named correctly to be recognized by pytest, and functions need to be in the correct place to be found by Python. The organization and configuration of the project is important.

Below are two ways how this can be done. Both examples are based on the example from [Create your first test](https://docs.pytest.org/en/stable/getting-started.html).

NOTE: If running `pytest` doesn't work, try running `python -m pytest` instead.

### Factions and tests in the same file

1. Create a folder `unit_testing_simple`.
2. In the folder create a file `test_sample.py`.
3. Copy the example code from the website to the file:

```python
def func(x):
    return x + 1

def test_answer():
    assert func(3) == 5
```

4. Ensure you are in a terminal where your virtual environment is activated and go to the folder `unit_testing_simple`.
5. Verify you are in the correct directory by running `ls` (or `dir`) to list all the files in the folder. You should see the `test_sample.py`.
6. Run `pytest`. The output should look like the output below. **NOTE:** Your software versions are likely to be different than the below output.  

```
================================================================= test session starts ==================================================================
platform linux -- Python 3.9.12, pytest-7.2.0, pluggy-1.0.0
rootdir: /home/adaltas/Documents/teaching_tests/unit_testing/unit_testing_simple
collected 1 item

test_sample.py F                                                                                                                                 [100%]

======================================================================= FAILURES =======================================================================
_____________________________________________________________________ test_answer ______________________________________________________________________

>   ???
E   assert 4 == 5
E    +  where 4 = func(3)

/home/adaltas/Documents/teaching_tests/unit_testing_simple/test_sample.py:7: AssertionError
=============================================================== short test summary info ================================================================
FAILED test_sample.py::test_answer - assert 4 == 5
================================================================== 1 failed in 0.01s ===================================================================
```

7. Try to understand what it means. Consult the documentation as needed.

### Functions and tests are separated

Having the functions and the tests in the same file is not a good practice. It works well for short learning examples, but it leads to unreadable projects in real life. Therefore, we need to split the functions and the tests into separate files and directories to properly organize a project.

1. Create a folder `unit_testing_best_practice`.
2. Inside, create two subfolders on the same level: `src`, `test`. All the functions will be saved to the `src` folder and all the tests to `test` folder.
3. In `src`, create a file `sample.py`.
4. Inside, copy the function `func` from [Create your first test](https://docs.pytest.org/en/stable/getting-started.html).

```python
def func(x):
    return x + 1
```

5. In `test`, create a file `test_sample.py`.
6. Inside, copy the test `test_answer` from [Create your first test](https://docs.pytest.org/en/7.0.x/getting-started.html#create-your-first-test).

```python
def test_answer():
    assert func(3) == 5
```

7. Since we have two folders now, from which location will we run `pytest`? Why?
8. Pytest will always be run from the `unit_testing_best_practice/test`. To execute correctly, we need to include two information to `test_sample.py`:
	- import the module with our functions
	- point to the module location (do you know the difference between the absolute and relative path?)

Now the `test_sample.py` should look like this:

```python
import sys
# Always run from unit_testing_best_practice/test
sys.path += ['../src']

from sample import *

def test_answer():
    assert func(3) == 5
```

9. In the terminal, go to the `unit_testing_best_practice/test`.
10. Run `pytest`. The output should be the same as before.
11. Do you need to fix the function or the test to make the test pass (and the function to be correct)?

## Unit testing

Imagine you are developing an application, where users need to register. They need to fill in their username, email and password.

Your job is to write the functions that will accept those parameters from the user input (in command line) and test if they follow the criteria:

- user name:
  - not empty
  - no spaces
- password:
  - at least 8 characters
  - at least one number
  - at least one letter
  - at least one special character
- email:
  - contains `@` and `.`

Run tests by running `pytest` in the terminal.
**TIP:** You will need to find a way to automate or mock user input. Learn how to use pytest monkeypatch [here](https://docs.pytest.org/en/stable/how-to/monkeypatch.html#monkeypatching-functions).  

If you finished, you can check out [what else you can do with pytest](https://towardsdatascience.com/pytest-with-marking-mocking-and-fixtures-in-10-minutes-678d7ccd2f70).

## Test-driven development (TDD)

Develop a function that will test if the input is a prime number. Use TDD to do that. Follow [this tutorial](https://stackabuse.com/test-driven-development-with-pytest/) if you need help.

## Test coverage

How to know if you wrote enough tests or not: [Code coverage with pytest](https://medium.com/@sumanrbt1997/code-coverage-with-pytest-1f72653b0bf2)

## Troubleshooting

If you have problems running your tests:

1. Copy your error to browser (or describe it properly) and Google it. Since pytest is much used by the Python community, you have a big chance that somebody else already resolved the same issue.
2. When you open a link with a potential solution, don't just scroll through the page, but read. (Even if you don't understand it. You have to start somewhere.)
3. Try out the proposed solution.

## Where to go from here

If you have finished the lab before the end of class or want to explore testing in python further, consider any of the following options .

1. Utilize any of the skills we have started practicing in the mlops class and apply them to an existing code base you want to clean up and use to highlight your skills.
2. Start going through [Test Driven Development with Python here](https://www.oreilly.com/library/view/test-driven-development-with/9781098148706/) in the case that you still have an oreily subscription, or [here](https://www.obeythetestinggoat.com/) in the case you want to access the open source version of the book .
3. In the case that you don't have past projects to which you want to add testing, imagine some common functions you find yourself using in your projects (I'm looking at you python function I've written to strip the punctuation from strings a million times). Write tests for those and start to build a library of your own functions to use in future projects.
