# Pandas - Counting the Rows that Match a Criteria
#### TL;DR
#### In depth
One of my first tasks during my PhD was to study a dataset containing file transfers statistics between several computer centers across the world. Each row contained the source and destination of the transfer, the amount of bytes transferred for that file, the duration of the transfer in seconds and some other information. I needed to calculate the list of different source sites ordered by popularity. The DataFrame had the order of 200 million rows and only opening it with read_csv took several minutes. 

My first solution was very naïve. The script was iterating the rows of the DataFrame one by one. If the source of the transfer was in the keys of the dictionary, then it incremented the value for that key, otherwise, it created the key equal to the source site and put a value one.

```python
In [1]: stats = dict()
   ...: for r in df.iterrows():
   ...:     if r[1]['src_site'] in stats.keys():
   ...:         stats[r[1]['src_site']] += 1
   ...:     else:
   ...:         stats[r[1]['src_site']] = 1
```

Run time speaking, this code can hardly be worse. The code took several hours to run, and when I finally got the answer I figured out that it would be better to calculate the total amount of bytes transferred per site, instead of only the number of transfers.

Lesson learned, I put myself into the task on how to make this calculation more efficient. I could take you on a tour on how I learn to parallelize the for loop and why this is not the best solution for this particular problem. Instead, I will leave the parallelization for the next chapter and will focus on the best solutions I can find here.

One of the best options here is to use the groupby() function of the DataFrames. A detailed explanation about group by in Pandas is beyond the scope of this book, but easily found in Pandas documentation. It is enough to say that the group by  function allows us to split the DataFrame according to some criteria, map a function to each chop to obtain a single value per split and combine the chops into a single DataFrame. If you have studied functional programming, you may have noticed a similarity with filter, map and reduce. 

It is possible to be very creative with the splitting and the mapping but the basic usage is very straightforward.
```python
In [1]: df = pd.DataFrame(
   ...:     {
   ...:         "A": ["foo", "bar", "foo", "bar", "foo", "bar", "foo", "foo"],
   ...:         "B": ["one", "one", "two", "three", "two", "two", "one", "three"],
   ...:         "C": np.random.randn(8),
   ...:         "D": np.random.randn(8),
   ...:     }
   ...: )

In [2]: df
Out[2]:  
    A      B         C         D
0  foo    one -0.961128 -0.554926
1  bar    one -0.585226 -0.520179
2  foo    two  0.519780 -0.854333
3  bar  three -0.428597  0.620880
4  foo    two -0.425696  1.687110
5  bar    two -1.238831  1.262809
6  foo    one  0.911248 -0.558482
7  foo  three -1.106441 -1.273141

In [3]: df.groupby("B").sum()
Out[3]:  
              C         D
B                         
one   -0.635107 -1.633587
three -1.535038 -0.652261
two   -1.144747  2.095586
```
The groupby() function returns a GroupBy object. Once this object has been created, it is possible to select columns and apply different functions to each independently. The flexibility of the Pandas Group By capabilities, combined with its overall efficiency, is one of the best options to write these kinds of solutions. In order to solve my first problem (to check which site was more popular by the amount of transfers) I needed to artificially add a column of ones, indicating that there was one transfer per line, and then group by and sum. 

But even when adding artificially the column, while the naïve solution took roughly 7 minutes to compute the sum of occurrences in a Zeek file with 20 million rows, the group by solution takes less than a second. 

```python
In [1]: df = read_zeek('conn.log')

In [2]: %%timeit
   ...: stats = dict()
   ...: for r in df.iterrows():
   ...:     if r[1]['id.orig_h'] in stats.keys():
   ...:         stats[r[1]['id.orig_h']] += 1
   ...:     else:
   ...:         stats[r[1]['id.orig_h']] = 1
6min 50s ± 4.06 s per loop (mean ± std. dev. of 7 runs, 1 loop each)

In [3]: %%timeit
   ...: df['transfers'] = 1
   ...: df[['id.orig_h','transfers']].groupby('id.orig_h').sum()
   ...:  
891 ms ± 12.2 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

Notice that the naïve solution returns a Python dictionary, not a Pandas DataFrame. Yet, the speed up is considerable, mostly due to the inefficiency of the naïve solution. This means that is not that the groupby solution is super good, but the naïve solution is terribly bad. So, when possible, avoid writing solutions like this one in the future.

Pandas Group By is great. Is flexible and is fast, but from the Pandas documentation, I extracted the following note:
![Pandas groupby documentation warning.](bc/images/groupby_pandas_doc_warning.png)

So be careful and measure the runtime of your solutions.
