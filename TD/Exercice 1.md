# Exercice 1

## Variant 1

### Specification

```python
Monitor Prod_Cons:
    put(data: elem_t)
    get() -> elem_t
```

### Blocking and unblocking conditions

|            | Producer      | Consumer      |
| ---------- | ------------- | ------------- |
| Blocking   | Full          | Empty         |
| Unblocking | Just consumed | Just produced |

### Variables and conditions

- 1 variable **empty** (or **full**) integer, the number of empty cells
- 2 conditions **c_prod** and **c_cons** as both behaviors are different

### Code

Ask the students to link each B/U condition to part of the code to check if none is forgotten (valid for all exercises)

```python
Monitor Prod_Cons:
    empty = N
    Condition c_prod, c_cons

    put(data:elem_t):
        while empty == 0:
            c_prod.wait()
        empty -= 1
        // Actually put the data, update in, out, ...
        c_cons.notify()

    get():
        while empty == N:
            c_cons.wait()
        empty += 1
        // actually get the data in a variable called val
        // RQ val must be a copied version, not a pointer !
        c_prod.notify()
        return val // explain that val is not shared so it can be after the notify
```

## Variant 2

### Specification

```python
Monitor Prod_Cons:
    put(data: elem_t, type:int)
    get() -> elem_t
```
with type 0 or 1

### Blocking and unblocking conditions

RQ: Discuss about the priority between the 'just produced' in both Producer and Consumer. Put the priority for consumer it makes writing the code easier but both are possible.

|            | Producer                                                                                    | Consumer      |
| ---------- | -------------                                                                               | ------------- |
| Blocking   | Full OR last type == mine                                                                   | Empty         |
| Unblocking | Just consumed OR just produced with different type and enough place and no consumer waiting | Just produced |

### Variables and conditions

- 2 variables
    - **empty** (or **full**) integer, the number of empty cells
	- **last_type** the type of the last one, or -1 if none yet
- 3 conditions **c_prod[2]** and **c_cons** with c_prod[0] for producers of type 0...

### Code 

```python
Monitor Prod_Cons:
    empty = N
    last_type = -1
    Condition c_prod[2], c_cons

    put(data:elem_t, type:int):
	    while empty == 0 or last_type == type:
		    c_prod[type].wait()
        empty -= 1
        // Actually put the data, update in, out, ...
        last_type = type
        if not c_cons.empty():
            c_cons.notify()
        else if empty != 0:
            c_prod[1-type].notify()

    get():
        while empty == N:
            c_cons.wait()
        empty += 1
        // actually get the data in a variable called val = copied_data
        // RQ val must be a copied version, not a pointer !
        c_prod[1-last_type].notify()
        return val
```

## Variant 3

### Specification

```python
Monitor Prod_Cons:
    put(data: elem_t, type:int)
    get(type:int) -> elem_t
```
with type 0 or 1

### Blocking and unblocking conditions

Priority on producer. And a producer unblock a consumer only if the system was empty before (otherwise the consumer is still blocked on the oldest element)

|            | Producer      | Consumer                                                                                                      |
| ---------- | ------------- | -------------                                                                                                 |
| Blocking   | Full          | Empty or oldest not my type                                                                                   |
| Unblocking | Just consumed | ( just produced or ( just consumed and no producer is waiting ) ) and oldest item is my type |

### Variables and conditions

- 2 variables
    - **empty** (or **full**) integer, the number of empty cells
	- **types[N]** here only one value is not sufficient, needs to store all the types
- 3 conditions **c_prod** and **c_cons[2]** 

Remind the 'data type' variables
- tab (name of the table), in (position for puting the next element), out (position of the oldest element), N (total number of elements)

### Code

```python
Monitor Prod_Cons:
    empty = N
    types[N] = [-1, ..]
    Condition c_prod, c_cons[2]

    put(data:elem_t, type:int):
        while empty == 0:
            c_prod.wait()
        empty -= 1
        types[in] = type;
        // Actually put the data, update in, out, ...
        c_cons[type[out]].notify()

    get(type:int):
        while empty == N or type[out] != type:
            c_cons[type].wait()
        empty += 1
        // actually get the data in a variable called val
        // RQ val must be a copied version, not a pointer !
        // out is updated in this part
        if not c_prod.empty():
            c_prod.notify()
        else if empty != N:
            c_cons[type[out]].notify()
        return val
```

## Variant 4

### Specification

```python
Monitor Prod_Cons:
    put(data: elem_t, id:int)
    get() -> elem_t
```

### Blocking and unblocking conditions

|            | Producer                                                                             | Consumer      |
| ---------- | -------------                                                                        | ------------- |
| Blocking   | Full OR a single element produced by another producer                                | Empty         |
| Unblocking | Just consumed OR a producer just produced its second element and no consumer waiting | Just produced |

### Variables and conditions

- 2 variables
     - **empty** (or **full**) integer, the number of empty cells
     - **stride** integer. -1 or id of the producer that should produce
- 2 conditions **c_prod** and **c_cons** as both behaviors are different


# Exercise 2 - Reader Writer

The main idea is to understand that to achieve parallelism, the Reader and Writer cannot be inside the monitor.

## Specification

```python
Monitor Reader_Writer:
    startRead()
    endRead()
    startWrite()
    endWrite()
```

### Blocking and unblocking conditions

|            | Reader                                                                            | Writer                                            |
| ---------- | --------------------------------------------------------------------------------- | ------------------------------------------------- |
| Blocking   | A writer is writing                                                               | Reader is reading or writer is writing            |
| Unblocking | Writer just finished writing and no writer waiting OR Reader just started reading | Writer just finished or last reader just finished |

### Variables and conditions

2 variables
 - writing : boolean to know is a writer is writing
 - nb_reader : number of readers actually reading
and 2 conditions
 - c_reader, c_writer

### Code

```python
Monitor Reader_Writer:
    writing = False
    nb_reader = 0
    Condition c_reader, c_writer
    startRead():
        while writing:
            c_reader.wait()
        nb_reader++
        c_reader.notify()
    
    endRead():
        nb_reader--
        if nb_reader == 0:
            c_writer.notify()

    startWrite():
        while writing or nb_reader != 0:
            c_writer.wait()
        writing = True

    endWrite():
        writing = False
        if !c_writer.empty():
            c_writer.notify()
        else:
            c_reader.notify()
```

## Variant 2

Specification and variables are the same as previous version

### Blocking and unblocking conditions

|            | Reader                                                                            | Writer                                            |
| ---------- | --------------------------------------------------------------------------------- | ------------------------------------------------- |
| Blocking   | A writer is writing **OR a writer is waiting**                                    | Reader is reading or writer is writing            |
| Unblocking | Writer just finished writing and no writer waiting OR Reader just started reading | Writer just finished or last reader just finished |

### Code

The only difference is on the blocking condition of a reader, so ask the students to focus on where should be the only difference

```python
    startRead():
        while writing or !c_writer.empty():
            c_reader.wait()
        nb_reader++
        c_reader.notify()
```    

## Variant 3

Start with making the students find example of starvation. On variant 1, there is possibility of starvation of Readers, or of Writer. In variant 2, there is a possibility of starvation of Reader.

### Blocking and unblocking conditions

The problem of this proposal is that when a reader and a writer will start, there is an overlap that will block all processes

|            | Reader                                                      | Writer                                                                      |
| ---------- |----------------------------------------------------------- | --------------------------------------------------------------------------- |
| Blocking   | A writer is writing **OR a writer is waiting**              | Reader is reading or writer is writing                                      |
| Unblocking | Writer just finished writing OR Reader just started reading | Writer just finished **and no reader waiting** OR last reader just finished | 


So we need to know we are currently doing the cascade

|            | Reader                                                            | Writer                                                                      |
| ---------- |----------------------------------------------------------------- | --------------------------------------------------------------------------- |
| Blocking   | A writer is writing **OR a writer is waiting and not in cascade** | Reader is reading or writer is writing                                      |
| Unblocking | Writer just finished writing OR Reader just started reading       | Writer just finished **and no reader waiting** OR last reader just finished |

### Variables and conditions

we need to add a 
  - cascade: boolean

### Code

```python


    cascade = False

    startRead():
        if writing or (!c_writer.empty() and not cascade):
            c_reader.wait()
        nb_reader+=1
        if !c_reader.empty():
            cascade = True
            c_reader.notify()
        else:
            cascade = False
    
    endWrite():
        writing = False
        if !c_reader.empty():
            cascade = True
            c_reader.notify()
        else:            
            c_writer.notify()

```    


# Exercise 3 : Factory

The specification is provived

### Blocking and unblocking conditions

In the table : *my command not ready* assumes that a command has already been done. Otherwise write it in the table.

|            | Client                                                                       | Employee                      |
| ---------- | ---------------------------------------------------------------------------- | ----------------------------- |
| Blocking   | All counters are full OR my command not ready                                | No cmd of my step available   |
| Unblocking | I'm outside and a client just left OR I'm inside and my command just arrived | a cmd of my step just arrived | 

### Variables and conditions

We need to know when the counters are full, and at which step is each order
  - free : array of NB_COUNTERS boolean
  - steps : array of NB_COUNTERS integer

Both can be merged with a -1 for empty for example

and the counters themselves
  - counters = init_counters(NB_COUNTERS)

We also need
- one condition `c_outside` for the clients waiting for a counter to be available
- one condition per counter `c_counter[NB_COUNTERS]`, for the client waiting on this counter
- one condition per step `c_employee[NB_STEPS]`, for the employees waiting for a task of this step

### Code

```python
Monitor OrderManagement:
    # save current status of each counter
    steps = [-1] * NB_COUNTERS # -1 for free counter
    counters = init_counters(NB_COUNTERS) # init list of counter, each counter = 1 available order
    Conditions c_outside, c_counter[NB_COUNTERS], c_employee[NB_STEPS] # conditions variables

    # Customer side
    start_command(order):
    # customer can start order when the counter is free
        while not (-1 in steps): # all counters are busy, not -1 -> need wait
            c_outside.wait()
        counter = free.index(-1) # find the free counter (-1 is free)
        steps[counter] = 0 # set 1st step as 0, start order
        counters[counter] = order
        c_employee[0].notify() # let employee know that have order, can work
        return counter # return id of counter to wait until order finished

    end_command(counter):
    # remember the id of counter (see return above), wait there until the order finish
        while step[counter] != NB_STEPS: # step not 4 (4 is last step)
            c_counter[counter].wait() # wait until the counter done all the step
        steps[counter] = -1 # free the step
        c_outside.notify() # let other customer outside can come

    # Employee side   
    start_step(step):
    # the employee who does the "step" only works when the counter give the request of this step
        while not (step in steps): # if no request this step
            c_employee[step].wait() # wait
        return steps.index(step)
    
    end_step(counter):
    # employee finish their work in this step, move to next step
        steps[counter] += 1 # counter give the request of next step -> +1
        new_step = steps[counter] 
        if new_step == NB_STEPS: # if the request = last step
            c_counter[counter].notify() # counter tell the custom that the order done
        else: # if request not last step
            c_employee[new_step].notify() # let employee do the next step. 
```
