# powerbi-dynamic-kpi-card

Interactive Power BI KPI card built with dynamic measure switching, custom period comparison, moving average logic, and reusable DAX patterns.

---

## Overview

This project showcases an interactive KPI card component developed in Power BI.

The purpose of this component is to go beyond a standard KPI visual and create a compact analytical UI element that supports dynamic user interaction and period-based performance analysis.

Users can:
- switch between different KPIs
- choose a custom time window
- compare the selected period with the previous one
- view absolute and relative KPI changes
- analyze the trend with a moving average
- interpret performance instantly through conditional formatting

This project was built as a standalone portfolio piece to demonstrate practical Power BI skills relevant for Data Analyst, BI Developer, and Analytics Engineer roles.

---

## Screenshot

![Dynamic KPI Card](KPI%20Card.jpeg)

---

## Functionalities

This KPI card combines multiple interactive features in one single component.

### 1. Dynamic KPI selection

The KPI can be changed through a disconnected selector table.

Available KPI options:
- Total Customers
- Total Orders
- Total Revenue

Changing the KPI updates:
- the main KPI callout
- the comparison label
- the previous period subtitle
- the bar chart
- the moving average line

---

### 2. Dynamic period selection

The period can be changed through a button-style slicer.

Available date window options:
- 1W
- 1M
- 3M
- TY
- 1Y
- ALL

These selections dynamically redefine the active date range used in the calculation logic.

---

### 3. Previous period comparison

The card compares the selected KPI value against the equivalent previous period.

Examples:
- 1W compares against the previous week
- 1M compares against the previous month
- 3M compares against the previous 3 months
- TY compares against the previous year period
- 1Y compares against the previous year
- ALL compares against the full previous available period logic

The result is shown as:
- absolute delta
- percentage delta

---

### 4. Conditional formatting

The comparison label uses DAX-driven conditional formatting.

Behavior:
- positive change → teal text / light teal background
- negative change → red text / light red background

This helps the viewer interpret KPI performance immediately.

---

### 5. Moving average integration

The lower combo chart displays:
- daily KPI values as columns
- a moving average as line overlay

The moving average is controlled by a second selector and can be changed dynamically.

Available moving average options:
- 3 days
- 7 days
- 14 days
- 30 days

---

### 6. Dynamic subtitle

The card includes a subtitle that explains the current reference period dynamically.

Examples:
- Previous Week: ...
- Previous Month: ...
- Previous 3 Months: ...
- Previous Year Period: ...
- Previous Year: ...

This improves readability and makes the KPI comparison more self-explanatory.

---

## Technical concept

The KPI card is based on three disconnected parameter tables:

- `Date Selection`
- `Measure Sel`
- `Moving Average`

These tables drive the interaction logic without changing the underlying data model relationships.

The DAX logic then:
1. detects the selected KPI
2. detects the selected time window
3. calculates the selected period start date
4. calculates the previous comparison period
5. calculates current KPI value
6. calculates previous KPI value
7. calculates delta and delta %
8. applies conditional formatting
9. renders the moving average for the active time range

---

## DAX logic

## 1) Date parameter table

```DAX
Date Selection =
DATATABLE(
    "Date Selection", STRING,
    "Index", INTEGER,
    {
        { "1W", 1 },
        { "1M", 2 },
        { "3M", 3 },
        { "TY", 4 },
        { "1Y", 5 },
        { "ALL", 6 }
    }
)
```
## 2) KPI selector table

```DAX
Measure Sel =
DATATABLE(
    "Measure Name", STRING,
    "ID", INTEGER,
    {
        { "1_Total Customers", 1 },
        { "1_Total Orders", 2 },
        { "1_Total Revenue", 3 }
    }
)
```
## 3) Moving average selector table

```DAX
Moving Average =
DATATABLE(
    "Days", INTEGER,
    {
        { 3 },
        { 7 },
        { 14 },
        { 30 }
    }
)
```

---

## 4) Selected date window

```DAX
3_Selected Dates =
SELECTEDVALUE('Date Selection'[Date Selection])
```

---

## 5) Selected KPI

```DAX
3_Selected Measure =
SWITCH(
    SELECTEDVALUE('Measure Sel'[Measure Name]),
    "1_Total Customers", [Total Customers],
    "1_Total Orders", [Total Orders],
    "1_Total Revenue", [Total Revenue]
)
```

---

## 6) Global min / max date

```DAX
2_MaxDate =
CALCULATE(
    MAX(dim_date[Date]),
    REMOVEFILTERS(dim_date[Date])
)
```

```DAX
2_MinDate =
CALCULATE(
    MIN(dim_date[Date]),
    REMOVEFILTERS(dim_date[Date])
)
```

---

## 7) Selected period start date

```DAX
3_Selected Min Date =
VAR MaxDate = [2_MaxDate] + 1
VAR MinDate = [2_MinDate] - 1
RETURN
    SWITCH(
        [3_Selected Dates],
        "1W", MaxDate - 7,
        "1M", EDATE(MaxDate, -1),
        "3M", EDATE(MaxDate, -3),
        "TY", DATE(YEAR(MaxDate), 1, 1),
        "1Y", EDATE(MaxDate, -12),
        "ALL", MinDate
    )
```

---

## 8) Previous comparison period start date

```DAX
3_Selected Min Date Past Period =
VAR MaxDate = [3_Selected Min Date] - 1
VAR MinDate2 = [2_MinDate] - 1
RETURN
    SWITCH(
        [3_Selected Dates],
        "1W", MaxDate - 7,
        "1M", EDATE(MaxDate, -1),
        "3M", EDATE(MaxDate, -3),
        "TY", DATE(YEAR(MaxDate), 1, 1),
        "1Y", EDATE(MaxDate, -12),
        "ALL", MinDate2
    )
```

---

## 9) KPI value for selected period

```DAX
4_SM Date Param =
CALCULATE(
    [3_Selected Measure],
    KEEPFILTERS(
        DATESBETWEEN(
            dim_date[Date],
            [3_Selected Min Date],
            [2_MaxDate]
        )
    )
)
```

---

## 10) KPI value for previous period

```DAX
4_SM Date Param Past Period =
CALCULATE(
    [3_Selected Measure],
    KEEPFILTERS(
        DATESBETWEEN(
            dim_date[Date],
            [3_Selected Min Date Past Period],
            [3_Selected Min Date]
        )
    )
)
```

---

## 11) Delta and delta percentage

```DAX
6_SM Date Param Delta =
[4_SM Date Param] - [4_SM Date Param Past Period]
```

```DAX
6_SM Date Param Delta % =
DIVIDE([6_SM Date Param Delta], [4_SM Date Param Past Period])
```

---

## 12) Moving average selection

```DAX
5_Moving Average Selection =
SELECTEDVALUE('Moving Average'[Days])
```

---

## 13) Moving average calculation

```DAX
5_SM Moving Average =
AVERAGEX(
    DATESINPERIOD(
        dim_date[Date],
        LASTDATE(dim_date[Date]),
        -[5_Moving Average Selection],
        DAY
    ),
    [3_Selected Measure]
)
```

---

## 14) Moving average filtered by selected date window

```DAX
5_SM Moving Average Date Param =
IF(
    ISBLANK([3_Selected Measure]),
    BLANK(),
    CALCULATE(
        [5_SM Moving Average],
        KEEPFILTERS(
            DATESBETWEEN(
                dim_date[Date],
                [3_Selected Min Date],
                [2_MaxDate]
            )
        )
    )
)
```

---

## 15) Dynamic reference label

```DAX
7_SM Past Period Reference Label =
VAR _Current = [4_SM Date Param]
VAR _LY = [4_SM Date Param Past Period]
VAR _Growth = DIVIDE(_Current - _LY, _LY)
VAR NumOfDigits =
    IFERROR(
        LEN(
            CONVERT(
                INT([4_SM Date Param]),
                STRING
            )
        ),
        0
    )
VAR Suffix =
    ".0" &
    SWITCH(
        TRUE(),
        NumOfDigits >= 13, "T",
        NumOfDigits >= 10, "B",
        NumOfDigits >= 7, "M",
        NumOfDigits >= 4, "K"
    )
VAR Commas =
    REPT(
        ",",
        SWITCH(
            TRUE(),
            NumOfDigits >= 13, 4,
            NumOfDigits >= 10, 3,
            NumOfDigits >= 7, 2,
            NumOfDigits >= 4, 1
        )
    )
VAR _Format =
    "#,##0" & Commas & Suffix & ";-#,##0" & Commas & Suffix
RETURN
IF(
    _Growth <> 0,
    "" & FORMAT(_Current - _LY, _Format)
        & FORMAT(_Growth, " | 0.0% ▲; | 0.0% ▼;")
        & "",
    ""
)
```

---

## 16) Dynamic subtitle for previous period

```DAX
7_SM Past Period subtitle =
VAR _LY = [4_SM Date Param Past Period]
VAR NumOfDigits =
    IFERROR(
        LEN(
            CONVERT(
                INT(_LY),
                STRING
            )
        ),
        0
    )
VAR Suffix =
    ".00" &
    SWITCH(
        TRUE(),
        NumOfDigits >= 13, "T",
        NumOfDigits >= 10, "B",
        NumOfDigits >= 7, "M",
        NumOfDigits >= 4, "K"
    )
VAR Commas =
    REPT(
        ",",
        SWITCH(
            TRUE(),
            NumOfDigits >= 13, 4,
            NumOfDigits >= 10, 3,
            NumOfDigits >= 7, 2,
            NumOfDigits >= 4, 1
        )
    )
VAR _Format =
    "#,##0" & Commas & Suffix & ";-#,##0" & Commas & Suffix
VAR _Time =
    SWITCH(
        [3_Selected Dates],
        "1W", "Previous Week: ",
        "1M", "Previous Month: ",
        "3M", "Previous 3 Months: ",
        "TY", "Previous Year Period: ",
        "1Y", "Previous Year: "
    )
RETURN
    _Time & FORMAT(_LY, _Format)
```

---

## 17) Conditional formatting measures

```DAX
8_Reference Label Background Color =
VAR x = [6_SM Date Param Delta]
RETURN
    SWITCH(
        TRUE(),
        x > 0, "#E7F3F5",
        x < 0, "#F6EDED"
    )
```

```DAX
8_Reference Label Font Color =
VAR x = [6_SM Date Param Delta]
RETURN
    SWITCH(
        TRUE(),
        x > 0, "#038D96",
        x < 0, "#A84545"
    )
```

---

## 18) Selected KPI text

```DAX
9_Selected measure Text =
SELECTEDVALUE('Measure Sel'[Measure Name])
```

---

## 19) Dynamic Y-axis max

```DAX
9_Y axis MAX =
CALCULATE(
    MAXX(VALUES(dim_date[Date]), [3_Selected Measure]),
    KEEPFILTERS(
        DATESBETWEEN(
            dim_date[Date],
            [3_Selected Min Date],
            [2_MaxDate]
        )
    )
)
```
