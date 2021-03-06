# TradeDataVis

## Contents

1. Introduction 
2. Live R Scripts
3. Complexities
4. ToDo


## Introduction

Welcome to my TradeDataVis project. This project is an online tool developed in Shiny and R, running off a PostgreSQL database, which visualises the UK's import/export of food products to non-EU and EU countries. The underlying data can be found by visiting the [UKTradeInfo](http://uktradeinfo.com) website.


## Live R Scripts

### Data Cleaning and Setup

#### DownloadData.R
This is an R Script which automatically downloads and unzips trade data from the UKTradeInfo website.

#### Importers.R?
This code creates a dataframe containing correct codes and cleaned importers/exporters data. This is old and needs to be looked at and incorporated into `InitialiseDB.R` and `DownloadData.R`.


### Data Loading

#### InitialiseDB.R
A PostgreSQL database must be defined prior to running this script. It loads the unzipped files from the working directory used in `DownloadData.R` into a PostgreSQL database defined in the script. Note that the tables need not be defined before the running of this script, the postgres database just needs to be defined and the correct details entered. Four tables are entered: `control` for SMKA Commodity Codes; `dispatches` and `arrivals` for EU exports/imports, and `imports` and `exports` for non EU trade.

#### ComcodeTableBuild.R
SMKA control files are used to load Commodity Codes into the database for use in the app. However, it is extremely flawed as a data source. Some of the problems are listed under the `Complexities/Commodity Codes` section below. However, the most crucial problem of these is that there are **no consolidated 2/4/6 character commodity codes in the SMKA control files**. So we don't have names and/or structure for any of these. 

It's obvious that you would want to view the data at summary levels - consolidations of all live horse and donkey related animals for example. This is denoted by a consolidated level with a 4 character commodity code `0101` - description *Horses, Asses, Mules and Hinnies*. This means SMKA control files are incomplete - because we need to be able to build a heirarchy (to control flow of numbers up to consolidated levels) and names for the consolidated levels. 

Fortunately, Eurostat has a yearly-updated Commodity Nomenclature [here](http://ec.europa.eu/eurostat/ramon/nomenclatures/index.cfm?TargetUrl=LST_CLS_DLD&StrNom=CN_2017&StrLanguageCode=EN&IntCurrentPage=1&StrLayoutCode=LINEAR#). This contains Codes (at Section level denoted by roman numerals I-XXI, and numeric CN codes at 2-4-6-8 char levels). It contains a Parents column, which contains the section the CN Code denoted by the row rolls up to. This is a *ragged* hierarchy, which means that not all 8-char CN codes will roll up to a 6-char CN code, and so on; some 8-char CN codes will roll up to a 4-char or even 2-char level. So we need a recursive algorithm to link child to parent elements in the hierarchy - you can't simply chop off the last two characters! Lastly it has a list of descriptions for the commodity codes.

This isn't a complete cure to our problem - this Commodity Nomenclature file does *not* include older commodity codes that are no longer in use for 2017 data. However, we have data going back to 2009 - so it's important to have a full list of CN codes stretching back to that time. The method of doing this is detailed in the `Complexities/Commodity Codes` section below. By applying data cleaning routines to the Eurostat file, extracting the full list of codes and descriptions from both SMKA and Eurostat sources, and using a recursive algorithm to determine the parents of the codes, we are able to get a clean, useful, hierarchized table `comcode` with every commodity code in use since 2009 along with their latest defined descriptions for use with our app. 


## Complexities

### Commodity Codes
Commodity Codes Control Files (SMKA_) contain some serious complexities. They are listed below in bulletted form. 
* Commodity Codes are obviously primary keys - you can't have the same commodity code contain completely different types of data! The way this is handled is that the commodity code is _added_ if it does not exist within the table. If it _already exists_, that entry is updated with the information in the current SMKA file. This method of adding/updating is referred to as `UPSERT` (portmanteau of update-insert). This has to be done using line-by-line SQL queries, as R's `DBI` package does not support UPSERT operations. As we consider SMKA files sequentially from 200901, we always have the most up-to-date description for each commodity code.
* SMKA files prior to 201201 have the SUB unicode character in one of the commodity code descriptions. All data analysis tools use this character as the EOF marker - stopping the dataload! This is an outstanding issue.
* Older pre-2012 SMKA files also split the description up with a | delimiter after it reaches a certain character limit for god knows what reason. SQL table limits pre-2012 maybe? I don't know. I do know that it's annoying to deal with. There's some lines which merge the final two columns in the data frame if they exceed the number of columns in the new data format to homogenise the data structures so everything can be loaded into the same table.
* Lastly, since the descriptions contain both " and ' chars, quoting is set to null for the `read.table` load. Apostrophes are all converted to double apostrophes `''` during the data cleaning routine, as SQL statements rely on the ' char for denoting strings!


## ToDo
* Adapt `DownloadData.R` and `InitialiseDB.R` to include importers/exporters data.
* Solve error for old SMKA including the sub character for some reason.
* Develop shiny app.
* Research tools to run on could