# Lab: AI Review Summaries

Amazon recently launched a new service where [AI models summarize product reviews](https://www.aboutamazon.com/news/amazon-ai/amazon-improves-customer-reviews-with-generative-ai).
For example, here's the AI generated review for [this package of uranium ore](https://www.amazon.com/dp/B000796XXM):

<img src=amazon-review-uranium.png width=100%>

In this lab, you will create your own version of this service.
You will use AI to summarize book reviews from the website <https://www.goodreads.com/>.
We'll use [a public dataset](https://mengtingwan.github.io/data/goodreads.html) that contains all activity on this website between 2006-2017.
It's approximately 30GB of data, and contains 15.7 million user reviews of 2.3 million books.

The hard part of this project will be to find the reviews for the particular books we're interested amidst all of this data.
To accomplish this, we will review basic shell commands.
<!--
Dataset URL at https://mengtingwan.github.io/data/goodreads.html from 2017

https://datarepo.eng.ucsd.edu/mcauley_group/gdrive/goodreads/goodreads_reviews_dedup.json.gz

**Learning Objectives:**

1. give you an overview of the bigdata course,
1. review how to work with csv and json files,
1. review basic python, terminal, and sql programming.
-->

## Part 0: Exploring the reviews.

You will be creating many large files in this lab.
Run the following command
```
$ cd ~/bigdata
```
to enter your `bigdata` folder.
You have 250GB of space allocated to you under this folder.

In this lab, we're going to dive right into the full dataset.
The file `/data-fast/goodreads/goodreads_reviews_dedup.json.gz` contains the full set of reviews.
Whenever you're working with a new file,
there's three common tasks that you should always do:
(a) measure the size of the file,
(b) count the number of data points,
(c) manually inspect the data.
Let's see how to do each of these in turn.

### Part 0.a: File size

The "standard" way to get the size of a file is with the `ls -l` command.
(The `-l` stands for displaying the "long" format.)
```
$ ls -l /data-fast/goodreads/goodreads_reviews_dedup.json.gz
-rw-rw-r-- 1 mizbicki mizbicki 5381053528 Oct 30 08:48 /data-fast/goodreads/goodreads_reviews_dedup.json.gz
```
In the output above,
the number `5381053528` is the number of bytes in the file.
This is not a particularly human-readable number, however, so I usually prefer to use the `du -h` command.
(`du` stands for "disk usage", and `-h` stands for "human readable output".)
This gives the result:
```
$ du -h /data-fast/goodreads/goodreads_reviews_dedup.json.gz
5.1G	/data-fast/goodreads/goodreads_reviews_dedup.json.gz
```
This is large enough that we won't be able to easily work with the file in memory and will need to use $O(1)$ memory algorithms.

### Part 0.b: Number of Lines

Our file is in [JSON lines](https://jsonlines.org/) format,
where each line has a single JSON object that represents one data point.
Therefore, counting the number of lines will tell us the number of book reviews in our dataset.

In the first part of this lab,
you already completed an exercise to count the number of lines in this file.
You should have written a command that looks something like:
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | wc -l
15739967
```
That's 15.7 million reviews.

We won't want to be passing all of these reviews into an AI model at once.
Instead, we'll need to search through these 15.7 million reviews to find just the subset of reviews about the book we're writing a summary for.

### Part 0.c: Inspect the Data

The most important task when confronting any new file is to manually inspect your file.
I've seen too many students (and professional data scientists!) skip this step.
This results in them writing code for data that they don't understand,
which makes the code not work,
which means they've wasted hours of work.
SO ALWAYS MANUALLY INSPECT YOUR DATA BEFORE DOING ANY ANALYSIS!!!

Recall that we saw before how to use the `zcat` and `head` commands to manually view the first few lines of a csv file.
The same commands will work to visualize a json file,
but the output can be a bit overwhelming.

Try running the following command:
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | head
```
You should get a huge wall of text that is difficult to read.

In order to understand this data better,
we'll need to clean up the output a bit using two new techniques:

1. Passing the `-n1` parameter to the head command.
    (This tells `head` to print only the first line of its input.
    Since this data file has one line per data point, this will result in printing the first data point.)

1. Piping the data point to python's `json.tool` pretty printer.
    (`json.tool` is a python module, and we can activate any python module from the command line with the command `python3 -m modulename`.)

Putting it all together, we get the incantation
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | head -n1 | python3 -m json.tool
```
Which gives output like
```
{
    "user_id": "8842281e1d1347389f2ab93d60773d4d",
    "book_id": "24375664",
    "review_id": "5cd416f3efc3f944fce4ce2db2290d5e",
    "rating": 5,
    "review_text": "Mind blowingly cool. Best science fiction I've read in some time. I just loved all the descriptions of the society of the future - how they lived in trees, the notion of owning property or even getting married was gone. How every surface was a screen. \n The undulations of how society responds to the Trisolaran threat seem surprising to me. Maybe its more the Chinese perspective, but I wouldn't have thought the ETO would exist in book 1, and I wouldn't have thought people would get so over-confident in our primitive fleet's chances given you have to think that with superior science they would have weapons - and defenses - that would just be as rifles to arrows once were. \n But the moment when Luo Ji won as a wallfacer was just too cool. I may have actually done a fist pump. Though by the way, if the Dark Forest theory is right - and I see no reason why it wouldn't be - we as a society should probably stop broadcasting so much signal out into the universe.",
    "date_added": "Fri Aug 25 13:55:02 -0700 2017",
    "date_updated": "Mon Oct 09 08:55:59 -0700 2017",
    "read_at": "Sat Oct 07 00:00:00 -0700 2017",
    "started_at": "Sat Aug 26 00:00:00 -0700 2017",
    "n_votes": 16,
    "n_comments": 0
}
```
The JSON object above represents the first review in our dataset.
This object contains a variety of information about the review,
but notice that the title of the book is not stored in the JSON object.
The only information we have about the book is the `book_id` field.
A different file `goodreads_books.json.gz` associates each `book_id` with information about the book, like the title, author, and publication year.
<!--
Storing this information in a separate file is called *normalizing* the dataset.
We will talk later in this class about the various advantages of normalizing and denormalizing datasets,
but the main idea is that normalized datasets can save a lot of disk space by not repeating the same information multiple times.
For example, we will see in a bit that goodreads stores a lot of information about each book.
In addition to the title, there is 
-->
In order to figure out which book reviews correspond with which titles,
we will have to *join* the `goodreads_reviews_dedup.json.gz` and `goodreads_books.json.gz` files together.
Joining datasets is a notoriously difficult and time consuming process.
We will spend a considerable amount of time in this class discussing how to join correctly and efficiently.
For this lab, we will use a simple, manual, and slow method.
Later in the course, we'll learn more complicated, automatic, and faster methods.

## Part 1: Convert a book id into a title.

The first step is to familiarize ourselves with the `goodreads_books.json.gz` file.

> **Exercise:**
>
> Recall that three good steps to familiarize yourself with a new file are: (1) check the size of a file, (2) count the number of lines in the file, and (3) and manually inspect that file.
> Write one line bash commands to complete each of these tasks.
> You can use the commands from Part 0 above as a guide.

<!--
```
$ du -h /data-fast/goodreads/goodreads_books.json.gz
2.1G	/data-fast/goodreads/goodreads_books.json.gz
$ zcat /data-fast/goodreads/goodreads_books.json.gz | wc -l
2360655
$ zcat /data-fast/goodreads/goodreads_books.json.gz | head -n1 | python3 -m json.tool
{
    "isbn": "0312853122",
    "text_reviews_count": "1",
    "series": [],
    "country_code": "US",
    "language_code": "",
    "popular_shelves": [
        {
            "count": "3",
            "name": "to-read"
        },
        {
            "count": "1",
            "name": "p"
        },
        {
            "count": "1",
            "name": "collection"
        },
        {
            "count": "1",
            "name": "w-c-fields"
        },
        {
            "count": "1",
            "name": "biography"
        }
    ],
    "asin": "",
    "is_ebook": "false",
    "average_rating": "4.00",
    "kindle_asin": "",
    "similar_books": [],
    "description": "",
    "format": "Paperback",
    "link": "https://www.goodreads.com/book/show/5333265-w-c-fields",
    "authors": [
        {
            "author_id": "604031",
            "role": ""
        }
    ],
    "publisher": "St. Martin's Press",
    "num_pages": "256",
    "publication_day": "1",
    "isbn13": "9780312853129",
    "publication_month": "9",
    "edition_information": "",
    "publication_year": "1984",
    "url": "https://www.goodreads.com/book/show/5333265-w-c-fields",
    "image_url": "https://images.gr-assets.com/books/1310220028m/5333265.jpg",
    "book_id": "5333265",
    "ratings_count": "3",
    "work_id": "5400751",
    "title": "W.C. Fields: A Life on Film",
    "title_without_series": "W.C. Fields: A Life on Film"
}
```
-->

You should see a large number of JSON fields,
but for our purposes the most important are the `title` and `book_id` fields.
In order to find the title of the book that corresponds to the book review we found above,
we need to search through the entire `goodreads_books.json.gz` file for a line with the appropriate `book_id` and extract the `title` field from this entry.
We will use the `grep` tool to search and a new command `jq` to extract.

### Part 1.a: Searching with `grep`

<!--
Our goal is to search through the `goodreads_books.json.gz` to find the `book_id` that corresponds to *The Name of the Wind*,
then we will search through `goodreads_reviews_dedup.json.gz` to find the corresponding reviews.
-->

`grep` is the standard tool for searching files on the command line.

> **Note:**
>
> There are many competing implementations of the `grep` tool.
> Linux machines traditionally use the GNU grep implementation,
> which is particularly efficient.
> GNU grep uses the [Boyer-Moore](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_string-search_algorithm) string searching algorithm,
> which has the nice property that it doesn't even need to examine every byte in the input stream!
> The author has a [mailing list post from 2010](https://lists.freebsd.org/pipermail/freebsd-current/2010-August/019310.html) where he describes the implementation details that I recommend (but don't require) you to read.

`grep` has no knowledge of JSON and works on any text file.
It takes a regular expression as a command line argument,
and removes all lines in its input that don't satisfy that regular expression.
In order to find the book with id `24375664`,
we will use the following regular expression to match the way this information is stored in the JSON object: `"book_id": "24375664"`.
The final incantation is
```
$ zcat /data-fast/goodreads/goodreads_books.json.gz | grep '"book_id": "24375664"'
```
The command above should output a very large JSON object.

> **NOTE:**
>
> Pay close attention to the quotation marks in the previous paragraph.
> Since the regex contains a space, it must be enclosed in quotation marks for the entire regex to be considered a single parameter to `grep`.

> **Note:**
>
> `grep` and `zcat` are *streaming* tools.
> This means they output their results as they find them, not when the program ends.
> You will therefore see the JSON object get printed to the screen before the programs end and return control to the terminal.
> You will know that the programs have ended when the command prompt `$` is printed to the screen.
> Streaming tools can be confusing at first, but they are the reason for the $O(1)$ memory performance.

### Part 1.b: Parsing the JSON

The JSON object associated with each book is large.
Even if we use `python3 -m json.tool` to pretty print that output,
it will be annoying to manually search through the entire object to find the `title` field.
We will use the `jq` command to extract the title field for us automatically.

`jq` is a popular command line tool for parsing JSON objects,
but it has a [notoriously complicated syntax](https://jqlang.github.io/jq/manual/).
Fortunately extracting just a single field from the JSON object is not too bad:
all you have to do is pass the field name prepended with a dot.
For example, `.title` will extract the `title` field from the json object and print it to the screen.

Combining `zcat`, `grep`, and `jq` gives us the following 1-liner for finding our book title.
```
$ zcat /data-fast/goodreads/goodreads_books.json.gz | grep '"book_id": "24375664"' | jq '.title'
```
You should get that the title for our first book review was
```
"The Dark Forest (Remembrance of Earth’s Past, #2)"
```

## Part 2: Get the reviews for a title

In the previous sections, we saw how to extract a review from `goodreads_reviews_dedup.json.gz` and then join with the file `goodreads_books.json.gz` to get the title of the book that was reviewed.
What we're really interested in, however, is going in the opposite direction.
We need to start with a title, and then find all of the reviews for that title.
This will turn out to be quite a bit trickier due to some messiness in the data.
As a running example, we will use the best selling fantasy novel *The Name of the Wind* (NOTW for short).

First, we'll see that there are multiple `book_id`s for NOTW.
Then we'll see how to perform our join using multiple `book_id`s at once.

### Part 2.a: Getting the `book_id`(s) for our Title

Consider the following command.
```
$ zcat /data-fast/goodreads/goodreads_books.json.gz | grep '"title": "The Name of the Wind"' | wc -l
```
This counts all of the entries in the `goodreads_books.json.gz` file with a `title` field equal to `The Name of the Wind`.
It turns out that there are 4 different entries in `goodreads_books.json.gz` for the title *The Name of the Wind*, each with their own `book_id`.
These 4 entries correspond to different editions of the book.
(There's a hardcover, a softcover, an audiobook, and a "Tenth Anniversary Edition" hardcover.)

Our command to count the number of entries took a long time to run because it needed to loop over the entire dataset.
To save time in the future, it is often good practice to store the results of intermediate steps into a file.
We can do this using the output redirection `>` operator.
The following command will take all of the entries for our book and store them in a file called `books-notw.json`.
```
$ zcat /data-fast/goodreads/goodreads_books.json.gz | grep '"title": "The Name of the Wind"' > books-notw.json
```
Now we can quickly count the number of books:
```
$ cat books-notw.json | wc -l
4
```
And we can do other processing on these books much more efficiently.
For example, we can extract the title from each of these books with the command
```
$ cat books-notw.json | jq '.title'
"The Name of the Wind"
"The Name of the Wind"
"The Name of the Wind"
"The Name of the Wind"
```
Notice that they are all the same and match our regex.
The differences between each entry are in the `format` and `edition_information` fields:
```
$ cat books-notw.json | jq '.format'
"Audio"
"Paperback"
"Hardcover"
"Hardcover"
$ cat books-notw.json | jq '.edition_information'
""
""
""
"Tenth Anniversary Edition"
```
And each entry has a different `book_id` field:
```
$ cat books-notw.json | jq '.book_id'
"17353642"
"18741780"
"12276953"
"34347493"
```
Recall that our overall goal is to get a summary of how readers review the book.
For this task, we'll want to summarize all the reviews for all editions, and not just a particular edition.
Therefore, we'll need to find the reviews for each of these `book_id`s.

### Part 2.b: Computing the Join

To complete our join, we need to find all of the lines in the `goodreads_reviews_dedup.json.gz` file where the `book_id` field is one of the 4 `book_id`s above.
As mentioned earlier, `grep` is the tool of choice anytime we are filtering files on the command line,
so we need to write a `grep` command to complete this join.

One way of doing that would be to write 4 grep commands (one for each of the `book_id`s),
using output redirection to concatenate the results together.
(**You don't need to run these commands.**)
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep '"book_id": "17353642"' >> reviews-notw.json
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep '"book_id": "18741780"' >> reviews-notw.json
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep '"book_id": "12276953"' >> reviews-notw.json
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep '"book_id": "34347493"' >> reviews-notw.json
```
But this is inefficient because each of these commands loops over the dataset separately,
and it's awkward to have to manually repeat all of these commands.

It is more efficient to take advantage of `grep`'s regular expression ability to combine all of our search criteria into a single regex, and a single call to `grep`.
We'll use regular expression [alternations](https://www.regular-expressions.info/alternation.html).
Recall that an alternation is expressed syntactically with the pipe character `|` but has the semantic meaning of "or" inside of a regular expression.
Thus, the regular expression
```
"17353642"|"18741780"|"12276953"|"34347493"
```
semantically means: find the string `"17353642"` or `"18741780"` or `"12276953"` or `"34347493"`.
This is the regex that we want to use with grep.

The following command will extract all reviews with any of these four book ids.
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep -E '"17353642"|"18741780"|"12276953"|"34347493"' > reviews-notw.json
```
The file `reviews-notw.json` now contains the output of our join.

> **Note:**
>
> Programmers have a culture of speaking with precision,
> and I have used precise language above which I want to call to your attention.
> Observe that the regular expression is `"17353642"|"18741780"|"12276953"|"34347493"` (without single quotes), and the argument being passed to grep is `'"17353642"|"18741780"|"12276953"|"34347493"'` (with single quotes).
> We need single quotes around the regex to create the argument so that the `|` operator is not interpreted as a pipe by the shell but gets passed to grep.

> **Note:**
> 
> Manually generating the regex, then copy/pasting it into our shell command is a bit awkward when the number of `book_id`s is large.
> In this note, we'll see how to automate that process.
>
> The following "simple" shell 1-liner generates the regex for finding `book_id`s:
> ```
> $ echo $(cat books-notw.json | jq '.book_id') | sed 's/ /|/g'
> "17353642"|"18741780"|"12276953"|"34347493"
> ```
> This is a rather cryptic incantation, and we'll break it down step-by-step to understand it.
> 
> 1. First we'll take a look at the command substitution `$( ... )`.
>   We've already seen that the `cat notw.json | jq '.book_id'` outputs the 4 `book_id`s each on their own line.
> 1. We pass that result to `echo` which replaces all the newlines with spaces.
>   This has the effect of putting all of the `book_id`s onto a single line.
> 1. Finally, `sed` places the alternation operator `|` between each `book_id`.
>
> It's okay if this command feels like magic to you right now.
> By the end of this semester, writing these 1-line shell commands should be easier for you than doing the copy/paste necessary to write out the regex manually :)
>
> Now, instead of manually copying pasting the regex into the command above,
> we can use command substitution:
> ```
> $ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep -E $(echo $(cat books-notw.json | jq '.book_id') | sed 's/ /|/g') > reviews-notw.json
> ```

### Part 2.c: Sanity Checking

We're not quite done.
We still need to sanity check our results.

<!--
> **Note:**
>
> One of the most common traps that students fall into when doing data analysis projects is not sanity checking your results.
That's even more common in big data projects because the results take so long to compute,
that when you finally get a number you're ready to just move onto the next phase of your project.
-->

A simple way to sanity check is to count the number of reviews we've found:
```
$ cat reviews-notw.json | wc -l
14
```

Hmmm... that's not a lot of reviews...

*The Name of the Wind* is a New York Times best selling book that has millions of copies sold.
14 reviews seems WAY too few.

What happened is that there are many more `book_id`s that correspond to *The Name of The Wind* that we didn't find.
It turns out that the `title` field inside of the `goodreads_books.json.gz` dataset is not guaranteed to be *equal* to the title of the book, like we implicitly assumed above.
Instead, the `title` field is only guaranteed to *begin* with the title of the book.
It is allowed to contain extra information about the book after the title.
Therefore, when searching for a book, we do not want to perform an exact match against the book title.

Recall that in Part 2.a above, we searched the file `goodreads_books.json.gz` with the following regex:
```
"title": "The Name of the Wind"
```
To find `title` fields that only begin with the title of our book, we make the simple change of removing the trailing `"` to get
```
"title": "The Name of the Wind
```
Now, if there is more information in the `title` field,
this new regex will still match.

With this regex, we will find many more than 4 JSON objects:
```
$ zcat /data-fast/goodreads/goodreads_books.json.gz | grep '"title": "The Name of the Wind' > books-notw-full.json
$ cat books-notw-full.json | wc -l
35
```
We can see that our new command captures several different titles that a human would "obviously" consider to be the same book:
```
$ cat books-notw-full.json | jq '.title'
```
And we can see that these all have many different format/edition combinations:
```
$ cat books-notw-full.json | jq '.format'
$ cat books-notw-full.json | jq '.edition_information'
```

> **Exercise:**
>
> Repeat the join process from Part 2.b with the new set of 35 `book_id`s we found above,
> and store the reviews in a file `reviews-notw-full.json`.
> You may (or may not) find the note at the end of Part 2.b helpful.
>
> Then do a sanity check of counting the number of reviews you found.
> You know you did everything correctly if you get a number a bit less than 6000.

<!--
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep -E $(echo $(cat books-notw-full.json | jq '.book_id') | sed 's/ /|/g') > reviews-notw-full.json
$ wc -l reviews-notw-full.json
5992
-->

You won't be able to complete the next section until you've completed the exercise above.

## Part 3: Working with AI

Now that we have our short list of reviews for a particular title,
it will be easy to pass these reviews to an AI tool for summary.

> **Note:**
>
> If you do not already have the llm tool setup, then follow the instructions at <https://github.com/mikeizbicki/lab-llm>.

Our first step is [prompt engineering](https://platform.openai.com/docs/guides/prompt-engineering), which is just writing out in plain English instructions what we want the LLM to do.
The following command will output a reasonable prompt to your terminal.
```
$ echo "Write a short 2-3 sentence summary of the following book reviews. The reviews are: $(cat ./reviews-notw-full.json | jq '.review_text')"
```
Now we can use the pipe to pass this prompt to our llm with a command like
```
$ echo "Write a short 2-3 sentence summary of the following book reviews. The reviews are: $(cat ./reviews-notw-full.json | jq '.review_text')" | llm
```
This command should give you an error that looks something like
```
Error: Error code: 413 - {'error': {'message': 'Request too large for model `llama-3.1-70b-versatile` in organization `org_01j5vb7h4tfz0tgw7e6vxfj3er` service tier `on_demand` on : Limit 200000, Requested 1155360, please reduce your message size and try again. Visit https://console.groq.com/docs/rate-limits for more information.', 'type': '', 'code': 'rate_limit_exceeded'}}
```
This is because your prompt is too large.
Let's use the `wc` command to see how large the prompt is.
```
$ echo "Write a short 2-3 sentence summary of the following book reviews. The reviews are: $(cat ./reviews-notw-full.json | shuf | jq '.review_text')" | wc
   5992  844126 4621422
```
The first number above is the number of lines;
the second is the number of words;
and the third is the number of characters.
(Recall that we previously used `wc -l` to count the number of lines, which outputs only the first number above.)

The Groq API currently limits requests to be only 8096 tokens in length.
Tokens are semantically meaningful parts of words,
and so the number of words is a lower bound on the number of tokens.
We are clearly above this threshold, and so we need to truncate our text to fit the threshold before passing it to the LLM.

> **Exercise:**
> 
> Modify the command above so that it selects only 20 reviews to pass to the llm.
> You can use the `head` command with the `-n` parameter.

<!--
$ echo "Write a short 2-3 sentence summary of the following book reviews. The reviews are: $(cat ./reviews-notw-full.json | head -n20 | jq '.review_text')" | llm
-->

<!--
Instead, we will be using a commandline interface called [llamafile](https://github.com/Mozilla-Ocho/llamafile).

Llamafile is an open source project developed by Mozilla (the non-profit best known for developing firefox).
It is a fairly recent project, having only been [announced in November 2023](https://hacks.mozilla.org/2023/11/introducing-llamafile/).
The idea of llamafile is that every LLM can be packaged as a single, standalone executable file that can easily be combined with other unix processes using pipes.
With llamafiles, using LLMs from the shell is trivial.

We'll first see how to download/use a llamafile,
then we'll combine it with our dataset to generate our review summaries.

> **Note:**
>
> You can find lots of examples of cool uses of llamafiles at the [developer's webpage](https://justine.lol/oneliners/).

### Part 3.a: Getting the Llamafile

There are many different LLMs to choose from,
and each has a different llamafile.
For this project we'll use the [Mistral LLM](https://mistral.ai/news/announcing-mistral-7b/).
This is a popular LLM because it is open source (Apache 2.0 license) and has a good balance of output quality and speed.

Get started by using the `wget` command to download the Mistral LLM llamafile.
```
$ wget 'https://huggingface.co/jartine/Mistral-7B-Instruct-v0.2-llamafile/resolve/main/mistral-7b-instruct-v0.2.Q5_K_M.llamafile'
```
You should be averaging speeds over 150MB/s.
The Claremont Colleges have a very fast internet connection,
but your laptops are limited by wifi bandwidth and so can only achieve about 1MB/s downloads.
The lambda server, in contrast, is connected by physical fiber optic cables to the internet.

LLMs use a lot of disk space.
```
$ du -h mistral-7b-instruct-v0.2.Q5_K_M.llamafile
4.9G	mistral-7b-instruct-v0.2.Q5_K_M.llamafile
```
When loaded into memory,
this model takes just a bit less than 16GB of ram to run.
LLMs are also notoriously slow computationally and cannot be reasonably run on most laptops.

The llambda server, however, is more powerful than your laptop and can generate text from these models in essentially real time.

### Part 3.b: Running the LLamafile

Once the download completes, try running the llamafile with the command
```
$ ./mistral-7b-instruct-v0.2.Q5_K_M.llamafile
```
You should get a `Permission denied` error.
For security reasons, all files that are downloaded from the internet by default do not have execute permissions set.

In this case, we trust the downloaded executable file, and so we can manually give executable permissions with the `chmod` command:
```
$ chmod u+x mistral-7b-instruct-v0.2.Q5_K_M.llamafile
```
Now, when you run the file
```
$ ./mistral-7b-instruct-v0.2.Q5_K_M.llamafile
```
You should get a large amount of debugging output followed by the line
```
llama server listening at http://127.0.0.1:8080
```
By default, llamafile provides a web interface for interacting with the LLM.
But we won't be using that interface,
and will instead use the command line interface.

Press `CTRL-C` to end the program and return to the command prompt.

We can pass the `-f` flag to our llamafile in order to specify that the input should come from a file instead of the web interface.
We will be combining `-f` with the special file `/dev/stdin` to enable piping into the llamafile.
Here's an example command:
```
$ echo "[INST]Write 1 paragraph explaining why the shell is important for big data.[/INST]" | ./mistral-7b-instruct-v0.2.Q5_K_M.llamafile -f /dev/stdin
```
You should see a large amount of debug output, followed by en explanation of why the shell is important for big data.

> **Computational Note:**
>
> The command above is extremely compute intensive.
> All of the previous commands that you've run have only used a single processor,
> and therefore they will take the same amount of time no matter how many people are using the lambda server.
> The mistral command, however, is highly parallel and will use all available compute resources on the lambda server (80 CPUs).
> The command took me 30 seconds to complete when the lambda server was completely idle,
> and will take longer as the lambda server comes under load.
> The overall runtime will be proportional to the number of students currently running the command,
> so if 10 students are running the command,
> expect it to take about 300 seconds to complete.
>
> Subsequent steps for this lab do not depend on the output of the command above,
> so you can stop the command early (by pressing `CTRL-C`) if you'd like.

The string inside of the echo command above
```
[INST]Write 1 paragraph explaining why the shell is important for big data.[/INST]
```
is called the *prompt* for our large language model.
The prompt provides natural language instructions for what the model should do.
[The Mistral website](https://www.promptingguide.ai/models/mistral-7b#mistral-7b-instruct) has guidelines for how to write good prompts,
but this is still an active area of research.
The most important thing is that the prompt should be surrounded by the `[INST]` and `[/INST]` tags
and contain instructions on what the AI should attempt to do.

### Part 3.c: Writing our Prompt

Our next step is to prepare a prompt for the LLM.
We will use a multi-line string with command substitution.

Run the following command.
```
$ echo "[INST]
hello world
[/INST]"
```
Notice that the shell allows multiline strings.
Depending on your shell's settings, you may see a `>` symbol prompting you at the beginning of each line of the string.
These `>` prompts are customarily not provided in tutorials because they make copy/pasting more difficult.

Now we will introduce command substitution.
Run the following command.
```
$ echo "[INST]
$(echo hello world)
[/INST]"
```
You should get the same output as the previous command.
The `$( ... )` syntax is called *command substitution*.
When the shell encounters this syntax, it runs the command within the parentheses (in this case `echo hello world`),
and places the output of that command (in this case `hello world`) within the string.

We can now use command substitution to build our prompt.
One possible prompt could look like
```
$ echo "[INST]
Write 1 short paragraph that combines all of the following book reviews into a single summary of the book.
The reviews are:
$(cat ./reviews-notw-full.json | jq '.review_text')
[/INST]
"
```

This prompt will work, but it is very long because we have so many reviews.
LLMs are notoriously computationally expensive, and so we will get results faster if we use only a small sample of the reviews.

> **Exercise:**
> 
> Modify the `echo` command that generates the prompt so that only 20 reviews are used in the prompt.
> You will have to use the `head` command with the `-n` parameter.
-->

## Part 4: Creating a Shell Script

Our goal in this section is to combine everything together into a convenient script that will generate summaries.

### Part 4.a: Shell Scripting Basics

Use Vim to create a file `summarize_reviews.sh` that contains the following contents.
```
#!/bin/sh
echo "book to summarize is: $1"
```
This file is called a *shell script*,
and it can be run with the `sh` program:
```
$ sh summarize_reviews.sh
book to summarize is:
```
Notice that the `$1` variable contains no text,
and so nothing is output to the screen.

`$1` is a special variable that is set automatically by the `sh` program when it runs a shell script,
and it contains the first parameter to the program.
If you rerun the command with the name of a book as a parameter,
then the shell script will print the name of the book:
```
$ sh summarize_reviews.sh "Name of the Wind"
book to summarize is: Name of the Wind
```
Notice that quotation marks are necessary in order for the entire book name to be interpretted as a single parameter.
```
$ sh summarize_reviews.sh Name of the Wind
book to summarize is: Name
```

> **Exercise:**
>
> Modify `summarize_reviews.sh` so that it returns a summary of the reviews for whatever book the user passes in as an argument.
> You should verify that your program works by running 
> ```
> $ sh summarize_reviews.sh "Name of the Wind"
> ```
> and verifying that you continue to get similar results as before.
> The results won't be exactly the same because the `llm` command is non-deterministic.

### Part 4.b: Cleaning up the Results

Your shell script from part 4.a likely created a number of files to store intermediate results.
It is annoying to have these intermediate files "polute" your current working directory,
And so it is common practice to store these files in a temporary directory.

This is easy to do in a shell script by putting the following commands at the top of your script.
```
tempdir=$(mktemp -d)
cd "$tempdir"
```
The `mkdtemp -d` command creates a temporary folder and outputs the name of this folder to stdout.
The `cd` command then changes the working directory to this folder,
so all subsequently created files will be in this temporary folder instead of the folder you ran the command from.

These temporary folders are located in the `/tmp` directory of the computer,
which are regularly deleted.
It is still polite, however, to automatically clean up your temporary folder at the end of your shell script with a command like
```
rm -rf "$tempdir"
```

> **Exercise:**
>
> Modify your shell script form Part 4.a so that it does not create any intermediate files in the folder the user called it from.

## Submission

Run the following commands in a terminal session to summarize the reviews of some classic CMC books.
Copy/paste the commands and output into sakai.
```
$ cat summarize_reviews.sh
$ sh summarize_reviews.sh "Wealth of Nations"
$ sh summarize_reviews.sh "Democracy in America"
```
Like in the first part of this lab, these commands take a long time to run.
The majority of the time is not spent in the LLM,
but in finding the data that goes into the LLM.
By the end of the course, you will be able to create systems that do this in milliseconds.
