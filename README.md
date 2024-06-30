# Florida Legislative Voting Database
6/27/24
 This repo creates a data pipeline supporting the [Jacksonville Tributary's](https://jaxtrib.org/) legislative voting dashboard (see [prior version of demo app](https://shiny.jaxtrib.org/)). The ETL scripts are adapted from an [R script originally created by apantazi](https://github.com/apantazi/legislator_dashboard/blob/main/pull-in-process-all-legiscan.R). My intent is to make it easier for others to maintain/develop the app, quickly adapt it to different jurisdictions, and create new apps from the same processed data using any programming language.
 
 Part of this work has involved reshaping nested lists (from API-acquired JSONs) into relational database format, which enables storage in Postgres as well as easy export to csv or (theoretically) SQLite. The Postgres database is currently managed locally on my Windows machine, with intent to deploy to the Tributary's Azure platform.

 See also my repo for front-end application development **[legislator dashboard](https://github.com/reliablerascal/fl-legislation-app-postgres)**.

## Overview of the database
<img src="./docs/etl-schematic.png" width=100%>

Note that the database currently integrates data from only LegiScan, supporting a single Shiny app.

|Layer|Purpose|
|---|---|
|**Raw**|Raw data retrieved as JSON via API, then parsed into tables. Data elements are retained without modification.|
|**Processed**|Cleaned and organized data including calculated fields. My intent is to also to align and integrate data between similar formats (e.g. LegiScan and LegiStar).|
|**Application**|Data prepared for specific applications such as the Jacksonville Tributary's legislator dashboard.|

ETL in the context of this legislative dashboard database means:
* **Extract** data via API from Legiscan, then parse it into tables in the raw layer.
* **Transform** data by cleaning, organizing, calculating, and aligning data so it's more useful and easier to understand
* **Load** the transformed data into the Postgres database


## Raw Layer
### Raw_LegiScan schema ###
* [Data Dictionary for raw_legiscan](docs/data-dictionary-raw-legiscan.xlsx)

This database acquires only a portion of LegiScan data (see LegiScan's [entity relationship diagram](https://api.legiscan.com/dl/Database_ERD.png) and [API user manual](https://legiscan.com/misc/LegiScan_API_User_Manual.pdf) for info on all available data). LegiScan's data is provided as three folders of JSON files- votes (which are really roll calls, with individual votes nested within), people (i.e. legislators), and bills (with lots of related info nested within).

The raw data schema of this database stores data parsed from the original JSON files but otherwise unaltered from the source format. It's organized as follows, with one row of data per unique combination of the primary key listed below.

|Table|Primary Key|Description and Notes|
|---|---|---|
|t_bills|bill_id|One record per bill. Note that bills can persist across multiple legislative sessions.|
|t_legislator_sessions|person_id, session|Because legislators can change roles (i.e. move from the House to the Senate), one record is tracked per legislator per legislative session.|
|t_roll_calls|roll_call_id|One record per roll call. Includes summary data on roll calls (e.g. how many voted aye vs. nay, etc.)|
|t_legislator_votes|person_id, roll_call_id|One record per legislator per roll call vote. Including data on how the legislator voted (aye, nay, absent, no vote).|

## Processed Layer
|Table|Primary Key|Status|Description and Notes|
|---|---|---|---|
|p_districts|district_id, district_type, year|blueprint TBD |One record per legislative district (Senate, House, City Council, etc.) summarizing Census demographics, electoral results, etc.|
|p_legislators|person_id, session|blueprint TBD |Because legislators can change roles (i.e. move from the House to the Senate), one record is tracked per legislator per legislative session.|
|p_roll_calls|roll_call_id|blueprint TBD |One record per roll call. Includes summary data on roll calls (e.g. how many voted aye vs. nay, etc.)|
|p_legislator_votes|person_id, roll_call_id|active|One record per legislator per roll call vote. Includes data on how the legislator voted (aye, nay, absent, no vote) and calculated partisan metrics (with their party, against their party, against both parties, etc.).|
<!-- The processed layer is intended to integrate data from multiple sources, organized as follows:
* Legislator data (name, party, district identifier etc.)
* Demographic data (by state/house/city council district)--->

## App Layer
This repo currently supports the legislative voting patterns Shiny app (see [prior version of demo app](https://shiny.jaxtrib.org/)).

Data is prepared to facilitate non-Shiny app development, and includes three types of fields:
* plot data (x = legislator_name, y= roll_call_id, values = partisan metric)
* context data (bill number, title, url, and description; roll call description and date, roll call vote and overall vote summary)
* filter data (party, chamber, session year)

See [Data Dictionary for app_voting_patterns](data-dictionary-app-shiny.xlsx).


<br><br>

# Guide to the Repository
Following is an overview of files in this repository:

* **data-raw**- raw data in JSON format, as bulk downloaded from LegiScan's API
* **[docs](docs/)**- data dictionaries and diagrams
* **[notebooks](notebooks/)**- API exploration using Jupyter Notebook and Python
* **[scripts](scripts/)**- ETL scripts

## Naming Conventions
Clear and consistent naming conventions are essential to code maintainability. Following are naming conventions used within this data pipeline.

|Prefix|Saved in Schema|Purpose|
|---|---|---|
|t_|raw|**T**ables of raw data kept intact in their original source format.|
|calc_|---|Performs intermediate **calc**ulations (e.g., partisanship metrics).|
|p_|proc|**P**rocessed data, which has been cleaned and organized from original tables. This includes newly-introduced calculated fields.|
|app_|app|**App**lication data, which has been filtered and organized from processed data. It's intended to support specific web applications but could also support data visualizations.|

## Running the ETL Script
The following instructions describe the process of running the ETL scripts. I hope to develop a SQLite and folder-of-csvs exports to facilitate app development for those who don't want to interact with our Postgres database.

To run these scripts, you'll need to know two passwords:
* password for the Postgres database
* API key for Legiscan

 Prior to running the ETL scripts, you'll need to set up and open a Docker container with the database. Then, from the command line you'll need to start the Docker container, open an interactive postgres terminal, and start the database fl_leg_votes.


 ```
 docker start my_postgres
 docker exec -it my_postgres bash
 psql -U postgres -d fl_leg_votes
```

 Then, you'll need to run [scripts/etl_main.R](scripts/etl_main.R), which calls the following scripts in sequence:

 | script                   | description              |
|--------------------------|--------------------------|
| [01_request_api_legiscan.R](scripts/01_request_api_legiscan.R)|requests data from LegiScan via API |
| [02_parse_legiscan.R](scripts/02_parse_legiscan.R)|parses LegiScan JSON data |
| [03_load_raw_tables.R](scripts/03_load_raw_tables.R)|saves parsed LegiScan data into Postgres as the raw layer|
| [04_transform.R](scripts/04_transform.R)|organizes parsed data and adds calculations, then prepares data for web apps |
| [05_load_views_and_app_layer.R](scripts/05_load_views_and_app_layer.R)|writes organized data frames (processed layer) and web app data frames (app layer) to Postgres |
| [00_install_packages.R](scripts/00_install_packages.R)|installation script which should later be repackaged as requirements |
| [functions_database.R](scripts/functions_database.R)|scripts to connect to Postgres, write tables, and test inputs |

<br><br>
# Ongoing Development Process
## Data Quirks and Other E-Varmints Standing in My Righteous Path
* A key focusing constraint is the short-term duration of my internship. This motivates an upfront emphasis on clear documentation and future-proofing my work.
* LegiScan data is originally tracked within a [relational database](https://api.legiscan.com/dl/Database_ERD.png), API requests are returned as [nested JSONs](https://legiscan.com/misc/LegiScan_API_User_Manual.pdf). These nested JSONs are sometimes redundant and can lead to messy/bloated data frames. Given the need for long-term scalability, my opinion is that this is best managed by restoring data to a relational database format with well-defined primary keys.
* The jury's still out for me regarding the usefulness of **Shiny** as a web application framework. I'm an old-school hand coder, so I may not find the tradeoffs in development speed vs. customizatability to be worthwhile. I'm developing the pipeline to facilitate future non-Shiny development.

## What I'm Learning
I'm continuing to re-tool from commercial database design tools (esp. Microsoft) to open source tools.
* This is my first **Postgres** database and my first **R** project, but I'm finding both very easy to pick up based on existing SQL Server and Python experience
* I'm really learning to appreciate working within an IDE with **R Studio**- particularly the ability to view entire dataframes and see a quick summary view of the number of rows and columns in each dataframe. I may likewise look for an IDE in developing future Python projects.  
* Other tools I'm picking up include **draw.io** for database diagramming and **Docker** for containerization.


## Development workplan
Following are some key goals for developing this data pipeline.
* Incorporate LegiStar voting data for Jacksonville and align this with state data, so it can be visualized with existing web apps
* Incorporate district data (e.g. census demographics and partisan leanings of the electorate) to provide context
* Develop automated CSV and SQLite exports to support non-Postgres access for data visualization and web app development
* Automate API requests via Github actions to keep legislative voting data up-to-date
* Deploy Postgres app on Azure to enable online connectivity