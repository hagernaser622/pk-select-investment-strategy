# DAX measures

The report uses **34 measures**, all stored in a dedicated `DAX` table. They follow two ideas:

1. **Brand Review** measures are simple building blocks (revenue, rooms, guest scores) combined into ratios with `DIVIDE`.
2. **Investment Strategy** measures all react to one scenario slicer (*Low / Expected / High*) through `[Selected Scenario]`, so the same page can show three views of the plan.


## Brand Review

### Total Revenue

Total revenue of all hotels in the selected period.

```DAX
Total Revenue = SUM(MonthlyPerformance[TotalRevenue])
```

### Total Room Revenue

Revenue from rooms only (used for ADR and RevPAR).

```DAX
Total Room Revenue = SUM(MonthlyPerformance[RoomRevenue])
```

### Total Operating Expenses

Total operating expenses.

```DAX
Total Operating Expenses = SUM(MonthlyPerformance[TotalOperatingExpenses])
```

### Operating Profit

Revenue minus operating expenses.

```DAX
Operating Profit = [Total Revenue]-[Total Operating Expenses]
```

### Operating Profit Margin

Operating profit as a share of revenue. `DIVIDE` avoids divide-by-zero errors.

```DAX
Operating Profit Margin = DIVIDE([Operating Profit],[Total Revenue],BLANK())
```

### Revenue YoY %

Revenue growth against the previous year. Takes the latest year in the filter context and compares it with the year before.

```DAX
Revenue YoY % = VAR _y = MAX('Date'[Year])
VAR _prev = CALCULATE([Total Revenue], 'Date'[Year] = _y - 1)
RETURN DIVIDE([Total Revenue] - _prev, _prev)
```

### Total Sold Rooms

Number of room nights sold.

```DAX
Total Sold Rooms = SUM(MonthlyPerformance[RoomsSold])
```

### Total Available Rooms

Number of room nights available for sale.

```DAX
Total Available Rooms = SUM(MonthlyPerformance[RoomsAvailable])
```

### Occupancy Rate

Sold rooms divided by available rooms.

```DAX
Occupancy Rate = DIVIDE([Total Sold Rooms],[Total Available Rooms],BLANK())
```

### ADR

Average Daily Rate: room revenue per sold room.

```DAX
ADR = DIVIDE([Total Room Revenue],[Total Sold Rooms],BLANK())
```

### RevPAR

Revenue per Available Room: room revenue per available room. Combines price and occupancy.

```DAX
RevPAR = DIVIDE([Total Room Revenue],[Total Available Rooms],BLANK())
```

### NPS (avg)

Average Net Promoter Score from guest surveys.

```DAX
NPS (avg) = AVERAGE(GuestExperience[NetPromoterScore])
```

### Complaints per 1,000 Room Nights

Complaints normalised by sold rooms, so hotels of different sizes can be compared.

```DAX
Complaints per 1,000 Room Nights = DIVIDE(SUM(GuestComplaints[Complaints]), [Total Sold Rooms]) * 1000
```

### Room Condition Complaints

Complaints in the *Room condition* category only. Used to link complaints to renovation age.

```DAX
Room Condition Complaints = CALCULATE(
    SUM(GuestComplaints[Complaints]),
    GuestComplaints[Category] = "Room condition"
)
```


## Investment Strategy · Scenario and budget

### Selected Scenario

Reads the scenario chosen in the slicer (Low / Expected / High). Defaults to *Expected* when nothing is selected. Every investment measure below follows it.

```DAX
Selected Scenario = SELECTEDVALUE(Scenario[Scenario], "Expected")
```

### Budget

Investment budget of the P&K Select brand, looked up from the Brands table.

```DAX
Budget = LOOKUPVALUE(Brands[InvestmentBudgetUSD], Brands[Brand], "P&K Select")
```

### Funded Capex Total

Total capital expenditure of the projects marked *Funded*.

```DAX
Funded Capex Total = CALCULATE(SUM(InvestmentProjects[CapexUSD]), InvestmentProjects[Decision] = "Funded")
```

### Unallocated Budget

Budget left after funding the selected projects.

```DAX
Unallocated Budget = [Budget] - [Funded Capex Total]
```

### Budget Used %

Share of the budget that is used.

```DAX
Budget Used % = DIVIDE([Funded Capex Total], [Budget])
```


## Investment Strategy · Cash flow and NPV

### Capex

Capital expenditure, summed from the monthly project table.

```DAX
Capex = SUM(ProjectMonths[Capex])
```

### Disruption Cost

Cost of rooms or operations being out of service during a project.

```DAX
Disruption Cost = SUM(ProjectMonths[Disruption])
```

### Profit Impact

Monthly operating-profit impact of a project under the selected scenario (`SWITCH` picks the Low, High or Expected column).

```DAX
Profit Impact = SWITCH(
    [Selected Scenario],
    "Low", SUM(ProjectMonths[ProfitL]),
    "High", SUM(ProjectMonths[ProfitH]),
    SUM(ProjectMonths[ProfitE])
)
```

### Net Cash Flow

Profit impact minus capex and disruption cost.

```DAX
Net Cash Flow = [Profit Impact] - [Capex] - [Disruption Cost]
```

### NPV 2032

Net present value up to 2032: each month's net cash flow is multiplied by its discount factor and summed. Follows the selected scenario.

```DAX
NPV 2032 = SUMX(
    ProjectMonths,
    (
        SWITCH(
            [Selected Scenario],
            "Low", ProjectMonths[ProfitL],
            "High", ProjectMonths[ProfitH],
            ProjectMonths[ProfitE]
        )
        - ProjectMonths[Capex] - ProjectMonths[Disruption]
    ) * ProjectMonths[DiscFactor]
)
```

### Run-rate Profit 2032

Annual profit once the project is running: the profit of the final month (index 71) times 12.

```DAX
Run-rate Profit 2032 = CALCULATE([Profit Impact] * 12, ProjectMonths[MonthIndex] = 71)
```


## Investment Strategy · The funded plan

### Plan NPV

NPV of the funded projects only (the whole plan).

```DAX
Plan NPV = CALCULATE([NPV 2032], InvestmentProjects[Decision] = "Funded")
```

### Plan Net Cash 2027-32

Net cash flow of the funded projects over 2027–2032.

```DAX
Plan Net Cash 2027-32 = CALCULATE([Net Cash Flow], InvestmentProjects[Decision] = "Funded")
```

### Plan Run-rate Profit 2032

Run-rate profit of the funded projects in 2032.

```DAX
Plan Run-rate Profit 2032 = CALCULATE([Run-rate Profit 2032], InvestmentProjects[Decision] = "Funded")
```


## Investment Strategy · Scenarios and project impact

### Cum Net Cash Low

Cumulative net cash by year in the Low scenario. `FILTER(ALL(Year), Year <= MAX(Year))` adds up all years up to the current one.

```DAX
Cum Net Cash Low = CALCULATE(
    SUMX(ProjectMonths, ProjectMonths[ProfitL] - ProjectMonths[Capex] - ProjectMonths[Disruption]),
    FILTER(ALL(ProjectMonths[Year]), ProjectMonths[Year] <= MAX(ProjectMonths[Year]))
)
```

### Cum Net Cash Expected

Same, for the Expected scenario.

```DAX
Cum Net Cash Expected = CALCULATE(
    SUMX(ProjectMonths, ProjectMonths[ProfitE] - ProjectMonths[Capex] - ProjectMonths[Disruption]),
    FILTER(ALL(ProjectMonths[Year]), ProjectMonths[Year] <= MAX(ProjectMonths[Year]))
)
```

### Cum Net Cash High

Same, for the High scenario.

```DAX
Cum Net Cash High = CALCULATE(
    SUMX(ProjectMonths, ProjectMonths[ProfitH] - ProjectMonths[Capex] - ProjectMonths[Disruption]),
    FILTER(ALL(ProjectMonths[Year]), ProjectMonths[Year] <= MAX(ProjectMonths[Year]))
)
```

### Annual Profit Impact

Yearly operating-profit impact of a project from the scenarios table, for the selected scenario.

```DAX
Annual Profit Impact = VAR _s = [Selected Scenario]
RETURN CALCULATE(SUM(InvestmentScenarios[AnnualOperatingProfitImpactUSD]), InvestmentScenarios[Scenario] = _s)
```

### Occupancy Change

Change in occupancy (percentage points) per project and scenario. `ISINSCOPE` returns a value only at project level, so totals stay blank.

```DAX
Occupancy Change = IF(
    ISINSCOPE(InvestmentProjects[ProjectName]),
    VAR _s = [Selected Scenario]
    RETURN CALCULATE(SUM(InvestmentScenarios[OccupancyChangePts]), InvestmentScenarios[Scenario] = _s)
)
```

### Satisfaction Change

Change in guest satisfaction (points) per project and scenario. Same `ISINSCOPE` logic.

```DAX
Satisfaction Change = IF(
    ISINSCOPE(InvestmentProjects[ProjectName]),
VAR _s = [Selected Scenario]
RETURN CALCULATE(SUM(InvestmentScenarios[SatisfactionChangePts]), InvestmentScenarios[Scenario] = _s))
```
