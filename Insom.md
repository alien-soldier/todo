# To Do List :octocat:

---
## *levelup*
* Clean code principals
* Using Mypy
* Unit test
* Pylint
* Using markdown for documenting
* Does Should I use Git and Github ???
---
## *general consideration*
* Writing corresponding raw query for all ORM query and saving all of them in one place with a key number which commented on the scripts
* Matching on calculating the r is based on gdate, can I use another things like incremental index for matching?
* on repair mode 'end' is before than 'guaranteed' which lead to some unreasonable  thing in dag. That doesn't matter
* we should replace all pd.to_numeric(downcast=integer) to  pd.to_numeric(downcast=float) and in table schema bigint to decimal. we did it only for 98
---
## *Pythonic consideration* 
* Increment parsing is available on Zeep Library. maybe you should have a look at it
* using decorator, generator
* we used some cool parsing string which was able to filter all non printable and punctuation from symbol. it was a good experiment.

---
## *Postgresql consideration* 
* upsert
* semi_index 


---

## *DB Utility*


---

## *L1*

### *C1 Asset*

1. Initial asset in 950301 in new version has some record more than previos version

2. Initail asset has some records with zero asset, maybe we should filter them before db insertion
3. Writing a script for calculating the portfolio on a date which is needed for making the initial asset of every new period. It is obvious we should run it for all databases and  add initial asset to them
4. Allowed limit for each asset table is about 10 million records, which needs some slicing on period. for example 95 and 96 have less than 10 million record and year period is good for them. 97 has 14 million records which should be divided to two semi year period. in 98 each quarter has 10 million which needs to be treated as a independent period.
naming convention for database follows this slicing necessity.some example: _95_y, 96_y, 97_s1, 98_q1 and maybe 99_m7    

    >Solution: 
    >* For example we had been saved the year 97 entierly and the best solution for splitting is to backing up it and restoring for another slice and then deleting the out of time records
 
5. Planning a back up structure and back up all databases.
6. Storing all .Pickle and .xml files. They are all in 'output' folder in my pc. not in the external hard drive. 
7. we should develop a solution for accessing the entire flow of some entity (for example symbol historical prices from beginning up to now for detecting the deprecated symbol and date of deprecation) which are not available in one period data.   
parquet file is a good idea. It is low size and we know that not all columns of a table is needed always. for above example maybe
only symbol, isin and traded_count is enough for finding the deprecated isin.
8. Downloading .zip file from csd website. Because of saving log in database it needs to changing postgresql instances.
  

### *C1 Price*
1. some symbol not exists in tse_client symbol list, while they had records before, all of them still availabe in tsetmc.com and we have implemented some method (helper) for getting their records
    
    >Solution: 
    >1.  we could postpone getting them to when someone has their in asset 
    >2.  we could use sql except on distinct symbol between both sourceand find all of them at onecea and then get them`
