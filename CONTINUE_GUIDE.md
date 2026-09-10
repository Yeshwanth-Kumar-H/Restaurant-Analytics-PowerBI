# Restaurant Power BI Dashboard — Continuation Guide

Picked up from your saved file: **`C:\Users\yeshw\Desktop\Restaurant Dashboard.pbix`** (Aug 10)
Dataset: `C:\Users\yeshw\Desktop\POWER BI\Restaurant\Restaurant_PowerBI_Project\dataset\`

**Where you are:**
- 10 CSVs imported, star-schema model with **10 active relationships + 1 inactive**
- `Date` table created (**`CALENDARAUTO`**, spans 2020-01-01 → 2020-07-01, marked as date table ✓)
- `KPI Measures` table with **6 core measures**
- Power Query cleaning is mid-way
- Report: blank "Page 1" — all work so far is in the model

**Dashboard plan (5 pages):**
1. **Landing Page** — dark background, KPI cards, nav buttons
2. **Executive Overview** — charts, trends, MoM
3. **Restaurants page** — restaurant performance
4. **Members page** — member analytics
5. **Menu page** — meal/menu analysis
6. Polish and test navigation

**Big picture (do these in order):**
1. Finish Power Query cleaning (renames + types + OrderStatus) → 2. Verify Date table → 3. Verify relationships → 4. Add the 15 remaining measures (6 → 21) → 5. Build the 5 pages

---

## 1. Power Query cleaning (per table)

Rules used: PascalCase names, explicit types, full-path `File.Contents`. Row counts to verify against after each load:

| Table | Rows | Table | Rows |
|---|---|---|---|
| orders | 36,000 | members | 200 |
| order_details | 70,577 | restaurants | 30 |
| meals | 350 | cities / meal_types / restaurant_types / serve_types | 5 / 4 / 5 / 3 |
| monthly_member_totals | 1,200 | | |

### orders (the important one)
Paste into Advanced Editor. Note the two fixes: `hour` has 7-digit fractional seconds (`11:00:00.0000000`) — trim to `HH:mm:ss` before typing it as Time; and `total_order = 0` marks orders that were never completed (verified: 0 such orders have any meal details), so we add an `OrderStatus` column.

```m
let
    Source = Csv.Document(File.Contents("C:\Users\yeshw\Desktop\POWER BI\Restaurant\Restaurant_PowerBI_Project\dataset\orders.csv"),[Delimiter=",", Columns=6, Encoding=65001, QuoteStyle=QuoteStyle.None]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Renamed Columns" = Table.RenameColumns(#"Promoted Headers",{
        {"id","OrderID"},{"date","OrderDate"},{"hour","OrderTime"},
        {"member_id","MemberID"},{"restaurant_id","RestaurantID"},{"total_order","TotalOrder"}}),
    #"Trimmed Time" = Table.TransformColumns(#"Renamed Columns",{{"OrderTime", each Text.Start(Text.From(_), 8), type text}}),
    #"Changed Type" = Table.TransformColumnTypes(#"Trimmed Time",{
        {"OrderID", Int64.Type},{"OrderDate", type date},{"OrderTime", type time},
        {"MemberID", Int64.Type},{"RestaurantID", Int64.Type},{"TotalOrder", type number}}),
    #"Added Order Status" = Table.AddColumn(#"Changed Type", "OrderStatus",
        each if [TotalOrder] > 0 then "Completed" else "Not Completed", type text),
    #"Added Hour of Day" = Table.AddColumn(#"Added Order Status", "HourOfDay",
        each Time.Hour([OrderTime]), Int64.Type)
in
    #"Added Hour of Day"
```

### order_details
```m
let
    Source = Csv.Document(File.Contents("C:\Users\yeshw\Desktop\POWER BI\Restaurant\Restaurant_PowerBI_Project\dataset\order_details.csv"),[Delimiter=",", Columns=3, Encoding=65001, QuoteStyle=QuoteStyle.None]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Renamed Columns" = Table.RenameColumns(#"Promoted Headers",{{"id","OrderDetailID"},{"order_id","OrderID"},{"meal_id","MealID"}}),
    #"Changed Type" = Table.TransformColumnTypes(#"Renamed Columns",{{"OrderDetailID", Int64.Type},{"OrderID", Int64.Type},{"MealID", Int64.Type}})
in
    #"Changed Type"
```

### meals
```m
let
    Source = Csv.Document(File.Contents("C:\Users\yeshw\Desktop\POWER BI\Restaurant\Restaurant_PowerBI_Project\dataset\meals.csv"),[Delimiter=",", Columns=7, Encoding=65001, QuoteStyle=QuoteStyle.None]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Renamed Columns" = Table.RenameColumns(#"Promoted Headers",{
        {"id","MealID"},{"restaurant_id","RestaurantID"},{"serve_type_id","ServeTypeID"},
        {"meal_type_id","MealTypeID"},{"hot_cold","HotCold"},{"meal_name","MealName"},{"price","Price"}}),
    #"Changed Type" = Table.TransformColumnTypes(#"Renamed Columns",{
        {"MealID", Int64.Type},{"RestaurantID", Int64.Type},{"ServeTypeID", Int64.Type},{"MealTypeID", Int64.Type},
        {"HotCold", type text},{"MealName", type text},{"Price", type number}})
in
    #"Changed Type"
```

### members
Rename: `id→MemberID`, `first_name→FirstName`, `surname→Surname`, `sex→Sex`, `email→Email`, `city_id→CityID`, `monthly_budget→MonthlyBudget`.
Types: MemberID/CityID Int64; FirstName/Surname/Sex/Email text; MonthlyBudget number.
Optional calculated column: `FullName = [FirstName] & " " & [Surname]`.

### restaurants
Rename: `id→RestaurantID`, `restaurant_name→RestaurantName`, `restaurant_type_id→RestaurantTypeID`, **`income_persentage→IncomePercentage` (fix the typo)**, `city_id→CityID`.
Types: RestaurantID/RestaurantTypeID/CityID Int64; RestaurantName text; IncomePercentage number (it's a decimal share, e.g. 0.075 = 7.5%).

### lookup tables
- **cities**: `id→CityID`, `city→City` (Int64, text)
- **meal_types**: `id→MealTypeID`, `meal_type→MealType` (Int64, text)
- **restaurant_types**: `id→RestaurantTypeID`, `restaurant_type→RestaurantType` (Int64, text)
- **serve_types**: `id→ServeTypeID`, `serve_type→ServeType` (Int64, text)

### monthly_member_totals (rename + types only)
Rename all to PascalCase: `member_id→MemberID, first_name→FirstName, surname→Surname, sex→Sex, email→Email, city→City, year→Year, month→Month, order_count→OrderCount, meals_count→MealsCount, monthly_budget→MonthlyBudget, total_expense→TotalExpense, balance→Balance, commission→Commission`.
Types: MemberID/Year/Month/OrderCount/MealsCount Int64; FirstName/Surname/Sex/Email/City text; MonthlyBudget/TotalExpense/Balance/Commission number.

**Keep this table OUT of the core star schema for now** — it duplicates member info and would double-count orders/revenue. It's a separate "member-month summary" fact you can use later for a loyalty/commission page (optional measures in §5).

---

## 2. Date table

Your `Date` table uses **`CALENDARAUTO`** ✓ — verify it's **marked as a date table** (Table tools → "Mark as date table" → Date column). CALENDARAUTO picks up the full range from your model (2020-01-01 → 2020-07-01), so you should be fine. If it needs rebuilding, this is a more explicit version with extra columns:

```dax
Date = 
VAR MinDate = MIN(orders[OrderDate])
VAR MaxDate = MAX(orders[OrderDate])
RETURN
ADDCOLUMNS(
    CALENDAR(MinDate, MaxDate),
    "Year", YEAR([Date]),
    "Month Number", MONTH([Date]),
    "Month Name", FORMAT([Date], "MMM"),
    "Quarter", "Q" & QUARTER([Date]),
    "Month Year", FORMAT([Date], "MMM YYYY"),
    "Weekday", FORMAT([Date], "ddd")
)
```

---

## 3. Relationships (10 active + 1 inactive — verify in Model View)

You have 10 active relationships and 1 inactive. Open Model View and check — the inactive one is likely `Date[Date]` ↔ `orders[OrderDate]` (auto-detected 1:1 which PBI disables by default, or a duplicate path). If any relationship you need is inactive, right-click → "Activate".

| From (1) | To (many) |
|---|---|
| `members[MemberID]` | `orders[MemberID]` |
| `restaurants[RestaurantID]` | `orders[RestaurantID]` |
| `restaurants[RestaurantID]` | `meals[RestaurantID]` |
| `meals[MealID]` | `order_details[MealID]` |
| `orders[OrderID]` | `order_details[OrderID]` |
| `cities[CityID]` | `members[CityID]` |
| `cities[CityID]` | `restaurants[CityID]` |
| `meal_types[MealTypeID]` | `meals[MealTypeID]` |
| `serve_types[ServeTypeID]` | `meals[ServeTypeID]` |
| `restaurant_types[RestaurantTypeID]` | `restaurants[RestaurantTypeID]` |
| `Date[Date]` | `orders[OrderDate]` ⚠️ check active |

---

## 4. The 21 measures (create in the **KPI Measures** table)

> ⚠️ Your `CALCULATED COLUMNS AND MEASURES.txt` roadmap was written for a different dataset (it references `OrderStatus`, `DeliveryTimeMins`, `CostForTwo`, `Rating`, `Votes`, a `Customer` table — none exist in your CSVs). Below is the same 21-measure structure, **adapted to your actual data**. Substitutions are marked with 🔁. You already have measures 1–5 — paste all 21 so the table is complete and self-consistent.

### 1) KPI measures (6)
```dax
Total Orders = COUNTROWS(orders)

Total Revenue = SUM(orders[TotalOrder])

Avg Order Value = DIVIDE([Total Revenue], [Total Orders])

Total Customers = DISTINCTCOUNT(orders[MemberID])

Total Restaurants = DISTINCTCOUNT(restaurants[RestaurantID])

// 🔁 replaces "Avg Delivery Time" (no delivery data exists)
Avg Meals per Order = DIVIDE([Total Meals Sold], [Total Orders])
```

### 2) Order status measures (5)
```dax
// 🔁 replaces "Delivered/Cancelled" — no status column exists.
// total_order = 0 reliably marks un-completed orders (verified: none have meal details)
Completed Orders = CALCULATE([Total Orders], orders[OrderStatus] = "Completed")

Not Completed Orders = CALCULATE([Total Orders], orders[OrderStatus] = "Not Completed")

Not Completed % = DIVIDE([Not Completed Orders], [Total Orders])

Total Meals Sold = COUNTROWS(order_details)

// 🔁 replaces the two "Late" measures (no delivery-time data exists)
Avg Meals per Completed Order = DIVIDE([Total Meals Sold], [Completed Orders])
```

### 3) Customer measures (3)
```dax
// 🔁 "Customers" = members
Repeat Customers =
COUNTROWS(
    FILTER(
        VALUES(orders[MemberID]),
        CALCULATE(COUNTROWS(orders)) > 1
    )
)

Repeat Customer % = DIVIDE([Repeat Customers], [Total Customers])

Orders Per Customer = DIVIDE([Total Orders], [Total Customers])
```

### 4) Restaurant performance (3)
```dax
Revenue per Restaurant = DIVIDE([Total Revenue], [Total Restaurants])

Meals per Restaurant = DIVIDE([Total Meals Sold], [Total Restaurants])

// 🔁 replaces "Avg Rating / Avg Votes" (not in this data). 
// IncomePercentage = the share of each order the restaurant keeps (e.g. 0.075 = 7.5%)
Restaurant Income = SUMX(orders, orders[TotalOrder] * RELATED(restaurants[IncomePercentage]))
```

### 5) Time intelligence (4)
```dax
Revenue YTD = TOTALYTD([Total Revenue], 'Date'[Date])

Orders YTD = TOTALYTD([Total Orders], 'Date'[Date])

Revenue Previous Month = CALCULATE([Total Revenue], DATEADD('Date'[Date], -1, MONTH))

Revenue Growth % = DIVIDE([Total Revenue] - [Revenue Previous Month], [Revenue Previous Month])
```

**Total: 6 + 5 + 3 + 3 + 4 = 21 ✅**

**Notes:**
- These assume the §1 renames are applied (`TotalOrder`, `MemberID`, `RestaurantID`, `OrderStatus`, `IncomePercentage`, `order_details`, etc.). Adjust table/column names if you named them differently.
- Time-intelligence measures require `Date` to be marked as a date table and related to `orders[OrderDate]`.

---

## 5. Optional (beyond the 21) — monthly_member_totals page

If you later want a "member loyalty / commission" page off the summary table:
```dax
Commission Earned = SUM(monthly_member_totals[Commission])
Total Expenses = SUM(monthly_member_totals[TotalExpense])
Avg Member Balance = AVERAGE(monthly_member_totals[Balance])
```
(Slice by `monthly_member_totals[Year]` / `[Month]`, or build a Year-Month column to join to `Date`.)

---

## 6. Dashboard pages (after measures are done)

**Page 1 — Landing Page**
Dark background, 4–6 KPI cards (Total Revenue, Total Orders, Avg Order Value, Total Customers, Total Restaurants, Completion %), nav buttons linking to the other pages.

**Page 2 — Executive Overview**
Revenue trend by Month (line chart), Revenue by Restaurant (bar), Revenue by Restaurant Type (donut), MoM Growth %, Top Meals by Revenue.

**Page 3 — Restaurants**
Restaurant table with Revenue/Meals/Income per restaurant, filterable by Restaurant Type and City, Avg Meals per Order.

**Page 4 — Members**
Repeat Customer %, Orders per Customer, Member budget vs expense, commission summary (from monthly_member_totals).

**Page 5 — Menu**
Meals by Meal Type (bar), by Serve Type (bar), Hot vs Cold (donut), Avg Price, Top meals by order count.

**Polish:** consistent colour scheme, slicers on each page (Date, City, Restaurant Type, Meal Type, Serve Type), page navigation buttons, tooltip formatting.
