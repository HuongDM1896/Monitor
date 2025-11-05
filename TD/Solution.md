# Remind class

## Example A: Left - Right Shout
### 1. Use Mutex (only 1 can shout each time)
**Define:**  
- Left: 1,2  
- Right: 1,2  
- Use 1 Mutex logic

```css
Timeline →
┌───────────────────────────────────────────────────────────┐
L1: |----LOCK----SHOUT----UNLOCK----------------------------|
L2: |-------------------WAIT(lock)---LOCK--SHOUT--UNLOCK----|
R1: |-------------------------------WAIT(lock)---LOCK-SHOUT-|
R2: |--------------------------------------WAIT(lock)-------|
└───────────────────────────────────────────────────────────┘
```
**Explain:**

- L1 first > keep Lock -> shout -> unlock (release Lock)
- In same time: L2, R1, R2 wait.
- After L1 unlock > other thread awake (signal) and keep Lock. 

-> Make sure only one person can shout each time, even they r in same group or not.

### 2. Use Monitor (multiple in same group can shout same time)

Use Monitor logic:  
mode = NONE -> no shout  
use notify including cond_left, cond_right -> condition to operate.

```css
┌──────────────────────────────────────────────────────────────┐
L1: |--ENTER_LEFT(set mode=LEFT)--SHOUT===================LEAVE|
L2: |-------ENTER_LEFT(shared LEFT)----SHOUT===================|
R1: |----------------WAIT(cond_right)--------------------------|
R2: |----------------WAIT(cond_right)--------------------------|
... (L1, L2 done)
L1: |========================LEAVE(mode=NONE, notify right)====|
R1: |------------------------------------WAKE→ENTER_RIGHT------|
R2: |------------------------------------ENTER_RIGHT-----------|
└──────────────────────────────────────────────────────────────┘
```

**Explain:**

- L1 enters first: See mode = NONE → change to LEFT -> Start shouting.
- L2 arrives: See mode = LEFT → allowed to enter immediately (same side).
--> Both shout at the same time.
- R1, R2 arrive: See mode = LEFT → blocked (wait) on cond_right.
- When L1 and L2 leave (no one on the left): nb_left = 0   
-→ mode = NONE, notify(cond_right) → wake up (signal) the right side.
-R1, R2 on the same right side are allowed to enter → shout in parallel.

**Conclusion:**  
- Both sides do not shout at the same time  
- Members on the same side are allowed to shout at the same time  
- wait()/notify() in the monitor helps to coordinate the right “policy”.

**Example B:  abt reader and writer:**

We have same resource:
- **Readers** only read -> can read in parallel, dont attack others.
- **Writers** only write -> need completely exclusive (no one can attack while writing).

```python
int nb_readers = 0 # num of readers r reading
bool writing = False # any writer is writing or not
Condition can_read, can_write # correspond queueing
```

So, we have:  
-    R1,2,3 : 3 readers  
-    W1,2 : 2 writers

Policy:  
Reader can read if:  
- no writer is writing (Writing == False)  
Writer can write if:  
- no reader is reading (nb_readers == 0)+ no other writer writing (writing == False)

-> Multiple readers can read in sametime, writer got blocked.

```css
Timeline 1: multiple readers are reading  →
┌──────────────────────────────────────────────────────────────┐
R1: |--start_read()--READ=========================end_read()----|
R2: |-----start_read()--READ=========================end_read()--|
R3: |-----------start_read()--READ==============================|
W1: |---------------------WAIT(can_write)-----------------------|
└──────────────────────────────────────────────────────────────┘
```

Explain:
- R1 starts → sees writing=False → starts reading. 
- R2, R3 come later → also see no writer → read together.  
- W1 comes → sees nb_readers>0 → must wait (wait(can_write)).  

→ Result: many readers read at the same time, writer is blocked.

```css
Timeline 2: Readers left and then writer can operate →
┌──────────────────────────────────────────────────────────────┐
R1: |================READ=end_read()=>nb_readers=2-------------|
R2: |================READ=end_read()=>nb_readers=1-------------|
R3: |================READ=end_read()=>nb_readers=0--notify()---|
W1: |--------------------------------WAKE→start_write()--WRITE=|
└──────────────────────────────────────────────────────────────┘
```

Explain:

- Each end_read() reduces 1 nb_readers  
- when nb_readers==0 -> stop read -> can write.
- wake up (signal) W1 -> writing by W1 (writing==True).


```css
Timeline 3: finish Writer and readers now can read again →
┌──────────────────────────────────────────────────────────────┐
W1: |====WRITE===end_write()→writing=False→notify(can_read)====|
R4: |------------------------------------WAKE→start_read()=====|
R5: |------------------------------------start_read()==========|
W2: |------------------------------------WAIT(can_write)-------|
└──────────────────────────────────────────────────────────────┘
```

Explain:  

- W1 done -> writing=False, can wake up can_read  
- All readers are waiting can join in same time. (for exp here is R4-5)  
- next writer (W2) have to wait until R finish.

*Pseudo code logic in Monitor:*

```python
Monitor RW:
    int nb_readers = 0
    bool writing = False
    Condition can_read, can_write

    start_read():
        while writing:
            can_read.wait()
        nb_readers += 1

    end_read():
        nb_readers -= 1
        if nb_readers == 0:
            can_write.notify()

    start_write():
        while writing or nb_readers > 0:
            can_write.wait()
        writing = True

    end_write():
        writing = False
        if not can_read.empty():
            can_read.notify_all()
        else:
            can_write.notify()
```

Conclusion: Reader & Writer do not use Mutex: 

- if only one Mutex -> Readers have to work in sequence -> no parallel  
- Monitor = multi readers same time, only Writers exclusive, clear policy.

# Correct the exercise

## Ex1.  

Producers: create data (messages) and put them into the buffer.

Consumers: take data out of the buffer and process it.

Buffer has a fixed size of N cells → must ensure:

Do not write when full

Do not read when empty

Data withdrawal is in FIFO order (circular).

**REMEMBER:**
```
For each monitor exercise, always follow the 4-step cycle:

1. Define the behavior (specification)

2. List the blocking / wakeup conditions (Blocking & Unblocking)

3. Deduce the state variable & condition

4. Write the code + relate each condition to each wait() / notify()
```

**VARIANT 1 – “Basic buffer of N cells”**

```python
Monitor Prod_Cons:
    put(data: elem_t)        # producer call
    get() -> elem_t          # consumer call
```
**Blocking and Unblocking Conditions**

| Role         | Blocking (when get block)                     | Unblocking (when get signal)               | Explain                                                                               |
| ------------ | ------------------------------------------ | --------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **Producer** | Buffer **full** (no empty space)         | A consumer has just **consumed** an element| Producer must wait until `empty == 0`, only when consumer takes some can continue production |
| **Consumer** | Buffer **empty** (no element to get) | Just had a producer **finish** an element| Consumer must wait when `empty == N`, only when producer adds it can it be taken.         |

**Variables and Conditions**

| var               | meaning                               |
| ------------------ | ------------------------------------- |
| `empty` (int)      | Number of empty cells in buffer (initial = N) |
| `Condition c_prod` | Queue for all producers         |
| `Condition c_cons` | Queue for all consumers         |


*Why 2 conditions?*

Producer and consumer have **different waiting behavior** (one side waits for “full”, one side waits for “empty”) -> separate queues.

Code:

```python
Monitor Prod_Cons:
    empty = N
    Condition c_prod, c_cons

    put(data: elem_t):
        while empty == 0:             # BLOCKING: buffer full
            c_prod.wait()
        empty -= 1                    # (add data to buffer in 'in')
        c_cons.notify()               # UNBLOCK consumer: just produced

    get() -> elem_t:
        while empty == N:             # BLOCKING: buffer empty
            c_cons.wait()
        empty += 1                    # (pick up data out of buffer in 'out')
        val = copied_data             # copy, no pointer
        c_prod.notify()               # UNBLOCK producer: just consumed
        return val
```

**Mapping B/U Conditions ↔ Code**

| Condition                           | Line               | For what                                              |
| ----------------------------------- | --------------------------------- | ---------------------------------------------------- |
| Producer Blocking = Full            | `while empty == 0: c_prod.wait()` | When no empty space in buffer - producer wait                      |
| Producer Unblocking = Just consumed | `c_prod.notify()` trong `get()`   | Consumer just picked up 1 element, buffer not full |
| Consumer Blocking = Empty           | `while empty == N: c_cons.wait()` | Buffer empty all, consumer wait                   |
| Consumer Unblocking = Just produced | `c_cons.notify()` trong `put()`   | Producer add 1 element, buffer no empty  |

-> *Each **Blocking/Unblocking** condition has a **corresponding line** of code.*

**VARIANT 2 – “Two message types, alternate deposits”**

Specification:

```python
Monitor Prod_Cons:
    put(data: elem_t, type: int)     # type = 0 hoặc 1
    get() -> elem_t
```

Same buffer size N, but 2 types message: 0 - 1 

Messages are produced alternately (0 – 1 – 0 – 1 – ...)

**Blocking and Unblocking Conditions**

| Role         | Blocking                                                                                                                               | Unblocking                                                                                                                                  | explain                                                                                             |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **Producer** | - Buffer full (`empty == 0`) <br> **or** last type (`last_type`) == my type (`type`) → cannot create 2 of the same type consecutively | - When a consumer **just consumed** an element (freeing up space) <br> **or** when a **producer of a different type** just finished producing (changing type, still has space) | Interleaving must be ensured: if a type 0 is just set, another type 0 producer must wait until a type 1 is set. |
| **Consumer** | - Buffer empty (`empty == N`)                                                                                                          | - When **a new message is produced**                                                                                                  | Same variant 1.                                                                                       |

Variables and Conditions:

| Biến                  | Ý nghĩa                                                                 |
| --------------------- | ----------------------------------------------------------------------- |
| `empty` (int)         | Num of empty (int = N)                                                |
| `last_type` (int)     | Type of last message |
| `Condition c_prod[2]` | 2 queues for producer — type 0, type 1                  |
| `Condition c_cons`    | only 1 queue for consumer (output)                                   |


**Code :**

```python
Monitor Prod_Cons:
    empty = N
    last_type = -1
    Condition c_prod[2], c_cons

    put(data: elem_t, type: int):
        while empty == 0 or last_type == type:   # BLOCK: full or same type
            c_prod[type].wait()
        empty -= 1
        # Put data to buffer
        last_type = type

        # Prefer to wake up (signal) consumer first if we have consumer
        if not c_cons.empty():
            c_cons.notify()                      # UNBLOCK consumer: just produced
        # if No consumer is waiting, let other producer with != type  work
        elif empty != 0:
            c_prod[1 - type].notify()            # UNBLOCK != type producer

    get() -> elem_t:
        while empty == N:                        # BLOCK: buffer empty
            c_cons.wait()
        empty += 1
        # data out of buffer is copy data
        val = copied_data

        # After consumer picked up, wake up producer != type
        c_prod[1 - last_type].notify()           # UNBLOCK producer != type
        return val
```
**NOTE:** Prefer consumer first (because it is easier to write, avoid buffer full), but can choose other ways — both are valid if the logic guarantees no deadlock.

**Liên hệ B/U Conditions ↔ Code**

| Điều kiện                                           | Dòng code tương ứng                            | Ý nghĩa                                         |
| --------------------------------------------------- | ---------------------------------------------- | ----------------------------------------------- |
| **Producer Blocking = Full**                        | `while empty == 0: ...`                        | Buffer đầy, phải chờ consumer                   |
| **Producer Blocking = Last type == mine**           | `while last_type == type: ...`                 | Không được đặt cùng loại liên tiếp              |
| **Producer Unblocking = Just consumed**             | `c_prod[1 - last_type].notify()` trong `get()` | Consumer vừa tiêu thụ xong, buffer có chỗ trống |
| **Producer Unblocking = Just produced (diff type)** | `c_prod[1 - type].notify()` trong `put()`      | Producer khác loại vừa chạy xong, có thể xen kẽ |
| **Consumer Blocking = Empty**                       | `while empty == N: c_cons.wait()`              | Buffer trống, phải chờ producer                 |
| **Consumer Unblocking = Just produced**             | `c_cons.notify()` trong `put()`                | Producer vừa thêm dữ liệu                       |


**Discuss about priority between ‘just produced’ in both Producer and Consumer**

That is:
After the producer has placed a message, we have two possibilities to wake up:

Consumer (to get it immediately)

Producer of a different type (to continue interleaving)



---

#  Variant 3 – Two message types (FIFO consumption by type)

## Specification

```python
Monitor Prod_Cons:
    put(data: elem_t, type: int)   # type ∈ {0, 1}
    get(type: int) -> elem_t
```

* Producers deposit messages labeled with `type`.
* Consumers specify the `type` of message they want to withdraw.
* Consumption still follows **FIFO**: only the **oldest** message in the buffer can be consumed.
* **Priority:** on **producers** (as stated by the professor).

---

## State and condition variables

| Variable      | Meaning                                                  |
| ------------- | -------------------------------------------------------- |
| `tab[0..N-1]` | Circular buffer storing messages and their types         |
| `in`, `out`   | Positions for next put / next get                        |
| `count`       | Number of occupied slots                                 |
| `N`           | Buffer capacity                                          |
| `c_prod`      | Condition variable for waiting producers                 |
| `c_cons[2]`   | Condition variables for consumers (one per message type) |

---

## Blocking and Unblocking conditions

| Process            | **Blocking condition**                                                              | **Unblocking condition**                                                                                                                                                          |
| ------------------ | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Producer**       | Buffer **full** (`count == N`)                                                      | A message was **consumed** (→ one empty cell available)                                                                                                                           |
| **Consumer(type)** | Buffer **empty** (`count == 0`) **or** the **oldest element type ≠ requested type** | (1) A message of their type becomes **oldest** after a `put` or `get`, **and**<br>(2) either the buffer was empty before, or no producer is currently waiting (producer priority) |

In summary:

* Producers block only when the buffer is full.
* Consumers block either because there’s nothing to consume or because the oldest message isn’t of their desired type.
* A consumer wakes up only when the **oldest message** matches their requested type.
* Producers have higher priority when both could be unblocked.

---

## Monitor pseudo-code

```python
Monitor Prod_Cons:
    tab[0..N-1]
    in = 0
    out = 0
    count = 0
    Condition c_prod, c_cons[2]

    put(value, type):
        while count == N:
            c_prod.wait()

        was_empty = (count == 0)
        tab[in] = (copy(value), type)
        in = (in + 1) % N
        count += 1

        # If buffer was empty before, a consumer might proceed
        if was_empty:
            c_cons[type].signal()

    get(type) -> value:
        while count == 0 or tab[out].type != type:
            c_cons[type].wait()

        value = copy(tab[out].value)
        out = (out + 1) % N
        count -= 1

        # Priority to producers
        if c_prod.has_waiters():
            c_prod.signal()
        elif count != 0:
            c_cons[tab[out].type].signal()

        return value
```

---

# Variant 4 – Producers produce twice in a row (bonus)

> Only steps 1–3 required.  
> Priority: **Consumers** in case of conflict.

---

## Specification

```python
Monitor Prod_Cons:
    put(data: elem_t, id: int)
    get() -> elem_t
```

* Each producer, once it starts producing, **must produce twice consecutively** before another producer can act.
* Consumers remove messages FIFO.
* When both sides can proceed, **consumers have priority**.

---

## State and condition variables

| Variable                  | Meaning                                                                                                                                   |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `tab[0..N-1]`             | Circular buffer for messages                                                                                                              |
| `in`, `out`, `count`, `N` | Standard circular buffer management                                                                                                       |
| `stride`                  | Indicates which producer must produce next:<br>• `-1`: free (any producer can produce)<br>• `id`: producer with that id must produce next |
| `c_prod`                  | Condition for waiting producers (full buffer or wrong turn)                                                                               |
| `c_cons`                  | Condition for waiting consumers (empty buffer)                                                                                            |

---

## Blocking and Unblocking conditions

| Process          | **Blocking condition**                                                                                           | **Unblocking condition**                                                                                                                                |
| ---------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Producer(id)** | (1) Buffer **full** (`count == N`) **OR**<br>(2) Another producer has the turn (`stride != -1 and stride != id`) | (1) A message **consumed** (frees space), or<br>(2) `stride` becomes **-1** (their turn is free), or<br>(3) Their **own turn** resumes (`stride == id`) |
| **Consumer**     | Buffer **empty** (`count == 0`)                                                                                  | A producer **produces** a message (buffer not empty)                                                                                                    |

---

## State updates (conceptual behavior)

| Event                           | State changes        | Notes                                                          |
| ------------------------------- | -------------------- | -------------------------------------------------------------- |
| **First `put` by producer `P`** | Sets `stride = P`    | Locks production turn to `P` for next put                      |
| **Second `put` by same `P`**    | Resets `stride = -1` | Frees turn for any producer                                    |
| **`get()`**                     | Removes one element  | May wake producers if space available                          |
| **Conflict resolution**         | Consumer priority    | If both consumer and producer can proceed, wake consumer first |

---

## Monitor logic summary (no full code required)

1. **State variables**: `tab`, `in`, `out`, `count`, `stride`, `c_prod`, `c_cons`
2. **Blocking / unblocking**: summarized in the table above
3. **Execution flow**:

   * Each producer must produce two consecutive elements.
   * After its second production, another producer may take over.
   * Consumers consume in FIFO order, with **priority** over producers.

---

### Optional diagram (mental model)

```
Producer A: put1 → put2 → yield turn
Producer B: (waits) → put1 → put2 → yield turn
Consumers: always allowed to read FIFO, if not empty
```

---


---

# Exercise 2 – Monitor `Prod_Cons`

---

## Variant 1 – Single message type (basic FIFO)

### **Specification**

```python
Monitor Prod_Cons:
    put(data: elem_t)
    get() -> elem_t
```

* **Producers** put messages into a bounded FIFO buffer.
* **Consumers** get messages from the buffer in FIFO order.
* **Priority:** given to **producers** (as specified by the professor).

---

### **State and condition variables**

| Variable      | Meaning                                  |
| ------------- | ---------------------------------------- |
| `tab[0..N-1]` | Circular buffer of elements              |
| `in`, `out`   | Indices for next `put` / next `get`      |
| `count`       | Number of occupied slots                 |
| `N`           | Buffer capacity                          |
| `c_prod`      | Condition variable for waiting producers |
| `c_cons`      | Condition variable for waiting consumers |

---

### **Blocking and Unblocking conditions**

| Process      | **Blocking condition**          | **Unblocking condition**                                |
| ------------ | ------------------------------- | ------------------------------------------------------- |
| **Producer** | Buffer **full** (`count == N`)  | A message was **consumed** (→ one slot becomes free)    |
| **Consumer** | Buffer **empty** (`count == 0`) | A message was **produced** (→ buffer becomes non-empty) |

**Producers have priority:**
If both can proceed, **signal producer first**.

---

### **Monitor pseudo-code**

```python
Monitor Prod_Cons:
    tab[0..N-1]
    in = 0
    out = 0
    count = 0
    Condition c_prod, c_cons

    put(value):
        while count == N:
            c_prod.wait()

        tab[in] = copy(value)
        in = (in + 1) % N
        count += 1

        # Signal a consumer if buffer was empty before
        if count == 1:
            c_cons.signal()

    get() -> value:
        while count == 0:
            c_cons.wait()

        value = copy(tab[out])
        out = (out + 1) % N
        count -= 1

        # Producer priority
        if c_prod.has_waiters():
            c_prod.signal()
        elif count > 0:
            c_cons.signal()

        return value
```

---

## Variant 2 – Two message types (independent FIFOs per type)

### **Specification**

```python
Monitor Prod_Cons:
    put(data: elem_t, type: int)   # type ∈ {0, 1}
    get(type: int) -> elem_t
```

* Each **type** of message has its **own FIFO queue**.
* Consumers specify which **type** they want.
* **Producers** only block when their own queue is full.
* **Priority:** on **producers** (same as Variant 1).

---

### **State and condition variables**

| Variable                | Meaning                                     |
| ----------------------- | ------------------------------------------- |
| `tab[type][0..N-1]`     | Two circular buffers (one per message type) |
| `in[type]`, `out[type]` | Indices for each buffer                     |
| `count[type]`           | Occupied slots for each type                |
| `N`                     | Capacity per buffer                         |
| `c_prod[2]`             | One condition variable per producer type    |
| `c_cons[2]`             | One condition variable per consumer type    |

---

### **Blocking and Unblocking conditions**

| Process            | **Blocking condition**                    | **Unblocking condition**            |
| ------------------ | ----------------------------------------- | ----------------------------------- |
| **Producer(type)** | Its buffer **full** (`count[type] == N`)  | A message of same type **consumed** |
| **Consumer(type)** | Its buffer **empty** (`count[type] == 0`) | A message of same type **produced** |

Independent queues mean **no cross-type blocking**.

---

### **Monitor pseudo-code**

```python
Monitor Prod_Cons:
    tab[2][0..N-1]
    in[2] = {0, 0}
    out[2] = {0, 0}
    count[2] = {0, 0}
    Condition c_prod[2], c_cons[2]

    put(value, type):
        while count[type] == N:
            c_prod[type].wait()

        tab[type][in[type]] = copy(value)
        in[type] = (in[type] + 1) % N
        count[type] += 1

        if count[type] == 1:
            c_cons[type].signal()

    get(type) -> value:
        while count[type] == 0:
            c_cons[type].wait()

        value = copy(tab[type][out[type]])
        out[type] = (out[type] + 1) % N
        count[type] -= 1

        # Producer priority
        if c_prod[type].has_waiters():
            c_prod[type].signal()
        elif count[type] > 0:
            c_cons[type].signal()

        return value
```

---

## Variant 3 – Two message types (FIFO consumption by type)

### **Specification**

```python
Monitor Prod_Cons:
    put(data: elem_t, type: int)   # type ∈ {0, 1}
    get(type: int) -> elem_t
```

* Producers deposit messages labeled with `type`.
* Consumers specify the `type` of message they want to withdraw.
* **Single shared FIFO**, messages stay in arrival order.
* **FIFO constraint:** only the **oldest** message in the buffer can be consumed.
* **Priority:** on **producers** (same rule).

---

### **State and condition variables**

| Variable      | Meaning                                                  |
| ------------- | -------------------------------------------------------- |
| `tab[0..N-1]` | Circular buffer storing messages and their types         |
| `in`, `out`   | Positions for next put / next get                        |
| `count`       | Number of occupied slots                                 |
| `N`           | Buffer capacity                                          |
| `c_prod`      | Condition variable for waiting producers                 |
| `c_cons[2]`   | Condition variables for consumers (one per message type) |

---

### **Blocking and Unblocking conditions**

| Process            | **Blocking condition**                                                              | **Unblocking condition**                                                                                                                                                          |
| ------------------ | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Producer**       | Buffer **full** (`count == N`)                                                      | A message was **consumed** (→ one empty cell available)                                                                                                                           |
| **Consumer(type)** | Buffer **empty** (`count == 0`) **or** the **oldest element type ≠ requested type** | (1) A message of their type becomes **oldest** after a `put` or `get`, **and**<br>(2) either the buffer was empty before, or no producer is currently waiting (producer priority) |

---

### **Monitor pseudo-code**

```python
Monitor Prod_Cons:
    tab[0..N-1]
    in = 0
    out = 0
    count = 0
    Condition c_prod, c_cons[2]

    put(value, type):
        while count == N:
            c_prod.wait()

        was_empty = (count == 0)
        tab[in] = (copy(value), type)
        in = (in + 1) % N
        count += 1

        # If buffer was empty before, a consumer might proceed
        if was_empty:
            c_cons[type].signal()

    get(type) -> value:
        while count == 0 or tab[out].type != type:
            c_cons[type].wait()

        value = copy(tab[out].value)
        out = (out + 1) % N
        count -= 1

        # Producer priority
        if c_prod.has_waiters():
            c_prod.signal()
        elif count != 0:
            c_cons[tab[out].type].signal()

        return value
```

---
