---
date: 2019-12-17 07:50:54 +0500
title: Snappy vs Zstd for Parquet in Pyarrow
summary: Comaring Zstandard and Snappy Compression for Parquet Files Using Pyarrow
category: Programming
tags: Python, Parquet, Data Storage, Pandas
author: Levi Sands
layout: post
comments: true
lang: en
---
<!--
image: /images/stress-apple.jpg
imageMeta:
  attribution:
  attributionLink: https://cdn.pixabay.com/photo/2015/02/04/20/31/stress-624220_960_720.jpg
-->

[TOC]

![https://cdn.pixabay.com/photo/2015/02/04/20/31/stress-624220_960_720.jpg]({static}../images/articles/black_logo.png)

I am working on a project that has a lot of data. In the process of extracting from its original bz2 compression I decided to put them all into parquet files due to its availability and ease of use in other languages as well as being just able to do everything I need of it.

By default pandas and dask output their parquet using snappy for compression. This uses about twice the amount of space as the bz2 files did but can be read thousands of times faster so much easier for data analysis. I recently became aware of zstandard which promises smaller sizes but similar read speeds as snappy.

I recently decided to see if it was worth the extra code to use pyarrow rather than pandas to read and package this data in order to save some space on my hard drive. Below is all of the code I used to test this.

First up is the actual test. I first create some text files to log the time spent on each operation. I then open one of my snappy compressed files containing 4,000,000 rows and around 90 columns. I record the time it takes to:

1. Read that into pandas
1. Save as a snappy parquet
1. Write to a zstd parquet
1. Read and import to pandas from the zstd parquet

```python
import time
import pathlib
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
import altair as alt

p = pathlib.Path(".")
files = p.glob("RC_*.parquet")

open
file = []
for item in files:
    file.append(item)
file = file[0]
print(file)
i = 0
read = open("read_snappy.txt", mode="a")
write = open("write_snappy.txt", mode="a")
readz = open("read_zstd.txt", mode="a")
writez = open("write_zstd.txt", mode="a")

while i < 100:
    start_time = time.time()
    dta = pq.read_table(file)
    dta = dta.to_pandas()
    elapsed_time = (time.time() - start_time) / 60
    test = str(elapsed_time) + "\n"
    read.write(test)

    start_time = time.time()
    dta.to_parquet("test.parquet")
    elapsed_time = (time.time() - start_time) / 60
    test = str(elapsed_time) + "\n"
    write.write(test)

    start_time = time.time()
    dta = pa.Table.from_pandas(dta)
    pq.write_table(dta, "test.parquet", compression="zstd")
    elapsed_time = (time.time() - start_time) / 60
    test = str(elapsed_time) + "\n"
    writez.write(test)

    start_time = time.time()
    dta = pq.read_table("test.parquet")
    dta = dta.to_pandas()
    elapsed_time = (time.time() - start_time) / 60
    test = str(elapsed_time) + "\n"
    readz.write(test)
    i += 1
    print(i)

read.close()
write.close()
readz.close()
writez.close()
```

Now for plotting the results

The code does the following:

1. Open the files made above
1. For each of the files it adds each time recorded to a list in float format
1. Then the average is taken for each of the actions

```python
read = open("read_snappy.txt", mode="r")
write = open("write_snappy.txt", mode="r")
readz = open("read_zstd.txt", mode="r")
writez = open("write_zstd.txt", mode="r")

read_sn = []
file_in = open("read_snappy.txt", mode="r")
for y in file_in.read().split('\n'):
    try:
        read_sn.append(float(y))
    except:
        pass

write_sn = []
file_in = open("write_snappy.txt", mode="r")
for y in file_in.read().split('\n'):
    try:
        write_sn.append(float(y))
    except:
        pass

readz = []
file_in = open("read_zstd.txt", mode="r")
for y in file_in.read().split('\n'):
    try:
        readz.append(float(y))
    except:
        pass

writez = []
file_in = open("write_zstd.txt", mode="r")
for y in file_in.read().split('\n'):
    try:
        writez.append(float(y))
    except:
        pass

read_sn = sum(read_sn)/len(read_sn)
write_sn = sum(write_sn)/len(write_sn)
readz = sum(readz)/len(readz)
writez = sum(writez)/len(writez)
```

Finally to plot the first chart showing the speed of the four actions

```python
dta = pd.DataFrame({
    "time_in_min": (read_sn, write_sn, readz, writez),
    "category": ("read_snappy", "write_snappy", "read_zstd", "write_zstd"),
})

chart = alt.Chart(dta).mark_bar().encode(
    y='time_in_min',
    x='category'
).properties(
    title="Read and write time (in minutes)"
)

chart.save("chart1.html")
```

![Read and write speeds of snappy vs zstd](https://github.com/ldsands/Levi-Sands-blog-site/blob/master/pics/snappyVSzstd/snappyVSzstd_chart1.png?raw=true)

To wrap it up here is the code to make a quick graph showing the difference in the size on the disk taken by each format.

```python
dta = pd.DataFrame({
    "size_in_MB": (830.239, 549.682),
    "compression": ("snappy", "zstd"),
})

alt.Chart(dta).mark_bar().encode(
    y='size_in_MB',
    x='compression'
).properties(
    title="Size of file (in MB)"
)

chart.save("chart2.html")
```

![Size taken on disk of snappy vs zstd](https://github.com/ldsands/Levi-Sands-blog-site/blob/master/pics/snappyVSzstd/snappyVSzstd_chart2.png?raw=true)
