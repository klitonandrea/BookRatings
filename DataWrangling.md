Data Wrangling

Retrieved data from dumps in Open Library Data Dumps (https://openlibrary.org/developers/dumps)

The data comes with the following format:

    - type - type of record (/type/edition, /type/work etc.)
    - key - unique key of the record. (/books/OL1M etc.)
    - revision - revision number of the record
    - last_modified - last modified timestamp
    - JSON - the complete record in JSON format

For this project we are focused on the last record which comes in JSON format.

It is necessary to transform the last column to the necessary structured table format to work with. In this project we are using pandas python package.

First we read the dump using the command:
	olde = pd.read_csv(path_to_dump, sep='\t', header=None, names=['Book'], usecols=[4])

This will result in a table with one column containing the json. The dump is huge and takes more than 16 GB of RAM. The dump was split in 5 files containing around 29602272 records each.

The transformation is applied in chunks of size 1000. We use Dewey classification to filter out all non-literature works. Will retain only those records starting with '8' or a letter. For each record we apply json_normalize function from 'pandas.io.json' module. The tranformation is not time efficient and going through all the records was taking a huge amount of time. Had to restrict the number of records for this project. There were chosen first 4000 literature works from each file. 

The resulting table contains 82 columns. Columns that contain identification information are discarded:

`BookInfo.drop([item for l in filter(None, BookInfo.columns.str.findall('identi.*')) for item in l], axis=1, inplace=True)`

Only ISBN will serve as a book identifier, although the same work from the same author could have several ISBNs. ISBN is not only the literature work identifier but also it is linked to the publisher. The ISBN is necessary to retrieve rating information from Goodreads (https://www.goodreads.com).

Goodreads provides API to retrieve the ratings from ISBNs. The ISBNs were parsed from the original table:

```
onlyISBN = booksDF[~booksDF.isbn_10.isnull()]
splittedISBN = onlyISBN.isbn_10.str.findall("\d*[a-z]*")
splittedISBN=splittedISBN.sum()
allISBNs=list(filter(None, splittedISBN))
```

The ratings table is built from Goodreads ratings. The ratings were retrieved in chunks of 500 and json_normalize(d) to build a pandas dataframe. This table contains 11 columns. The most important is *average_rating*. We will use this value as the dependent variable.

The isbns are provided as strings in the following format: "['isbn1', 'isbn2']". Need to get them to a list object to manipulate:

`BookInfo['isbn10'] = BookInfo.isbn_10.apply(lambda x: eval(str(x)) if type(x)==str else '')`

To complete the book table it is necessary to add another column 'average_rating':

`BookInfo['average_rating'] = BookInfo.isbn10.apply(lambda x: BookRatings[BookRatings.isbn10.isin(x if type(x)=='list' else list(x))].average_rating.mean()`

Not all the books that we have retrieved do have a rating in Goodreads. We will remove the records without an average rating:

`BkInfNNullRtng = BookInfo[BookInfo.average_rating.notnull() ]`

After a quick look at the histogram:

`BkInfNNullRtng.average_rating.hist()`

It seems that there are a lot of 0 rated books. These books are not rated yet in Goodreads. Removing them as well:

`BookInfoAverage = BkInfNNullRtng[BkInfNNullRtng.average_rating != 0]`

Now there are 12265 records that have not null non-zero average ratings.
