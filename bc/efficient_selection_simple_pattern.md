# Pandas - Select Rows Matching a Simple Pattern

#### TL;DR
When filtering a DataFrame, use numpy arrays to calculate the criteria. Prefer always
```python
df[df['mycolumn'].values > 5]
```
instead of
```python
df[df['mycolumn'] > 5]
```
Impact can be an order of magnitude less time.
#### In depth
Sometimes I need to select the rows given that the value of a column matches a certain pattern. For example, if my DataFrame contains IP addresses of source and destination of data packets being transferred, I may be interested in all the packets sent by a certain host. If your dataset contains information about the collisions of particles in a particle accelerator,  you may be interested in getting all the particles of a certain type or above or below a certain mass. 

The usual way to do a selection is to create a mask indexer that match the criteria, that is, an array of Bool the same length of the DataFrame, that contains True in the position of the rows we want in the result and False in the positions of the rows we want filtered out of the results. It is possible to use the indexer directly inside brackets, similar to the standard array notation in python, or using the .loc[] property of the DataFrame.
```python
In [1]: df = pd.DataFrame(
   ...:     {"AAA": [4, 5, 6, 7],
   ...:      "BBB": [10, 20, 30, 40],
   ...:      "CCC": [100, 50, -30, -50]}
   ...: )
   ...: df
Out[1]:  
   AAA  BBB  CCC
0    4   10  100
1    5   20   50
2    6   30  -30
3    7   40  -50


In [2]: df[df.AAA > 5]  # or df.loc[df.AAA > 5]
Out[2]:  
   AAA  BBB  CCC
2    6   30  -30
3    7   40  -50
```
The .loc[] property allows several options to select rows, and some of them are really fancy, like using a python callable object (a function)  that returns an index. Some of the methods are more efficient than others, and in the example above we are using a DataFrame as the index. But what really makes a difference here is how the criteria is calculated.
```python
In [3]: df.AAA > 5
Out[3]:  
0    False
1    False
2     True
3     True
Name: AAA, dtype: bool

In [4]: %timeit df.AAA > 5
51.2 µs ± 247 ns per loop (mean ± std. dev. of 7 runs, 10,000 loops each)

In [5]: df.AAA.values > 5
Out[5]: array([False, False,  True,  True])

In [6]: %timeit df.AAA.values > 5
4.82 µs ± 22.7 ns per loop (mean ± std. dev. of 7 runs, 100,000 loops each)
```
While calculating the criteria over a column of the DataFrame takes around 51 μs, calculating the criteria over a NumPy array takes less than 5 μs, that is an order of magnitude less. This simple change can have a dramatic impact on the performance when selecting rows in a DataFrame, especially in those with tens of millions rows or more, or when there is the need to calculate the criteria repeatedly in loops.
```python
In [7]: df = read_zeek('conn.log')

In [8]: df
Out[8]:  
# . . . Lots of rows and columns . . .

[14035547 rows x 21 columns]

In [9]: %timeit df[df['id.orig_h'] == '192.168.0.130']
822 ms ± 38 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)

In [10]: %timeit df[df['id.orig_h'].values == '192.168.0.130']
186 ms ± 8.66 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```
