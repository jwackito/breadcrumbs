# Pandas - Select Rows Matching Several Simple Patterns

#### TL;DR
- Operations over DataFrames are, usually, more expensive than operations over NumPy arrays. Try always to use the second, specially when creating criteria to index rows in  DataFrames.
- Some operations, like creating a datetime using Pandas, may seem harmless but they take a lot of time to execute. If the code is inside a loop, the impact on the total time can be huge. When coding, think if you can write the relatively expensive code outside the loop.
- Use `line_profiler` or even `%timeit` (or `%%timeit` for multi-line) to test the runtime of your code.
```python
In [1]: %load_ext line_profiler
In [2]: def f1():
   ...:    ... your function here...
In [3]: def f2():
   ...:    ... your other funciton here ...
In [4]: %lprun -f f1 -f f2 [(f1(), f2()) for i in range(1000)]
# run the functions in a closed loop 1000 times.
```

#### In depth
Same principle as in [Select Rows Matching a Simple Pattern](efficient_selection_simple_pattern.md) applies when more than one criteria is used over one or several columns. If the DataFrame is a Zeek Log, I may be interested to get all the rows going from a particular client to a particular server, that is, the `df[df["id.orig_h"] == "some IP") & (df["id.resp_h"] == "some other IP")]`. If your dataset contains particle physics information, you may be interested in collisions with energies between two values, let's say the `collisions[(collisions.energy >= 10) & (collisions.energy <= 100)]`. If your dataset contains clients, you may be interested in getting the information of all the clients whose names match with either Gabriela or Joaquin that have spent more than 10000 in your store. That is, the `clients[((clients.name == "Gabriela") | (clients.name == "Joaquin")) & (clients.total_spent >= 10000)]`.

The examples of Pandas CookBook documentation shows solutions like the one in the figure below. Here I used several data types to show the impact this could have on the runtime of the solutions.

```python
In [1]: df = pd.DataFrame(
   ...:     {"AAA": ['a', 'b', 'c', 'd'],
   ...:      "BBB": [10, 20, 30, 40],
   ...:      "CCC": pd.date_range(start='2022-01-01', end='2022-01-04')}
   ...: )
   ...: Crit1 = df.AAA == 'b'
   ...: Crit2 = df.BBB >= 20
   ...: Crit3 = df.CCC <= pd.to_datetime('2022-01-03')
   ...: AllCrit = Crit1 | Crit2 & Crit3
   ...: df[AllCrit]
Out[1]:  
 AAA  BBB        CCC
1   b   20 2022-01-02
2   c   30 2022-01-03
```
In order to compare the execution times of these to methods, let's build two functions, f1 and f2, and measure their execution time line by line using the **inline_profiler** capabilities. If you are not familiar with this extension, it is installable by `pip install line-profiler` and allows us to identify which lines of each function are taking more time to execute.
```python
In [2]: %load_ext line_profiler

In [3]: def f1():
   ...:     Crit1 = df.AAA == 'b'
   ...:     Crit2 = df.BBB >= 20.0
   ...:     d = pd.to_datetime('2022-01-03')
   ...:     Crit3 = df.CCC <= d
   ...:     AllCrit = Crit1 | Crit2 & Crit3
   ...:     df[AllCrit]
   ...:  

In [4]: def f2():
   ...:     Crit1 = df.AAA.values == 'b'
   ...:     Crit2 = df.BBB.values >= 20.0
   ...:     d = pd.to_datetime('2022-01-03')
   ...:     Crit3 = df.CCC.values <= d
   ...:     AllCrit = Crit1 | Crit2 & Crit3
   ...:     df[AllCrit]
   ...:
In [5]: %lprun -f f1 -f f2 [(f1(), f2()) for i in range(1000)]
```
In the line `In [2]` the line_profiler extension is loaded. In lines `In [3]` and `In [4]` the functions `f1()` and `f2()` are defined. Both functions do the same but while function `f1()` uses DataFrames as indexes, the function `f2()` uses NumPy arrays. Finally in the line `In [5]`, the line profiler magic is called. The `-f` argument indicates which are the functions we want to profile. In this case, we want to profile both, `f1` and `f2`. Then we are invoking both functions inside an explicit list comprehension, that will execute both functions 1000 times each. 

The output of the magic command includes very detailed information of the run of each function.  First the line `Timer unit: 1e-06 s` indicates the resolution of the rest of the numbers, in this case, microseconds (μs). The rest of the information is per function. The `f1()` function took 1.32 seconds of the total run time, while the `f2()` took 0.55 seconds. The most important differences are between the lines 2, 3, and 5 of each function, where each criteria is calculated, and between line 6 of each function, where the criteria are consolidated together.  I did separate the creation of the datetime on purpose to show how much time it takes in the total run time, and to separate it from the comparison in both cases. Clearly, an improvement in both cases can be to declare the d variable outside the function definition, saving around 175 ms in the total time of each function. 
```
Timer unit: 1e-06 s

Total time: 1.32061 s
File: <ipython-input-215-f15297257859>
Function: f1 at line 1

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
    1                                           def f1():
    2      1000     187151.0    187.2     14.2      Crit1 = df.AAA == 'b'
    3      1000     164126.0    164.1     12.4      Crit2 = df.BBB >= 20.0
    4      1000     171967.0    172.0     13.0      d = pd.to_datetime('2022-01-03')
    5      1000     198570.0    198.6     15.0      Crit3 = df.CCC <= d
    6      1000     259318.0    259.3     19.6      AllCrit = Crit1 | Crit2 & Crit3
    7      1000     339473.0    339.5     25.7      df[AllCrit]

Total time: 0.554853 s
File: <ipython-input-216-031ea8e9fba2>
Function: f2 at line 1

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
    1                                           def f2():
    2      1000      31025.0     31.0      5.6      Crit1 = df.AAA.values == 'b'
    3      1000      21543.0     21.5      3.9      Crit2 = df.BBB.values >= 20.0
    4      1000     175008.0    175.0     31.5      d = pd.to_datetime('2022-01-03')
    5      1000      29356.0     29.4      5.3      Crit3 = df.CCC.values <= d
    6      1000       3823.0      3.8      0.7      AllCrit = Crit1 | Crit2 & Crit3
    7      1000     294098.0    294.1     53.0      df[AllCrit]
```
The most interesting improvement occurs when the `AllCrit` variable is calculated, in line 6 of each function. In `f1`, operating over DataFrames, each execution takes more than 259 μs, while in `f2`, operating over NumPy arrays, it takes less than 4 μs. That is more than an order of magnitude improvement, and of course, the more code like this is executed, the bigger the impact in the final run time. 

Even when it is possible to do a more in depth analysis, for example to determine if the execution time grows logarithmically, linearly or exponentially with respect to the size of the input, I think it is clear that working with NumPy arrays whenever is possible, will reduce the execution time, especially in large DataFrames.
