# Parliamentary Election - Power BI dashboard

## Author
- **Name:** Tadeáš Bartoš
- **GitHub:** @TadeasBartos

## Description
The goal of this project is to extend the existing [report](https://github.com/TadeasBartos/class-pbix) with map visuals and to prepare it for expansion to all levels of territorial units (counties, districts, municipalities, city parts or city boroughs) (CZ eq. - kraje, okresy, obce, městské části nebo městské obvody).

## Input Data
The input data consists of publicly available data from the website [volby.cz](https://www.volby.cz/).
Scraping this website was the subject of the following repositories:
- [one year, one territorial level unit (city boroughs)](https://github.com/TadeasBartos/class-python-web-scraper)
- [all five years, four territorial unit levels](https://github.com/TadeasBartos)

> [!CAUTION]
> Secodn repository not finished yet - future reference.

All input files had the same data structure, with ```,``` delimiter:
code, location, registered, envelopes, valid, (votes)

Example:
```
code,location,registered,envelopes,valid,OBČANÉ.CZ,Věci veřejné,Konzervativní strana,Komunistická str.Čech a Moravy,Koruna Česká (monarch.strana),Česká strana národně sociální,Česká str.sociálně demokrat.,Strana Práv Občanů ZEMANOVCI,STOP,TOP 09,EVROPSKÝ STŘED,Křesť.demokr.unie-Čs.str.lid.,Volte Pr.Blok www.cibulka.net,Strana zelených,Suverenita-blok J.Bobošíkové,Humanistická strana,Česká pirátská strana,Dělnic.str.sociální spravedl.,Strana svobodných občanů,Občanská demokratická strana,Klíčové hnutí
500054,Praha 1,24 178,16 869,16 752,37,1710,11,643,39,6,1954,370,5,5263,5,580,67,1265,198,18,120,72,157,4184,48
```

These CSV files were loaded directly into Power BI using Power Query. One import was performed for each year. These imports are then linked in a many-to-one relationship to the code-district table.

> [!NOTE]  
> This procedure will be replaced with loading all files from selected folder for universal approach in future.

The file [strany.xlsx](https://github.com/TadeasBartos/project-volby2025-pbix/blob/main/data/strany.xlsx) ensures consistent naming of political parties across different years. The year-to-year matching of party names was done manually.

### Calculated Metrics

#### % of valid votes
```
% z platných hlasů = 
DIVIDE(
    SUM(master_hlasy[POČET HLASŮ]),
    [Platné hlasy v obvodu]
)
``` 

#### % of valid votes (whole area)
```
% z platných hlasů (celé území) = 
DIVIDE(
    SUM(master_hlasy[POČET HLASŮ]),
    [Platné hlasy v celém území]
) * 100
```

#### % of valid votes in 2021
```
% z platných hlasů celkem 2021 = 
DIVIDE(
    CALCULATE(
        SUM('master_hlasy'[POČET HLASŮ]),
        REMOVEFILTERS('master_hlasy'[KÓD], 'master_hlasy'[NÁZEV OBVODU], 'master_hlasy'[ROK]),
        'master_hlasy'[ROK] = 2021
    ),
    CALCULATE(
        SUM('master_účast'[PLATNÝCH HLASŮ]),
        REMOVEFILTERS('master_účast'),
        'master_účast'[ROK] = 2021
    )
)
```

#### % change 2021 vs. 2017
```
% Změna hlasů 2021 vs. 2017 = 
VAR Hlasy2021 =
    CALCULATE(
        SUM(master_hlasy[POČET HLASŮ]),
        master_hlasy[ROK] = "2021"
    )
VAR Hlasy2017 =
    CALCULATE(
        SUM(master_hlasy[POČET HLASŮ]),
        master_hlasy[ROK] = "2017"
    )

-- Pokud strana v roce 2021 nekandidovala (žádný záznam), vrať BLANK
RETURN
    IF(
        ISBLANK(Hlasy2021),
        BLANK(),
        DIVIDE(Hlasy2021 - Hlasy2017, Hlasy2017, 0)
    )
```

#### Valid votes in area 
```
Platné hlasy v celém území = 
VAR AktRok = SELECTEDVALUE(master_hlasy[ROK])
RETURN
    CALCULATE(
        SUM('master_účast'[PLATNÝCH HLASŮ]),
        'master_účast'[ROK] = AktRok,
        REMOVEFILTERS('master_účast'[KÓD])
    )
```

#### Valid votes in city boroughs
```
Platné hlasy v obvodu = 
VAR AktKod = SELECTEDVALUE(master_hlasy[KÓD])
VAR AktRok = SELECTEDVALUE(master_hlasy[ROK])
RETURN
    CALCULATE(
        SUM('master_účast'[PLATNÝCH HLASŮ]),
        'master_účast'[KÓD] = AktKod,
        'master_účast'[ROK]= AktRok
    )
```

#### Ranking (x.)
```
Pořadí = 
VAR AktObvod = SELECTEDVALUE(master_hlasy[KÓD])
VAR AktRok   = SELECTEDVALUE(master_hlasy[ROK])

VAR TabulkaStran =
    FILTER(
        ALL('master_hlasy'[STRANA]),
        NOT ISBLANK(
            CALCULATE(
                SUM(master_hlasy[POČET HLASŮ]),
                'master_hlasy'[KÓD] = AktObvod,
                'master_hlasy'[ROK] = AktRok
            )
        )
    )

RETURN
    RANKX(
        TabulkaStran,
        CALCULATE(
            DIVIDE(
                SUM('master_hlasy'[POČET HLASŮ]),
                CALCULATE(
                    SUM('master_účast'[PLATNÝCH HLASŮ]),
                    'master_účast'[KÓD] = AktObvod,
                    'master_účast'[ROK] = AktRok
                )
            )
        ),
        ,
        DESC,
        DENSE
    )
```

#### Total ranking (x. from y.)
```
Pořadí s tečkou a celkem = 
VAR AktObvod = SELECTEDVALUE(master_hlasy[KÓD])
VAR AktRok   = SELECTEDVALUE(master_hlasy[ROK])

VAR TabulkaStran =
    FILTER(
        ALL('master_hlasy'[STRANA]),
        NOT ISBLANK(
            CALCULATE(
                SUM(master_hlasy[POČET HLASŮ]),
                'master_hlasy'[KÓD] = AktObvod,
                'master_hlasy'[ROK] = AktRok
            )
        )
    )

VAR Poradi =
    RANKX(
        TabulkaStran,
        CALCULATE(
            DIVIDE(
                SUM('master_hlasy'[POČET HLASŮ]),
                CALCULATE(
                    SUM('master_účast'[PLATNÝCH HLASŮ]),
                    'master_účast'[KÓD] = AktObvod,
                    'master_účast'[ROK] = AktRok
                )
            )
        ),
        ,
        DESC,
        DENSE
    )

VAR PocetStran =
    COUNTROWS(TabulkaStran)

RETURN
    FORMAT(Poradi, "0") & ". z " & FORMAT(PocetStran, "0")
```

#### Winning party
```
ViteznaStrana = 
VAR TabulkaStran =
    ADDCOLUMNS(
        VALUES(master_hlasy[STRANA]),
        "PocetHlasu", CALCULATE(SUM(master_hlasy[POČET HLASŮ]))
    )
VAR Vitezna =
    TOPN(
        1,
        TabulkaStran,
        [PocetHlasu], DESC,
        master_hlasy[STRANA], ASC
    )
RETURN
MAXX(Vitezna, master_hlasy[STRANA])
```

#### Winning party (respecting region and year)
```
ViteznaStranaRokObvod = 
VAR VyfiltrovanaTabulka =
    SUMMARIZE(
        master_hlasy,
        master_hlasy[ROK],
        master_hlasy[KÓD],
        master_hlasy[NÁZEV OBVODU],
        master_hlasy[STRANA],
        "CelkemHlasu", SUM(master_hlasy[POČET HLASŮ])
    )

VAR TabulkaProAktualniKontext =
    FILTER(
        VyfiltrovanaTabulka,
        master_hlasy[ROK] = SELECTEDVALUE(master_hlasy[ROK]) &&
        master_hlasy[NÁZEV OBVODU] = SELECTEDVALUE(master_hlasy[NÁZEV OBVODU])
    )

VAR ViteznaStranaTabulka =
    TOPN(
        1,
        TabulkaProAktualniKontext,
        [CelkemHlasu], DESC,
        master_hlasy[STRANA], ASC
    )

RETURN
MAXX(ViteznaStranaTabulka, master_hlasy[STRANA])
```

#### Voter turnout
```
[[Volební účast [%]]] = 
CALCULATE(
    DIVIDE(
        SUM('master_účast'[ODEVZDANÝCH HLASŮ]),
        SUM('master_účast'[REGISTROVANÝCH VOLIČŮ]),
        0
    ) * 100,
    TREATAS(
        VALUES('master_hlasy'[ROK]),
        'master_účast'[ROK]
    ),
    TREATAS(
        VALUES('master_hlasy'[KÓD]),
        'master_účast'[KÓD]
    )
)
```

## Pages

### Homepage
The introductory page of the report serves solely for basic navigation and to familiarize the user with the content.
The user can navigate through the report by selecting the arrow; each page of the report also offers a button to return to the main page and to go back or forward by one page.

![ ](https://github.com/TadeasBartos/project-volby2025-pbix/blob/main/_pictures/homepage.png)


### First page
INPUT: city borough, year, political party.

OUTPUT:
- three summary cards showing total votes cast, the winning party, and total votes received — for the entire area in the selected year;
- three detail cards showing votes received, voter turnout, and party ranking — filtered by the selected borough, year and party;
- a county-level map using a color scale from green (highest ranking) to red (lowest ranking) to visualize the party’s popularity across boroughs;
- a ranking table with percentage values to show comparative party performance;
- box plot visual highlighting the distribution of party strengths across the area.

DESCRIPTION:
For the selected year, the report links overall election results, voter turnout, and vote counts with the selected borough and party. Based on these inputs, it provides insight into the party’s strength within the chosen borough and shows its ranking across all boroughs using a map and supporting visuals.

![ ](https://github.com/TadeasBartos/project-volby2025-pbix/blob/main/_pictures/page_1.png)

### Second page
INPUT: city borough, year, political party.

OUTPUT:
- bar chart of ten best parties, respecting user selection;
- bubble chart showing relation between borough size and voter turnout;
- line chart representing voters count and voter turnout in selected borough; 
- heatmap creating using matrice of party ranking across all five years splitted into boroughs. 

DESCRIPTION:
Based on the initial selection, it provides information on the party’s ranking across districts using color shading, along with a bar chart of the 10 strongest parties in the selected year and district. This is contextualized with the ratio of valid votes to voter turnout in the district. Additionally, it includes a year-over-year trend of registered and active voters.

> [!NOTE]  
> This version of the report focused on more efficient use of dashboard space — user inputs previously displayed as tile-based slicers have been replaced with dropdowns. This enabled to fit one vizual more than in older report - [same page for comparison](https://github.com/TadeasBartos/class-pbix/blob/main/_pictures/page_2.png).

![ ](https://github.com/TadeasBartos/project-volby2025-pbix/blob/main/_pictures/page_2.png)

### Third page
INPUT: party.

OUTPUT:
- a card displaying vote count in last elections;
- a line chart illustrating the year-over-year trend in the number of voters ;
- a card with percent change between 2021 and 2017 elections; 
- a table for each year showing: district – valid votes in the district – votes received by the selected party – percentage – party’s ranking in the district

DESCRIPTION:
The third page provides users with the most comprehensive insight into the selected party’s performance across districts. Using eight tabs, users can sort the tables based on:
- number of voters in the region - reflects the size of the region;
- Number of votes – reflects the absolute number of votes received, but does not account for the size of the region (more populous regions tend to rank higher)
- percentage – reflects the party’s strength in a given borough, in smaller regions, this figure can be misleading and cause large gaps between rankings;
- ranking – shows the districts where the party ranked the highest and lowest. It supports detailed analysis of where the party performed well, poorly, or is seeing a decline in popularity.

This page can be very helpful to find the relationship between the success of a political party, the size of the region and the number of voters. This can lead to succesful campaign coordination into attractive boroughs.

In the example below, we can see that the selected party's popularity declined in large populated regions during all five elections - with a massive drop to one tenth of the vote.

![ ](https://github.com/TadeasBartos/project-volby2025-pbix/blob/main/_pictures/page_3.png)

### Fourth page
INPUT: political parties to compare. Year and city borough for detailed comparison.

OUTPUT (for each party):
- three card vizuals displaying ranking, number of votes and percentage; 
- bar chart representing overall party popularity over all elected years;
- map vizual displaying borough in unified color range (<10% red; 10% - 20% yellow; >30% green); 
- detailed analysis on the bottom with card vizuals of vote count and ranking.

DESCRIPTION:
The fourth page allows for a comparison of two selected parties in terms of overall ranking in the most recent election, percentage of votes received, and total number of votes. This is complemented by bar charts showing the vote percentages and a map with a unified color scale. In the lower section, users can perform a local comparison within a specific district and year.

![ ](https://github.com/TadeasBartos/project-volby2025-pbix/blob/main/_pictures/page_4.png)

### Fifth page
INPUT: year.

OUTPUT:
- three card vizuals displaying the most populated city borough in last elections;
- list of city boroughs to compare vote turnover, vote count and valid votes;
- map highlighting vote turnover based on selected year across all boroughs;
- bar chart comparing vote turnover over all five election years.

DESCRIPTION:
The fifth and final page provides a general overview of voter turnout across years, highlighting the significant drop in 2013 and the subsequent increase between 2017 and 2021.
The year slicer in the upper-right corner controls only the table below it. In the table, users can view the size of each district based on the number of registered voters, submitted votes, valid votes, and voter turnout.

![ ](https://github.com/TadeasBartos/project-volby2025-pbix/blob/main/_pictures/page_5.png)