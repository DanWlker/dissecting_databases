# dissecting_databases

## General

## Postgres specific

1. [There is no (or negligable) difference between varchar and text](https://stackoverflow.com/questions/4848964/difference-between-text-and-varchar-character-varying)
    - But, if you need to index it, it may bring some [concerns](https://stackoverflow.com/a/49774665)
    - [A benchmark](https://www.depesz.com/2010/03/02/charx-vs-varcharx-vs-varchar-vs-text/)
    - [Another benchmark with recommendation](https://stackoverflow.com/a/36806014)
    - [From the postgres wiki, see tip section](https://www.postgresql.org/docs/current/datatype-character.html)

1. [Single column value hard limit is 1GB](https://stackoverflow.com/a/39966079)

1. [Maximum size for row, table, db from postgres wiki](https://wiki.postgresql.org/wiki/FAQ#What_is_the_maximum_size_for_a_row.2C_a_table.2C_and_a_database.3F) 
