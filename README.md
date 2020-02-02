# To Do List :octocat:

---

## *DB Utility*
* semi_index 


## *L1*

### *C1 Asset*

* Initial asset in 950301 in new version has some record more than previos version

* Initail asset has some records with zero asset, maybe we should filter them before db insertion
* Writing a script for calculating the portfolio on a date which is needed for making the initial asset of every new period. It is obvious we should run it for all databases and  add initial asset to them
* Allowed limit for each asset table is about 10 million records, which needs some slicing on period. for example 95 and 96 have less than 10 million record and year period is good for them. 97 has 14 million records which should be divided to two semi year period. in 98 each quarter has 10 million which needs to be treated as a independent period.
naming convention for database follows this slicing necessity.some example: _95_y, 96_y, 97_s1, 98_q1 and maybe 99_m7    

    >Solution: 
    >* For example we had been saved the year 97 entierly and the best solution for splitting is to backing up it and restoring for another slice and then deleting the out of time records
 
* Planning a back up structure and back up all databases.
* Storing all .Pickle and .xml files. They are all in 'output' folder in my pc. not in the external hard drive. 
* we should develop a solution for accessing the entire flow of some entity (for example symbol historical prices from beginning up to now for detecting the deprecated symbol and date of deprecation) which are not available in one period data.   
parquet file is a good idea. It is low size and we know that not all columns of a table is needed always. for above example maybe
only symbol, isin and traded_count is enough for finding the deprecated isin.
 

### *C1 Price*
* some symbol not exists in tse_client symbol list, while they had records before, all of them still availabe in tsetmc.com and we have implemented some method (helper) for getting their records
    
    >Solution: 
    >1.  we could postpone getting them to when someone has their in asset 
    >2.  we could use sql except on distinct symbol between both sourceand find all of them at onecea and then get them`
