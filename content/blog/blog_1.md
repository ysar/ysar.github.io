Title: #001: Caveats of custom binary data formats
Date: 2023-12-03
Summary: 
Author: Yash Sarkango
Tags: Code
Slug: 001

# TL;DR
Use column-major data formats when saving a table in binary format that has more rows than columns, and row-major format if vice-versa.

# Context

Scientific data today is large and complex, requiring large-scale data repositories to disseminate and teams of people to process and analyze. Most data in the space science context is structured, and often it is multi-dimensional. However, due to the uniqueness of each instrument from which data is collected, data formats vary widely. When writing data to ASCII tables with rows and columns (i.e. a two-dimensional data structure), a choice must be made to unwrap the other dimensions into additional columns or rows.  

ASCII data is very useful for quick browsing and sharing with other researchers. However, it has serious limitations in that data files stored as plain text are much larger than their binary counterparts, and require a parser to convert text into numeric values - which uses more processing and is hence a slower process. For smaller files on the order of MBs, the difference in reading time is negligible and the benefit of readibility and shareability outweighs vastly the cost. However, using ASCII data for GBs sized data or larger is inefficient and should be avoided for the sake of all parties involved. Hence, data is often stored and shared in binary data formats, which are briefly described below.  

Binary data can be stored in a *standard* or *custom* format. *Standard* binary data formats like CDF, HDF5, netCDF, FITS, etc. are used widely. When used appropriately they are powerful and fast. Nevertheless, reading and writing in these format typically requires external libraries. This can make them cumbersome especially when using machines without administrative access or permissions. In addition, these data formats are easiest to use when handling relatively simple data structures (like large `numpy` arrays). They are more challenging (but quite fun!) to use with complex data that has layers of metadata, multiple-dimensions or groups, columns with string data types, etc. 

> **Note**: Unlike ASCII data in the form of tables of comma-separated values (CSV) files, binary data is not explicitly two-dimensional. Line breaks (e.g. `\n`) to separate rows of data are mostly meaningless in the binary context. Within memory, it is similar to a one-dimensional stream of 1s and 0s. It is how we parse this data that gives it dimensionality.

*Custom* binary formats describe the structure of the data down to every byte. For e.g., the format would prescibe that the first 4 bytes make up an integer, the next 8 bytes make up a float, etc. This customization is powerful and quite straightforward from the perspective of a data provider. From the perspective of a data reader, it is a complete nightmare.

Consider the following data table, which contains 100000 rows and 101 columns. The first 100 columns are numeric floating point numbers and the last column contains 10-byte strings. This sort of table with mixed types is quite common. 

<img src="{static}../images/blog1/table.png" alt="Picture" width="500"/>  
*(A table of random numeric and text data.)*  

> Code to generate this kind of table  

    :::python
    Nx = int(1e5)
    Ny = int(1e2)

    def randomword(length):
        letters = string.ascii_lowercase
        return ''.join(random.choice(letters) for i in range(length))
            
    def create_data(Nx, Ny):
        
        data = np.random.rand(Nx ,Ny)
        data.dtype = np.float64

        df = pd.DataFrame(columns=['n{:d}'.format(x) for x in range(Ny)], data=data)
        df['s0'] = df.apply(lambda x: randomword(10), axis=1)
        
        return df
        
    df = create_data(Nx, Ny)

This sort of row and column structure is very intuitive. Each row is a unique measurement and each column is a property of that measurement. In space science, rows are typically the time *when* the measurement was made and the columns contain information of what data was measured. Perhaps because of our intuitive familiarity with rows and columns, binary data is also stored *implicitly* as rows and columns. This can take one of two forms - the row-major approach, when the entire row is written out first, then the next, and so on; and the column-major approach, when the entire column contains data from all rows is written out at once, then the next, and so on. These two approaches are illustrated below.

<img src="{static}../images/blog1/major-order-illustration.png" alt="Picture">  
*(Row-major v/s column-major data storage.)*  


If each row signifies a unique time when the measurement was made, then it is obvious that data collected by the spacecraft and stored onboard is in the row-major format. That is, each row is written out to disk as the measurement is made. In the vast majority of cases, even after data is transmitted to the ground, processed and stored as a higher-level data product on repositories it retains its row-major format. However, in the vast majority of cases, the number of columns (data collected at each time), is much smaller than the number of rows (unique data accumulation times) in each data file. This raises the question - would it not be faster to read data that is stored in column-major format, as each column would have its own data type, and larger chunks of data could be read from the disk at once?  

# Methodology

Let us first write the `DataFrame` that we have created earlier to binary files using the row-major and column-major approaches. For the purposes of this exercise I will not use `pandas` or `numpy` inbuilt tools and opt for using the Python standard library as much as possible.  

First, we have the code for writing the data in the row-major format, where I iterate over each row of the DataFrame and save it to the binary file.  

    :::python
    def save_binary_rowmajor(df, Nx, Ny):
        "Write 1 row at at time. Each row has Ny + 1 columns"
        with open('data_binary_rowmajor.dat', 'wb') as f:
            for i, row in df.iterrows():
                # Here we iterate over rows and write all columns in that row.
                f.write(struct.pack(f'{Ny}f', *row.values[:-1]))
                f.write(struct.pack('10s', row.values[-1].encode('utf-8')))

Next, is the code for writing to the binary file in the column-major format, where I iterate over each column and save all rows in that column.  

    :::python
    def save_binary_columnmajor(df, Nx, Ny):
        """Write 1 column at a time. Each column has Nx + 1 'rows'."""
        with open('data_binary_columnmajor.dat', 'wb') as f:
            for j in range(Ny):   
                # Here we iterate over columns and write all rows at once.
                f.write(struct.pack(f'{Nx}f', *df.loc[:, f'n{j}']))

            # The last column has a different format so needs special treatment.
            for i in range(Nx): 
                f.write(struct.pack('10s', df.loc[i, 's0'].encode('utf-8')))

This creates two files `..._rowmajor.dat` and `..._columnmajor.dat` that contain all the data and in fact have the exact same size of 41000000 bytes. Now, let us write some code to read the data.  

Here is the code to read the row-major data. We have to read a series of bytes according to the format (which we know beforehand) and unpack it. This unpacking has to be done for each row, and within each row, it has to be done separately for the floats and the string.  

    :::python
    def read_binary_rowmajor(Nx, Ny):
        df = pd.DataFrame(
            index=np.arange(0, Nx, 1), 
            columns=['n{:d}'.format(x) for x in range(Ny)] + ['s0'])

        with open('data_binary_rowmajor.dat', 'rb') as f:
            for i in range(Nx):
                # Loop over each row
                # Unpack the binary data for each column in each row separately. 
                floatvals = f.read(Ny * 4)
                df.iloc[i, :Ny] = struct.unpack(f'{Ny}f', floatvals)
                stringval = f.read(10)
                df.iloc[i, -1] = stringval.decode('utf-8')
        return df

Likewise, here is the code to read the column-major data. Each column is unpacked at once. We know the data type (i.e. number of bytes per item) of the column, as well as its size. 

    :::python
    def read_binary_columnmajor(Nx, Ny):
        df = pd.DataFrame(
            index=np.arange(0, Nx, 1), 
            columns=['n{:d}'.format(x) for x in range(Ny)] + ['s0'])\

        with open('data_binary_columnmajor.dat', 'rb') as f:
            for j in range(Ny):
                # For each column, read the entire row.
                floatvals = f.read(Nx * 4)
                df.iloc[:, j] = struct.unpack(f'{Nx}f', floatvals)

            # column with strings needs special treatment.
            for i in range(Nx):
                stringval = f.read(10)
                df.iloc[i, -1] = stringval.decode('utf-8')
                
        return df

# Results: Which is faster?

Now, let us use `timeit` to see which one is faster. Here are the results on my machine.  

    :::python
    %timeit df_rowmajor = read_binary_rowmajor(Nx, Ny)
    %timeit df_columnmajor = read_binary_columnmajor(Nx, Ny)
    4.95 s ± 147 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
    1.34 s ± 36.3 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)

With 101 columns and 100000 rows, the column-major approach is approximately *4 times faster*, despite the same file size! But why? Let us use a profiler to examine our code.  

    :::python
    %load_ext line_profiler
    %lprun -f read_binary_rowmajor read_binary_rowmajor(Nx, Ny)  

<img src="{static}../images/blog1/profiler-rowmajor.png" alt="Picture" width="800"/>  
*(Profiling results - row-major storage.)*  

As highlighted in red, most of the time is spent unpacking each row. Interestingly, the time spent to read in the string column (33.7%) is more than half of the time spent to read in all the remaining 100 float values (54.9%). Reading the binary data was a 6 times faster process than unpacking the data into a float value. The entire row of 100 float values (minus the last string column) was unpacked 100000 times, with each row unpack operation taking about 49000 ns, or **490 ns/item**.

    :::python
    %lprun -f read_binary_columnmajor read_binary_columnmajor(Nx, Ny)
   
<img src="{static}../images/blog1/profiler-columnmajor.png" alt="Picture" width="800"/>  
*(Profiling results - column-major storage.)*  

In the column-major approach, we unpack the entire column of 100000 values at once. This was done for each of the 100 float columns and for the string column. The decode operation was the most expensive, and since we decoded each string separately this dominated most the processing time. Despite this, our overall time was smaller than in the row-major case. For the floating point numbers alone, each column with 100000 items took ~2.4 ms to unpack giving us a rate of --> **24 ns/item**.  

Why is the unpacking rate different? Firstly, we should note that the `struct.unpack` method is implemented in C, so any loops that are implicitly passed to the unpack method will be faster than those executed via Python. In the row-major case, there is a large loop that is executed 100000 via Python, whereas in the column-major case we have a much smaller Python loop for the floats. (The string column is more tricky that we will examine in another post).

Hence, it appears that column-major ordering of binary data is faster since it uses smaller for-loops in Python and larger loops in the underlying C implementation. But this should only be true when the number of columns is smaller than the number of rows. We can test this hypothesis by look at different combinations of `(nRows, nColumns)` and measuring the time it takes to unpack the data structure. Since absolute numbers are meaningless when comparing data of different sizes, we will compare the ratio between the time taken to read a row-major oriented file versus the time taken to read the same data in column-major format.  

    :::python
    Nx = [int(x) for x in 10**(np.linspace(1, 4, 20))]
    Ny = [int(y) for y in 10**(np.linspace(4, 1, 20))]

    row2cols = np.zeros(20)
    tratio = np.zeros(20)

    i = 0
    for nx, ny in zip(Nx, Ny):
        row2cols[i]  = nx / ny
        print(row2cols[i])
        
        df = create_data(nx, ny)

        save_binary_rowmajor(df, nx, ny)
        save_binary_columnmajor(df, nx, ny)
        
        start = time.time()
        read_binary_rowmajor(nx, ny)
        etime_row = time.time() - start

        start = time.time()
        read_binary_columnmajor(nx, ny)
        etime_column = time.time() - start

        tratio[i] = etime_row / etime_column
        
        i += 1

    fig = plt.figure()
    ax = plt.subplot(111)
    ax.semilogx(row2cols, tratio, '-o', color='k')
    ax.semilogx(row2cols[row2cols > 1], tratio[row2cols > 1], '-o', color='b')
    ax.semilogx(row2cols[row2cols < 1], tratio[row2cols < 1], '-o', color='r')
    ax.grid(which='both', ls='dotted', lw=0.5)
    ax.axvline(1, color='k', ls='dashed')
    ax.axhline(1, color='k', ls='dashed')

    ax.set_xlabel('Number of rows / Number of columns')
    ax.set_ylabel('Ratio of read time \n (row-major to column-major)')
    ax.text(1.5, 3.2, 'Column-major faster')
    ax.text(1e-3, 0.75, 'Row-major faster')
    ax.text(1e-3, 0.5, 'N$_{rows}$ < N$_{cols}$', horizontalalignment='left', color='r')
    ax.text(1.5, 3.0, 'N$_{rows}$ > N$_{cols}$', horizontalalignment='left', color='b')
    fig.savefig('images/trend.png', dpi=300, bbox_inches='tight', facecolor='w')

This gives us the following plot,

<img src="{static}../images/blog1/trend.png" alt="Picture" width="600"/>  
*(Change in speedup with changing nRows / nColumns)*  

Some errors are to be expected as we are only timing the functions once and because we are using very small numbers of the two ends of the curve with nRows=10 or nColumns=10. Nevertheless, the data shows very clearly that reading data in column-major format is faster when nColumns < nRows and slower when nColumns > nRows. This is quite intuitive and expected based on our previous hypothesis. 

# Conclusion

Scientific data is often organized in tables with each row corresponding to a unique measurement ID and each column corresponding to a property that is being measured. This kind of data (especially at Gb sizes) is often stored in the form of binary files with a custom data format for which parsers need to be developed. I show in this blog post that in higher level languages like Python, when relying on for-loops and read statements and not on external libraries, column-major ordering of such data is often a few factors faster than row-major ordering. Row-major ordering is intuitive for ASCII data where the row and column structure can be visually inspected, but it not meaningful when storing data when the number of rows vastly exceeds the number of columns.

I recommend for practical use is that all higher-end data products, if not using ASCII formats or standardized binary formats like CDF or HDF5, store data in column-major ordering.

> Note: All code and discussion shown here is purely academic. I would not recommend using it as is and point towards standard libraries whenever available. Both `numpy` and `pandas` have in-built functionalities to read binary data, which are finicky but much faster than what is shown here.

