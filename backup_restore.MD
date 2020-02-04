
### Backup

Reference: https://www.postgresql.org/docs/9.1/app-pgdump.html

***`pg_dump [connection-option...] [option...] [dbname]`***

#####Steps:
1. Run cmd and Go to `C:\Program Files\PostgreSQL\12\bin\`
2. Write below code and press enter: 
    
    `pg_dump.exe --file "backup file path" --host "..." --port "..." --username "..." --password --verbose --format=c "DB Name"`
    
    yor real parameter would be:
    
    `pg_dump.exe --file "E:\\Backup\\_95_y_1\\_95_y_1" --host "localhost" --port "5432" --username "postgres" --password --verbose --format=c "_95_y_1"`

#### Notes:

`--format=c`   : Creates a custom archive file that you can use with pg_restore to create a copy of a database. Custom archive format is recommended for medium to large databases as it is compressed by default.

`--section=pre-data --section=data --section=post-data`: is for selecting sub-part for backup which by default includes all. passing all item is equal to not passing them.

## Restore

reference: https://www.postgresql.org/docs/9.1/app-pgrestore.html

***`pg_restore [connection-option...] [option...] [filename]`***

#####Steps:
1. Create a database with any desired name. Of course it is recommended to naming it exactly the one you backed up
2. Run cmd: and Go to `C:\Program Files\PostgreSQL\12\bin\`
3. Write below code and press enter: 

    `pg_restore.exe --host "---" --port "---" --username "---" --password --dbname "Empty cretaed DB" --verbose "backup file path"`
    
     yor real parameter would be:
     
    `pg_restore.exe --host "localhost" --port "5432" --username "postgres" --password --dbname "_95_y_1" --verbose "E:\\Backup\\_95_y_1\\_95_y_1"`
    

#### Notes:  
 `--section=pre-data --section=data --section=post-data`: is for selecting sub-part for restoring. which by default includes all. passing all item is equal to not passing them.
