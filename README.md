Python Elevator Challenge
=========================

So You Think You Can Program An Elevator
----------------------------------------

Many of us ride elevators every day. We intuitively feel like we have a good grasp on them. We understand the how elevators decide where to go. But say you wanted to express this algorithm in code. How would you go about it?

Test Harness
------------

Lets consider a simplified model of an elevator. Like all elevators, it can go up or down. We define constants for these. This elevator also happens to be in a building with six floors.

    >>> UP = 1
    >>> DOWN = 2
    >>> FLOOR_COUNT = 6

We will make an `Elevator` class that simulates an elevator. This will delegate to another class which contains the elevator business logic, i.e. deciding what the elevator should do. Your challenge is to implement the business logic.

### User actions

A user can interact with the elevator in two ways. She can call the elevator by pressing the up or down  button on any floor, and she can select a destination floor by pressing the button for that floor on the panel in the elevator. Both of these actions get passed straight through to the logic delegate.

    >>> class Elevator(object):
    ...     def call(self, floor, direction):
    ...         self._logic_delegate.on_called(floor, direction)
    ... 
    ...     def select_floor(self, floor):
    ...         self._logic_delegate.on_floor_selected(floor)

### Elevator actions

The logic delegate can respond by setting the elevator to move up, move down, or stop. It can also read the current floor and movement direction of the elevator. These actions are accessed through `Callbacks`, a mediator provided by the `Elevator` class to the logic delegate.

    >>> class Elevator(Elevator):
    ...     def __init__(self, logic_delegate, starting_floor=1):
    ...         self._current_floor = starting_floor
    ...         print "%s..." % starting_floor,
    ...         self._motor_direction = None
    ...         self._logic_delegate = logic_delegate
    ...         self._logic_delegate.callbacks = self.Callbacks(self)
    ... 
    ...     class Callbacks(object):
    ...         def __init__(self, outer):
    ...             self._outer = outer
    ... 
    ...         @property
    ...         def current_floor(self):
    ...             return self._outer._current_floor
    ... 
    ...         @property
    ...         def motor_direction(self):
    ...             return self._outer._motor_direction
    ... 
    ...         @motor_direction.setter
    ...         def motor_direction(self, direction):
    ...             self._outer._motor_direction = direction

### Simulation

The simulation runs in steps. Every time step consists of either a change of floor, or a pause at a floor. Either way, the business logic delegate gets notified. Along the way, we print out the movements of the elevator so that we can keep track of it. We also define a few helper methods.

    >>> class Elevator(Elevator):
    ...     def step(self):
    ...        delta = 0
    ...        if self._motor_direction == UP: delta = 1
    ...        elif self._motor_direction == DOWN: delta = -1
    ... 
    ...        if delta:
    ...            self._current_floor = self._current_floor + delta
    ...            print "%s..." % self._current_floor,
    ...            self._logic_delegate.on_floor_changed()
    ...        else:
    ...            self._logic_delegate.on_ready()
    ... 
    ...        assert self._current_floor >= 1
    ...        assert self._current_floor <= FLOOR_COUNT
    ...     
    ...     def run_until_stopped(self):
    ...         self.step()
    ...         while self._motor_direction is not None: self.step()
    ...     
    ...     def run_until_floor(self, floor):
    ...         for i in range(100):
    ...             self.step()
    ...             if self._current_floor == floor: break
    ...         else: assert False

That's it for the framework.

Business Logic
--------------

As for the business logic, an example implementation is provided in the `elevator.py` file in this project.

    >>> from elevator_solution import ElevatorLogic

As provided, it doesn't pass the tests in this document. Your challenge is to fix it so that it does. To run the tests, run this in your shell:

    python -m doctest -v README.md

With the correct business logic, here's how the elevator should behave:

### Basic usage

Make an elevator. It starts at the first floor.

    >>> e = Elevator(ElevatorLogic())
    1...

Somebody on the fifth floor wants to go down.

    >>> e.call(5, DOWN)

Keep in mind that the simulation won't actually advance until we call `step` or one of the `run_*` methods.

    >>> e.run_until_stopped()
    2... 3... 4... 5...

The elevator went up to the fifth floor. A passenger boards and wants to go to the first floor.

    >>> e.select_floor(1)

Also, somebody on the third floor wants to go down.

    >>> e.call(3, DOWN)

Even though the first floor was selected first, the elevator services the call at the third floor...

    >>> e.run_until_stopped()
    4... 3...

...before going to the first floor.

    >>> e.run_until_stopped()
    2... 1...

### Directionality

Elevators want to keep going in the same direction. An elevator will serve as many requests in one direction as it can before going the other way. For example, if an elevator is going up, it won't stop to pick up passengers who want to go down until it's done with everything that requires it to go up.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(2, DOWN)
    >>> e.select_floor(5)

Even though the elevator was called at the second floor first, it will service the fifth floor...

    >>> e.run_until_stopped()
    2... 3... 4... 5...

...before coming back down for the second floor.

    >>> e.run_until_stopped()
    4... 3... 2...

In fact, if a passenger tries to select a floor that contradicts the current direction of the elevator, that selection is ignored entirely. You've probably seen this before. You call the elevator to go down. The elevator shows up, and you board, not realizing that it's still going up. You select a lower floor. The elevator ignores you.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.select_floor(3)
    >>> e.select_floor(5)
    >>> e.run_until_stopped()
    2... 3...
    >>> e.select_floor(2)

At this point the elevator is at the third floor. It's not finished going up because it's wanted at the fifth floor. Therefore, selecting the second floor goes against the current direction, so that request is ignored.

    >>> e.run_until_stopped()
    4... 5...
    >>> e.run_until_stopped()  # nothing happens, because e.select_floor(2) was ignored

Now it's done going up, so you can select the second floor.

    >>> e.select_floor(2)
    >>> e.run_until_stopped()
    4... 3... 2...

### Changing direction

The process of switching directions is a bit tricky. Normally, if an elevator going up stops at a floor and there are no more requests at higher floors, the elevator is free to switch directions right away. However, if the elevator was called to that floor by a user indicating that she wants to go up, the elevator is bound to consider itself going up.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(2, DOWN)
    >>> e.call(4, UP)
    >>> e.run_until_stopped()
    2... 3... 4...
    >>> e.select_floor(5)
    >>> e.run_until_stopped()
    5...
    >>> e.run_until_stopped()
    4... 3... 2...

If nobody wants to go further up though, the elevator can turn around.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(2, DOWN)
    >>> e.call(4, UP)
    >>> e.run_until_stopped()
    2... 3... 4...
    >>> e.run_until_stopped()
    3... 2...

If the elevator is called in both directions at that floor, it must wait once for each direction.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.select_floor(5)
    >>> e.call(5, UP)
    >>> e.call(5, DOWN)
    >>> e.run_until_stopped()
    2... 3... 4... 5...

Here, the elevator considers itself to be going up, as it favors continuing in the direction it came from.

    >>> e.select_floor(4)  # ignored
    >>> e.run_until_stopped()

Since nothing caused the elevator to move further up, it now waits for requests that cause it to move down.

    >>> e.select_floor(6)  # ignored
    >>> e.run_until_stopped()

Since nothing caused the elevator to move down, the elevator now considers itself idle. It can move in either direction.

    >>> e.select_floor(6)
    >>> e.run_until_stopped()
    6...

### En passant

Keep in mind that a user could call the elevator or select a floor at any time. The elevator need not be stopped. If the elevator is called or a floor is selected before it has reached the floor in question, then the request should be serviced.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.select_floor(6)
    >>> e.run_until_floor(2)  # elevator is not stopped
    2...
    >>> e.select_floor(3)
    >>> e.run_until_stopped()  # stops for above
    3...
    >>> e.run_until_floor(4)
    4...
    >>> e.call(5, UP)
    >>> e.run_until_stopped()  # stops for above
    5...

On the other hand, if the elevator is already at, or has passed the floor in question, then the request should be treated like a request in the wrong direction. That is to say, a call is serviced later, and a floor selection is ignored.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.select_floor(5)
    >>> e.run_until_floor(2)
    2...
    >>> e.call(2, UP)  # missed the boat, come back later
    >>> e.step()  # doesn't stop
    3...
    >>> e.select_floor(3)  # missed the boat, ignored
    >>> e.step()  # doesn't stop
    4...
    >>> e.run_until_stopped()  # service e.select_floor(5)
    5...
    >>> e.run_until_stopped()  # service e.call(2, UP)
    4... 3... 2...

Fuzz testing
------------

No amount of legal moves should compel the elevator to enter an illegal state, which would be caught by asserts.

    >>> import random
    >>> e = Elevator(ElevatorLogic())
    1...
    >>> try: print '-',  # doctest:+ELLIPSIS
    ... finally:
    ...     for i in range(100000):  
    ...         r = random.randrange(6)
    ...         if r == 0: e.call(
    ...             random.randrange(FLOOR_COUNT) + 1,
    ...             random.choice((UP, DOWN)))
    ...         elif r == 1: e.select_floor(random.randrange(FLOOR_COUNT) + 1)
    ...         else: e.step()
    - ...


More Examples
-------------

The rest of these examples may be useful for catching bugs. They are meant to be run via doctest, so they may not be very interesting to read through.

An elevator is called but nobody boards. It goes idle.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(5, UP)
    >>> e.run_until_stopped()
    2... 3... 4... 5...
    >>> e.run_until_stopped()
    >>> e.run_until_stopped()

The elevator is called at two different floors.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(3, UP)
    >>> e.call(5, UP)
    >>> e.run_until_stopped()
    2... 3...
    >>> e.run_until_stopped()
    4... 5...

Like above, but called in reverse order.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(5, UP)
    >>> e.call(3, UP)
    >>> e.run_until_stopped()
    2... 3...
    >>> e.run_until_stopped()
    4... 5...

The elevator is called at two different floors, but going the other direction.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(3, DOWN)
    >>> e.call(5, DOWN)
    >>> e.run_until_stopped()
    2... 3... 4... 5...
    >>> e.run_until_stopped()
    4... 3...

The elevator is called at two different floors, going in opposite directions.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(3, UP)
    >>> e.call(5, DOWN)
    >>> e.run_until_stopped()
    2... 3...
    >>> e.run_until_stopped()
    4... 5...

Like above, but with directions reversed.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(3, DOWN)
    >>> e.call(5, UP)
    >>> e.run_until_stopped()
    2... 3... 4... 5...
    >>> e.run_until_stopped()
    4... 3...

The elevator is called at two different floors, one above the current floor and one below. It first goes to the floor where it was called first.

    >>> e = Elevator(ElevatorLogic(), 3)
    3...
    >>> e.call(2, UP)
    >>> e.call(4, UP)
    >>> e.run_until_stopped()
    2...
    >>> e.run_until_stopped()
    3... 4...

Like above, but called in reverse order.

    >>> e = Elevator(ElevatorLogic(), 3)
    3...
    >>> e.call(4, UP)
    >>> e.call(2, UP)
    >>> e.run_until_stopped()
    4...
    >>> e.run_until_stopped()
    3... 2...

The elevator is called while it's already moving.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(5, UP)
    >>> e.run_until_floor(2)
    2...
    >>> e.call(3, UP)
    >>> e.run_until_stopped()
    3...
    >>> e.run_until_stopped()
    4... 5...

If the elevator is already at, or has passed the floor where it was called, it comes back later.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(5, UP)
    >>> e.run_until_floor(3)
    2... 3...
    >>> e.call(3, UP)
    >>> e.run_until_stopped()
    4... 5...
    >>> e.run_until_stopped()
    4... 3...

Two floors are selected.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.select_floor(3)
    >>> e.select_floor(5)
    >>> e.run_until_stopped()
    2... 3...
    >>> e.run_until_stopped()
    4... 5...

Like above, but selected in reverse order.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.select_floor(5)
    >>> e.select_floor(3)
    >>> e.run_until_stopped()
    2... 3...
    >>> e.run_until_stopped()
    4... 5...

Two floors are selected, one above the current floor and one below. The first selection sets the direction, so the second one is completely ignored.

    >>> e = Elevator(ElevatorLogic(), 3)
    3...
    >>> e.select_floor(2)
    >>> e.select_floor(4)
    >>> e.run_until_stopped()
    2...
    >>> e.run_until_stopped()

Like above, but selected in reverse order.

    >>> e = Elevator(ElevatorLogic(), 3)
    3...
    >>> e.select_floor(4)
    >>> e.select_floor(2)
    >>> e.run_until_stopped()
    4...
    >>> e.run_until_stopped()

If the elevator is called to a floor going up, it should ignore a request to go down.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(5, UP)
    >>> e.run_until_stopped()
    2... 3... 4... 5...
    >>> e.select_floor(6)
    >>> e.select_floor(4)
    >>> e.run_until_stopped()
    6...
    >>> e.run_until_stopped()

Like above, but going in other direction.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(5, DOWN)
    >>> e.run_until_stopped()
    2... 3... 4... 5...
    >>> e.select_floor(6)
    >>> e.select_floor(4)
    >>> e.run_until_stopped()
    4...
    >>> e.run_until_stopped()

Elevator is called to a floor and a passenger also selects the same floor. The elevator should not go back to that floor twice.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(5, DOWN)
    >>> e.select_floor(5)
    >>> e.run_until_stopped()
    2... 3... 4... 5...
    >>> e.select_floor(4)
    >>> e.run_until_stopped()
    4...
    >>> e.run_until_stopped()

Similarly, if the elevator is called at a floor where it is stopped, it should not go back later.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(3, UP)
    >>> e.run_until_stopped()
    2... 3...
    >>> e.call(3, UP)
    >>> e.call(5, DOWN)
    >>> e.run_until_stopped()
    4... 5...
    >>> e.run_until_stopped()

Elevator is ready to change direction, new call causes it to keep going in same direction.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.call(2, DOWN)
    >>> e.call(4, UP)
    >>> e.run_until_stopped()
    2... 3... 4...
    >>> e.call(5, DOWN)  # It's not too late.
    >>> e.run_until_stopped()
    5...
    >>> e.run_until_stopped()
    4... 3... 2...

When changing directions, wait one step to clear current direction.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.select_floor(5)
    >>> e.call(5, UP)
    >>> e.call(5, DOWN)
    >>> e.run_until_stopped()
    2... 3... 4... 5...
    >>> e.select_floor(4)  # ignored
    >>> e.run_until_stopped()
    >>> e.select_floor(6)  # ignored
    >>> e.select_floor(4)
    >>> e.run_until_stopped()
    4...
    >>> e.run_until_stopped()

Like above, but going in other direction.

    >>> e = Elevator(ElevatorLogic(), 6)
    6...
    >>> e.select_floor(2)
    >>> e.call(2, UP)
    >>> e.call(2, DOWN)
    >>> e.run_until_stopped()
    5... 4... 3... 2...
    >>> e.select_floor(3)  # ignored
    >>> e.run_until_stopped()
    >>> e.select_floor(1)  # ignored
    >>> e.select_floor(3)
    >>> e.run_until_stopped()
    3...
    >>> e.run_until_stopped()

If other direction is not cleared, come back.

    >>> e = Elevator(ElevatorLogic())
    1...
    >>> e.select_floor(5)
    >>> e.call(5, UP)
    >>> e.call(5, DOWN)
    >>> e.run_until_stopped()
    2... 3... 4... 5...
    >>> e.select_floor(6)
    >>> e.run_until_stopped()
    6...
    >>> e.run_until_stopped()
    5...
    >>> e.select_floor(6)  # ignored
    >>> e.select_floor(4)
    >>> e.run_until_stopped()
    4...
    >>> e.run_until_stopped()
