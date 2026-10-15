# Netflix Dashboard: Power BI Build Kit
Stack: Power BI Desktop, Power Query (M), DAX
Dataset: Kaggle "Netflix Movies and TV Shows" (`netflix_titles.csv`, columns: show_id, type, title, director, cast, country, date_added, release_year, rating, duration, listed_in, description)

---
## 1. Power Query (ETL)

Load the CSV once, then **Reference** it (not Duplicate) to create each table below. Rename the base query `Raw_Netflix` and disable its load.

### 1.1 `Raw_Netflix` (staging, load disabled)
```m
let
    Source = Csv.Document(File.Contents("C:\Data\netflix_titles.csv"),
        [Delimiter=",", Columns=12, Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    Promoted = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    Typed = Table.TransformColumnTypes(Promoted, {
        {"show_id", type text}, {"type", type text}, {"title", type text},
        {"director", type text}, {"cast", type text}, {"country", type text},
        {"date_added", type text}, {"release_year", Int64.Type},
        {"rating", type text}, {"duration", type text},
        {"listed_in", type text}, {"description", type text}}),
    Trimmed = Table.TransformColumns(Typed,
        {{"date_added", Text.Trim, type text}, {"country", Text.Trim, type text}})
in
    Trimmed
```

### 1.2 `Titles` (fact/dimension hybrid, one row per title)
```m
let
    Source = Raw_Netflix,
    // Fix known data-quality issues: ratings that leaked into duration
    FixRating = Table.AddColumn(Source, "rating_clean", each
        if [rating] <> null and Text.Contains([rating], "min") then "Unrated"
        else if [rating] = null or [rating] = "NR" or [rating] = "UR" then "Unrated"
        else [rating], type text),
    // Parse date_added ("September 25, 2021")
    DateAdded = Table.AddColumn(FixRating, "DateAdded", each
        try Date.FromText([date_added], [Culture="en-US"]) otherwise null, type nullable date),
    // Duration split: minutes for Movies, seasons for TV Shows
    Minutes = Table.AddColumn(DateAdded, "DurationMin", each
        if [type] = "Movie" and [duration] <> null
        then try Number.FromText(Text.BeforeDelimiter([duration], " ")) otherwise null
        else null, Int64.Type),
    Seasons = Table.AddColumn(Minutes, "Seasons", each
        if [type] = "TV Show" and [duration] <> null
        then try Number.FromText(Text.BeforeDelimiter([duration], " ")) otherwise null
        else null, Int64.Type),
    // Missing values
    NoDirector = Table.ReplaceValue(Seasons, null, "Unknown", Replacer.ReplaceValue, {"director"}),
    PrimaryCountry = Table.AddColumn(NoDirector, "PrimaryCountry", each
        if [country] = null or [country] = "" then "Unknown"
        else Text.Trim(Text.BeforeDelimiter([country] & ",", ",")), type text),
    // Audience group
    Audience = Table.AddColumn(PrimaryCountry, "Audience", each
        if List.Contains({"TV-MA","R","NC-17"}, [rating_clean]) then "Adults"
        else if List.Contains({"TV-14","PG-13"}, [rating_clean]) then "Teens"
        else if List.Contains({"TV-PG","PG"}, [rating_clean]) then "Older Kids"
        else if List.Contains({"TV-Y7","TV-Y7-FV","TV-G","G","TV-Y"}, [rating_clean]) then "Kids"
        else "Unrated", type text),
    Keep = Table.SelectColumns(Audience,
        {"show_id","type","title","director","release_year","rating_clean",
         "DateAdded","DurationMin","Seasons","PrimaryCountry","Audience"}),
    Renamed = Table.RenameColumns(Keep, {{"type","ContentType"},{"rating_clean","Rating"}}),
    Dedup = Table.Distinct(Renamed, {"show_id"})
in
    Dedup
```

### 1.3 `Bridge_Country` (one row per title-country, many-to-many fix)
```m
let
    Source = Table.SelectColumns(Raw_Netflix, {"show_id","country"}),
    NoNulls = Table.SelectRows(Source, each [country] <> null and [country] <> ""),
    Split = Table.ExpandListColumn(
        Table.TransformColumns(NoNulls, {{"country",
            each List.Transform(Text.Split(_, ","), Text.Trim)}}), "country"),
    Clean = Table.SelectRows(Split, each [country] <> ""),
    Renamed = Table.RenameColumns(Clean, {{"country","Country"}}),
    Dedup = Table.Distinct(Renamed)
in
    Dedup
```

### 1.4 `Bridge_Genre`
```m
let
    Source = Table.SelectColumns(Raw_Netflix, {"show_id","listed_in"}),
    Split = Table.ExpandListColumn(
        Table.TransformColumns(Source, {{"listed_in",
            each List.Transform(Text.Split(_, ","), Text.Trim)}}), "listed_in"),
    Renamed = Table.RenameColumns(Split, {{"listed_in","Genre"}}),
    Dedup = Table.Distinct(Renamed)
in
    Dedup
```

### 1.5 `Dim_Country` (for map; add Latitude/Longitude via a small lookup or let Bing geocode)
```m
let
    Source = Table.Distinct(Table.SelectColumns(Bridge_Country, {"Country"})),
    Sorted = Table.Sort(Source, {{"Country", Order.Ascending}})
in
    Sorted
```
Then in the model set **Data category = Country/Region** on `Country`.

### 1.6 `Dim_Genre`
```m
let
    Source = Table.Distinct(Table.SelectColumns(Bridge_Genre, {"Genre"}))
in
    Source
```

### 1.7 `Dim_Date` (use DAX instead, see below) 

---
## 2. Data Model (star/snowflake)

| Relationship | Cardinality | Filter direction |
|---|---|---|
| `Dim_Date[Date]` → `Titles[DateAdded]` | 1 : * | Single |
| `Titles[show_id]` → `Bridge_Country[show_id]` | 1 : * | Single |
| `Dim_Country[Country]` → `Bridge_Country[Country]` | 1 : * | Single |
| `Titles[show_id]` → `Bridge_Genre[show_id]` | 1 : * | Single |
| `Dim_Genre[Genre]` → `Bridge_Genre[Genre]` | 1 : * | Single |

Tip: to slice titles by Country/Genre, set the Titles→Bridge relationships to **Both** directions, or use `CROSSFILTER` in measures (preferred to avoid ambiguity).
Mark `Dim_Date` as the date table.

### Dim_Date (calculated table)
```dax
Dim_Date =
ADDCOLUMNS (
    CALENDAR ( DATE ( 2008, 1, 1 ), DATE ( 2022, 12, 31 ) ),
    "Year", YEAR ( [Date] ),
    "Month No", MONTH ( [Date] ),
    "Month", FORMAT ( [Date], "MMM" ),
    "Quarter", "Q" & QUARTER ( [Date] ),
    "YearMonth", FORMAT ( [Date], "YYYY-MM" )
)
```

---
## 3. DAX Measures (create a `_Measures` table)

```dax
Total Titles = DISTINCTCOUNT ( Titles[show_id] )

Movies = CALCULATE ( [Total Titles], Titles[ContentType] = "Movie" )

TV Shows = CALCULATE ( [Total Titles], Titles[ContentType] = "TV Show" )

Movie % = DIVIDE ( [Movies], [Total Titles] )

TV Show % = DIVIDE ( [TV Shows], [Total Titles] )

Countries Covered =
CALCULATE (
    DISTINCTCOUNT ( Bridge_Country[Country] ),
    CROSSFILTER ( Titles[show_id], Bridge_Country[show_id], BOTH )
)

Genres Covered =
CALCULATE (
    DISTINCTCOUNT ( Bridge_Genre[Genre] ),
    CROSSFILTER ( Titles[show_id], Bridge_Genre[show_id], BOTH )
)

Titles by Country =
CALCULATE (
    [Total Titles],
    CROSSFILTER ( Bridge_Country[show_id], Titles[show_id], BOTH )
)

Titles by Genre =
CALCULATE (
    [Total Titles],
    CROSSFILTER ( Bridge_Genre[show_id], Titles[show_id], BOTH )
)

Avg Movie Duration (min) =
CALCULATE ( AVERAGE ( Titles[DurationMin] ), Titles[ContentType] = "Movie" )

Avg Seasons per Show =
CALCULATE ( AVERAGE ( Titles[Seasons] ), Titles[ContentType] = "TV Show" )

Titles Added LY =
CALCULATE ( [Total Titles], SAMEPERIODLASTYEAR ( Dim_Date[Date] ) )

YoY Growth % = DIVIDE ( [Total Titles] - [Titles Added LY], [Titles Added LY] )

Cumulative Titles =
CALCULATE ( [Total Titles], FILTER ( ALL ( Dim_Date ), Dim_Date[Date] <= MAX ( Dim_Date[Date] ) ) )

Adult Content % =
DIVIDE ( CALCULATE ( [Total Titles], Titles[Audience] = "Adults" ), [Total Titles] )

Avg Release-to-Add Lag (yrs) =
AVERAGEX ( Titles, YEAR ( Titles[DateAdded] ) - Titles[release_year] )

Country Rank = RANKX ( ALL ( Dim_Country[Country] ), [Titles by Country], , DESC, DENSE )

Top Country =
VAR t = TOPN ( 1, ALL ( Dim_Country[Country] ), [Titles by Country], DESC )
RETURN MAXX ( t, Dim_Country[Country] )
```

---
## 4. Report Layout (16:9, dark Netflix theme)

Theme colors: `#E50914` (red), `#221F1F` (card), `#141414` (bg), `#F5F5F1` (text), `#B20710` (dark red).

**Page 1 Overview**
- Top row, 5 KPI cards: Total Titles, Movies, TV Shows, Countries Covered, Avg Movie Duration
- Left: **Filled map**, Location = `Dim_Country[Country]`, Color saturation = `[Titles by Country]`
- Middle: **Donut**, Legend = `Titles[ContentType]`, Values = `[Total Titles]`
- Right: **Clustered bar**, Top 10 Genres, Axis = `Dim_Genre[Genre]`, Values = `[Titles by Genre]` (Top N filter = 10)
- Bottom: **Line/area**, Axis = `Dim_Date[Year]`, Values = `[Total Titles]`, Legend = ContentType
- Slicers: Year, Content Type, Rating/Audience

**Page 2 Content Trends**
- Cumulative Titles line, YoY Growth % column chart, Rating bar chart, Avg release-to-add lag card, Top 10 countries bar chart, Decomposition tree (Total Titles by Country > Genre > Rating)

Add **drill-through** from Country to a Title list table (Title, Type, Rating, Release Year, DateAdded).

---
## 5. Resume bullets (ready to paste)
**Netflix Dashboard | Power BI, Power Query, DAX**
- Cleaned and transformed ~8.8K Netflix titles with Power Query (ETL): parsed dates, fixed rating/duration leaks, split multi-value country and genre fields into bridge tables.
- Designed a relational data model (date, country, genre dimensions with many-to-many bridges) and wrote 15+ DAX measures (YoY growth, cumulative totals, ranking, CROSSFILTER-based counts).
- Built an interactive dashboard with KPI cards, filled map, bar and donut charts, and trend lines to analyze content distribution by region, genre, rating and year added.
