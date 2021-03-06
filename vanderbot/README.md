# VanderBot

The short link to this page is [vanderbi.lt/vanderbot](http://vanderbi.lt/vanderbot)

## Description

This project is currently focused on author disambiguation and association with identifiers in Wikidata.  The code associated with this work is referred to as "VanderBot" and it does the work for the [Wikidata VanderBot bot](https://www.wikidata.org/wiki/User:VanderBot), a non-autonomous bot.  

As of 2020-04-23, VanderBot has created or curated records for 4436 scholars or researchers at Vanderbilt and made 4998 record updates. The number includes nearly all researchers in Vanderbilt colleges and schools except the School of Medicine. Records for nearly all of the faculty in the School of medicine have been curated, but research staff and postdocs have not yet been done.

Here are some queries that can be run to explore the data:

[Count the total number of unique affiliates of Vanderbilt in Wikidata](https://w.wiki/NpE)

[List units with the total number of affiliates for each](https://w.wiki/NpF)

[Overall gender balance of scholars and researchers at Vanderbilt (those in Wikidata with known sex/gender)](https://w.wiki/Nph)

[Examine gender balance by unit](https://w.wiki/NpG)

[Count the number of works linked to authors by department](https://w.wiki/Nqm)

[Calculate the fraction of works linked to authors by gender](https://w.wiki/Nqr)

## How it works

This [video shows VanderBot in operation](https://youtu.be/4zi9wj7EwRU)



## Release v1.0 (2020-04-20) notes 

There are a number of scripts involved that are run sequentially:

- **vb1_process_department.ipynb** - A Jupyter notebook containing a variety of Python scripts used to scrape names of "employees" (researchers/scholars) from departmental websites and directories. The scripts are ideosyncratic to the particular web pages but output to consistently formatted CSV files. There is also a script for downloading data from the ORCID API for all employees associated with a particular institution. Example output file: `medicine-employees.csv`
- **vb2_match_orcid.py** - A Python script that matches the scraped departmental employee names with the dowloaded ORCID records for the institution. Example output file: `medicine-employees-with-wikidata.csv` (has additional data and `gender` column added by the next script)
- **vb3_match_wikidata.py** - A Python script that uses a variety of methods (including matching ORCIDs and fuzzy string matching) to match the employees with Wikidata Q IDs. The script queries the Wikidata SPARQL endpoint and also retrieves data from PubMed, ORCID, and Crossref when necessary to assist in the disambiguation. An optional followup step is to add the sex/gender of employees manually. Example output file: `medicine-employees-with-wikidata.csv`
- **vb4_download_wikidata.py** - A Python script that downloads existing data from Wikidata using SPARQL for employees that were matched with their Q IDs. New data are generated based on rules or known information. The script also tests the validity of all ORCID IRIs by dereferencing them. A followup step is to manually clean up descriptions and names in the output CSV. Example output file: `medicine-employees-to-write.csv` (unaffected by next script)
- **vb5_check_labels_descriptions.py** - A Python script that performs SPARQL queries to check for conflicts with label/description combinations that already exist in Wikidata (creating new records with the same label/description is not allowed by the API). It also sets the CSV file name in the `csv-metadata.json` mapping file used by the following script.
- **vb6_upload_wikidata.py** - A Python script that reads column headers from a CSV file and maps them to the Wikidata data model using the schema in the `csv-metadata.json` mapping file. The script then reads each record from the file and writes the item data to the Wikidata API. The API response is recorded in the table as a record that the write was successfully completed. A followup loop adds references to existing statements that didn't already have them. The file `medicine-employees-to-write.csv` contains identifiers (Item Q IDs, statement UUIDs, and reference hashes) added from data returned by the API.

The file **vb_common_code.py** is a module that contains functions that are used across the scripts above.

The CSV file that feeds data from the fifth script to the sixth uses a mapping from the CSV headers to Wikidata properties that is specified using the [W3C Generating RDF from Tabular Data on the Web](http://www.w3.org/TR/csv2rdf/) Recommendation.  The JSON mapping file is [here](https://github.com/HeardLibrary/linked-data/blob/master/vanderbot/csv-metadata.json). 

For details about the design and operation of VanderBot, see [this series of blog posts](http://baskauf.blogspot.com/2020/02/vanderbot-python-script-for-writing-to.html).



## Query() class (defined in `vb_common_code.py`)

Methods of the general-purpose `Query()` class sends queries to Wikibase instances. It has the following methods:

`.generic_query(query)` Sends a specified query to the endpoint and returns a list of item Q IDs, item labels, or literal values. The variable to be returned must be `?entity`.

`.single_property_values_for_item(qid)` Sends a subject Q ID to the endpoint and returns a list of item Q IDs, item labels, or literal values that are values of a specified property.

`.labels_descriptions(qids)` Sends a list of subject Q IDs to the endpoint and returns a list of dictionaries of the form `{'qid': qnumber, 'string': string}` where `string` is either a label, description, or alias. Alternatively, an added graph pattern can be passed as `labelscreen` in lieu of the list of Q IDs. In that case, pass an empty list (`[]`) into the method. The screening graph pattern should have `?id` as its only unknown variable.

`.search_statement(qids, reference_property_list)` Sends a list of Q IDs and a list of reference properties to the endpoint and returns information about statements using a property specified as the pid value. If no value is specified, the information includes the values of the statements. For each statement, the reference UUID, reference property, and reference value is returned. If the statement has more than one reference, there will be multiple results per subject. Results are in the form `{'qId': qnumber, 'statementUuid': statement_uuid, 'statementValue': statement_value, 'referenceHash': reference_hash, 'referenceValue': reference_value}`

It has the following attributes:

| key | description | default value | applicable method |
|:-----|:-----|:-----|:-----|
| `endpoint` | endpoint URL of Wikabase | `https://query.wikidata.org/sparql` | all |
| `mediatype` | Internet media type | `application/json` | all |
| `useragent` | User-Agent string to send | `VanderBot/0.9` etc.| all |
| `requestheader` | request headers to send |(generated dict) | all |
| `sleep` | seconds to delay between queries | 0.25 | all |
| `isitem` | `True` if value is item, `False` if value a literal | `True` | `generic_query`, `single_property_values_for_item` |
| `uselabel` | `True` for label of item value , `False` for Q ID of item value | `True` | `generic_query`, `single_property_values_for_item` | 
| `lang` | language of label | `en` | `single_property_values_for_item`, `labels_descriptions`|
| `labeltype` | returns `label`, `description`, or `alias` | `label` | `labels_descriptions` |
| `labelscreen` | added triple pattern | empty string | `labels_descriptions` |
| `pid` | property P ID | `P31` | `single_property_values_for_item`, `search_statement` |
| `vid` | value Q ID | empty string | `search_statement` |


## Employee matching to Wikidata in `vb3_match_wikidata.py`

The script `vb3_match_wikidata.py` attempts to match records of people that Wikidata knows to work at Vanderbilt with departmental employees by matching their ORCIDs, then name strings. If there isn't a match with the downloaded Wikidata records, for employees with ORCIDs the script attempts to find them in Wikidata by directly doing a SPARQL search for their ORCID.

As people are matched (or determined to not have a match), a code is recorded with information about how the match was made.  Here are the values:

```
0=unmatched
1=matched with ORCID in both sources
2=ORCID from match to ORCID records but name match to Wikidata (no ORCID)
3=no ORCID from match to ORCID records but name match to Wikidata (with ORCID); could happen if affiliation isn't matched in ORCID
4=no ORCID from match to ORCID records but name match to Wikidata (no ORCID)
5=ORCID from match to ORCID records and found via SPARQL ORCID search (likely non-VU affiliated in Wikidata)
6=ORCID from match to ORCID records and found via SPARQL name search (non-VU affiliated without ORCID)
7=no name match
8=ORCID from match to ORCID records, error in SPARQL ORCID search
9=no ORCID from match to ORCID records, error in SPARQL name search
10=affiliation match in article
11=match by human choice after looking at entity data
12=no matching entities were possible matches
13=match pre-existing Wikidata entry from another department
```

## Downloading existing statements and references from Wikidata in `vb4_download_wikidata.py`

### Generation of statement values

There are two categories of statements whose existing data are retrieved from Wikidata.

1\. One type are statements that **must have a specific value**. For example, if we want to state that the employer (P108) is Vanderbilt University (Q29052), we do not care whether the employee already has an employer statement with some value other than Q29052 -- we only care if our particular property and value have already been asserted by someone or not. If the statement has already been asserted, we don't need to make it. If it has not been asserted, we will add it. In this case, there will never be missing values. These items have a `discovery_allowed` value of `False`. 

2\. The other type of statements **can have varying values** depending on the individual employee. We may know the value by some means independent of Wikidata or we may not have any value for that property and be interested in discovering it from Wikidata. These items have a `discovery_allowed` value of `True`. Examples would include ORCID (P496) and sex or gender (P21).

If a value is not known, this script will record it if it has been discovered in Wikidata. Since the data storage system used by the script is flat (a spreadsheet with a single row per item), only the first discovered value will be recorded. However, the script will issue a warning if there are additional values that are found for the item after the first one. 

If discovery is allowed and the known value agrees with the Wikidata value, nothing will happen other than the recording of any relevant reference information. However, if a discovered value is different than the previously known value, the script will issue a warning.

### Assumptions about references

The generic `.search_statements()` method used in this script to retrieve data about statements does not assume any particular kind or order of reference properties. However, this script assumes that there are zero to two reference properties. The possibilities are:

- no reference properties (length of `refProps` list = 0)
- one reference property that is a retrieved date (length of `refProps` list = 1)
- two reference properies that are the reference URL and a retrieved date (length of `refProps` list = 2)

In the case where there are no reference properties, there also isn't any reference hash being tracked (i.e. there isn't any reference at all).

If there are reference property combinations other than this, the `generate_statement_data()` function can't be used and custom code must be written for that statement.


----
Revised 2020-04-23
