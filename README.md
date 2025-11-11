# Country-aggregate-procedure
This procedure aggregates financial and quality of life data by continent from country-level details, enabling companies to quickly analyze regional economic and social conditions. It automates reporting, handles missing data gracefully, and supports data-driven strategic decisions across global operations.

# Documentation: PL/SQL Continent

## Purpose

This document outlines the steps and code components for creating a PL/SQL procedure that
generates a summary report per continent. The summary includes the total net exports, total
government spending, and average quality of life index for the countries within each continent.
The procedure demonstrates effective use of PL/SQL collections, records, and GOTO
statements for control flow and error handling.

## Database Schema Overview

The procedure works with the following tables:

● **Continents**
Columns: continent_id (NUMBER), continent_name (VARCHAR2)

● **Regions**
Columns: region_id (NUMBER), region_name (VARCHAR2), continent_id
(NUMBER) (foreign key)

● **Countries**
Columns: country_id (NUMBER), country_name (VARCHAR2), region_id
(NUMBER) (foreign key)

● **NetExports**
Columns: country_id (NUMBER), net_exports (NUMBER)

● **GovernmentSpending**
Columns: country_id (NUMBER), spending_amount (NUMBER)

● **QualityOfLife**
Columns: country_id (NUMBER), qol_index (NUMBER)

## Key PL/SQL Concepts Demonstrated

```
● Collections: Associative arrays indexed by PLS_INTEGER are used to store continent
summary records.
● Records: Custom record type captures continent name and aggregated metrics.
```

```
● GOTO Statements: Used for flow control to skip processing countries or continents if
data is missing or incomplete.
● Error Handling: Uses PL/SQL exceptions and labels to control behavior gracefully.
```
## Procedure Description

1. **Cursor Declaration:**
    Cursor to iterate over continents, ordered by continent_id.
2. **Record and Collection Types:**
    ○ ContinentSummary record holds aggregated data fields for a continent.
    ○ SummaryTable associative array (collection) holds all continent summaries
       indexed by PLS_INTEGER.
3. **Variables:**
    Used for summing net exports, government spending, quality of life, and counting
    countries processed.
4. **Data Aggregation:**
    For each continent, an inner loop fetches countries within regions linked to that
    continent.
    For each country:
       ○ Attempts to select net_exports, spending_amount, and qol_index.
       ○ Uses GOTO labels to skip countries if any data is missing.
       ○ Accumulates aggregation sums and counts valid countries.
5. **Continent Summary Storage:**
    If at least one country's data is valid, summary data is stored in the collection.
6. **Output Report:**
    After processing all continents, outputs each continent's results with a message
    indicating if average quality of life is "good" (above a defined threshold).

## Sample PL/SQL Procedure Code
```
 DECLARE
    -- Record type to store summary data per continent
    TYPE ContinentSummary IS RECORD (
        continent_name VARCHAR2(50),           -- Continent name
        total_net_exports NUMBER,              -- Sum of net exports
        total_gov_spending NUMBER,             -- Sum of government spending
        avg_qol NUMBER                         -- Average quality of life index
    );

    -- Associative array (collection) indexed by PLS_INTEGER to hold summaries
    TYPE SummaryTable IS TABLE OF ContinentSummary INDEX BY PLS_INTEGER;
    continent_summaries SummaryTable;

    -- Cursor to fetch all continents ordered by continent_id
    CURSOR cur_continents IS
        SELECT continent_id, continent_name FROM Continents ORDER BY continent_id;

    -- Variables for accumulating values across countries per continent
    v_net_exports_sum NUMBER;
    v_gov_spending_sum NUMBER;
    v_qol_sum NUMBER;
    v_country_count NUMBER;

    -- Threshold defining 'good' quality of life
    c_good_qol_threshold CONSTANT NUMBER := 80;

    -- Index for storing records in the collection
    idx PLS_INTEGER := 0;

BEGIN
    -- Loop over each continent fetched by cursor
    FOR cont_rec IN cur_continents LOOP
        -- Reset aggregators for current continent
        v_net_exports_sum := 0;
        v_gov_spending_sum := 0;
        v_qol_sum := 0;
        v_country_count := 0;

        BEGIN
            -- Loop through countries linked to the current continent (via region)
            FOR rec IN (
                SELECT c.country_id
                FROM Countries c
                JOIN Regions r ON c.region_id = r.region_id
                WHERE r.continent_id = cont_rec.continent_id
            ) LOOP
                -- Variables to hold individual country data
                DECLARE
                    v_net_exports NUMBER;
                    v_gov_spending NUMBER;
                    v_qol NUMBER;
                BEGIN
                    -- Attempt to get net exports for current country, skip if not found
                    BEGIN
                        SELECT net_exports INTO v_net_exports FROM NetExports WHERE country_id = rec.country_id;
                    EXCEPTION WHEN NO_DATA_FOUND THEN GOTO skip_country; END;

                    -- Attempt to get government spending for current country, skip if not found
                    BEGIN
                        SELECT spending_amount INTO v_gov_spending FROM GovernmentSpending WHERE country_id = rec.country_id;
                    EXCEPTION WHEN NO_DATA_FOUND THEN GOTO skip_country; END;

                    -- Attempt to get quality of life index, skip if not found
                    BEGIN
                        SELECT qol_index INTO v_qol FROM QualityOfLife WHERE country_id = rec.country_id;
                    EXCEPTION WHEN NO_DATA_FOUND THEN GOTO skip_country; END;

                    -- Accumulate country values into continent totals
                    v_net_exports_sum := v_net_exports_sum + v_net_exports;
                    v_gov_spending_sum := v_gov_spending_sum + v_gov_spending;
                    v_qol_sum := v_qol_sum + v_qol;
                    v_country_count := v_country_count + 1;

                    <<skip_country>>
                    NULL; -- Label to skip current country if needed

                END;
            END LOOP;

            -- Skip continent if no valid country data found
            IF v_country_count = 0 THEN
                GOTO skip_continent;
            END IF;

            -- Store aggregated results in collection using index
            idx := idx + 1;
            continent_summaries(idx).continent_name := cont_rec.continent_name;
            continent_summaries(idx).total_net_exports := v_net_exports_sum;
            continent_summaries(idx).total_gov_spending := v_gov_spending_sum;
            continent_summaries(idx).avg_qol := v_qol_sum / v_country_count;

        EXCEPTION
            -- Handle unexpected errors gracefully per continent
            WHEN OTHERS THEN
                DBMS_OUTPUT.PUT_LINE('Error processing continent ' || cont_rec.continent_name || ': ' || SQLERRM);
                GOTO skip_continent;
        END;

        <<skip_continent>>
        NULL; -- Label to skip continent if necessary

    END LOOP;

    -- Output all continent summaries
    FOR i IN 1 .. idx LOOP
        DBMS_OUTPUT.PUT_LINE('Continent: ' || continent_summaries(i).continent_name);
        DBMS_OUTPUT.PUT_LINE(' Total Net Exports: ' || continent_summaries(i).total_net_exports);
        DBMS_OUTPUT.PUT_LINE(' Total Government Spending: ' || continent_summaries(i).total_gov_spending);
        DBMS_OUTPUT.PUT_LINE(' Average Quality of Life: ' || ROUND(continent_summaries(i).avg_qol, 2));

        -- Qualitative assessment of quality of life
        IF continent_summaries(i).avg_qol >= c_good_qol_threshold THEN
            DBMS_OUTPUT.PUT_LINE(' Quality of Life is considered GOOD.');
        ELSE
            DBMS_OUTPUT.PUT_LINE(' Quality of Life is below the good threshold.');
        END IF;

        DBMS_OUTPUT.PUT_LINE('-----------------------------');
    END LOOP;

END;

```
## How to View Results

```
● Enable server output before running the procedure:
SET SERVEROUTPUT ON; in SQL*Plus or SQL Developer.
● Run the PL/SQL block.
● The summary will print in the output console showing per-continent aggregated values
and quality of life assessment.
```

## Notes

```
● Ensure the user running this has required privileges to SELECT from relevant tables.
● Handle missing data gracefully with GOTO for skipping countries/continents.
● Collection indexing uses PLS_INTEGER to avoid common errors.
● This is an educational procedure for demonstrating PL/SQL features; enhancement for
performance or scalability may be needed for production.
```

