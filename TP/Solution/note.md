

# Exercise 1 – Producer / Consumer

## Goal

The goal of this exercise is to understand:
- How **unsynchronized access** to shared data leads to inconsistencies.
- How to progressively introduce **shared memory**, **locks**, and **conditions** to synchronize producers and consumers.

---

## Part 1 – Version without synchronization

### Step 1. Run the base code

Retrieve the file `m1_prodCons_base.py` from Moodle and run it with different parameters:

```bash
python3 prod_cons_base.py 3 3 1
python3 prod_cons_base.py 2 2 2
python3 prod_cons_base.py 4 4 4
````

**Observe:**

* The buffer content (`self.storage_val`) sometimes changes unpredictably.
* Multiple producers and consumers may access the same buffer cell at once.

### Step 2. Add runtime parameters

Modify the code so that:

```python
python3 prod_cons_base.py <nb_prod> <nb_cons> <nb_cases> <nb_times_prod> <nb_times_cons>
```

For example:

```bash
python3 prod_cons_base.py 2 2 4 5 5
```

This means:

* `nb_prod`: number of producer processes
* `nb_cons`: number of consumer processes
* `nb_cases`: number of slots in the buffer
* `nb_times_prod`: how many items each producer will produce
* `nb_times_cons`: how many items each consumer will consume

### Step 3. Use shared variables

Replace all local lists (`self.storage_val`, etc.) with shared objects from the `multiprocessing` module:

* Use `Array` for lists of integers.
* Use `Value` for integer pointers like `ptr_prod`, `ptr_cons`, and counters.

```python
from multiprocessing import Array, Value
```

**Important:** At this stage, do *not* use `Lock` or `Condition` yet.
You should see **corrupted buffer states** because multiple processes modify the shared memory simultaneously.

---

## Part 2 – Synchronization (Lock + Condition)

### Step 4. Add synchronization primitives

In the `Buffer` class constructor:

```python
self.lock = Lock()
self.cond_prod = Condition(self.lock)
self.cond_cons = Condition(self.lock)
```

This will allow you to safely control access to shared data.

### Step 5. Protect the critical sections

Use the `with self.lock:` context to ensure that only one process at a time can modify the buffer:

```python
def produce(...):
    with self.lock:
        # critical section
        ...
```

Same for `consume()`.

### Step 6. Manage full and empty buffer conditions

Introduce a shared counter `self.empty = Value('i', nb_cases)` to track available slots.

* In `produce()`:

  ```python
  while self.empty.value == 0:
      self.cond_prod.wait()     # buffer full → wait
  ```

  After producing:

  ```python
  self.empty.value -= 1
  self.cond_cons.notify()      # wake up a waiting consumer
  ```

* In `consume()`:

  ```python
  while self.empty.value == self.nb_cases:
      self.cond_cons.wait()     # buffer empty → wait
  ```

  After consuming:

  ```python
  self.empty.value += 1
  self.cond_prod.notify()      # wake up a waiting producer
  ```

### Step 7. Test locking behavior

Run with few cases to force waiting:

```bash
python3 prod_cons_cond.py 2 2 1 3 3
```

producers **blocked** when the buffer is full and consumers **blocked** when the buffer is empty.

---

## What to Observe

| Situation    | Condition                      | Who waits? | Who is notified? |
| ------------ | ------------------------------ | ---------- | ---------------- |
| Buffer full  | `self.empty.value == 0`        | Producer   | Consumer         |
| Buffer empty | `self.empty.value == nb_cases` | Consumer   | Producer         |

---


```python
class Buffer:
    def __init__(self, nb_cases):
        self.nb_cases = nb_cases
        self.lock = Lock()
        self.cond_prod = Condition(self.lock)
        self.cond_cons = Condition(self.lock)
        self.storage_val = Array('i', [-1] * nb_cases, lock=False)
        self.ptr_prod = Value('i', 0, lock=False)
        self.ptr_cons = Value('i', 0, lock=False)
        self.empty = Value('i', nb_cases, lock=False)

    def produce(self, val, typ, pid):
        with self.lock:
            while self.empty.value == 0:
                # HINT: buffer full → wait for consumer
                self.cond_prod.wait()
            # HINT: insert element and update pointers
            # ...
            self.cond_cons.notify()

    def consume(self, cid):
        with self.lock:
            while self.empty.value == self.nb_cases:
                # HINT: buffer empty → wait for producer
                self.cond_cons.wait()
            # HINT: remove element and update pointers
            # ...
            self.cond_prod.notify()
````
