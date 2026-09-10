# Data Cleaning — Manual Power Query Walkthrough

Work through this at your own pace in Power Query Editor (Transform Data).
Do the small dimension tables first, then the bigger fact tables.

---

## 1–4. Lookup tables (cities, meal_types, restaurant_types, serve_types)

All four are tiny (2–5 rows). For each one:

| What | How | Why |
|---|---|---|
| Rename `id` | Right-click header → Rename → `CityID` / `MealTypeID` / `RestaurantTypeID` / `ServeTypeID` | `id` is ambiguous across10 tables |
| Rename the other column | `city→City`, `meal_type→MealType`, `restaurant_type→RestaurantType`, `serve_type→ServeType` | Clean names for visuals |
| Set data types | Click icon left of header → Whole Number for IDs, Text for names | Prevents type mismatches in relationships |

**Row counts:** cities=5, meal_types=4, restaurant_types=5, serve_types=3.

---

## 5. members (200 rows)

| Step | What to do | Why |
|---|---|---|
| **Rename** | `id→MemberID` | Self-documenting in relationships |
| **Rename** | `first_name→FirstName`, `surname→Surname`, `sex→Sex`, `email→Email` | Consistent PascalCase |
| **Rename** | `city_id→CityID` | Must match cities table for the relationship |
| **Rename** | `monthly_budget→MonthlyBudget` | Cleaner name for measures |
| **Set types** | MemberID/CityID = Whole Number, MonthlyBudget = Decimal Number, all text columns = Text | Type mismatches break relationships |
| **Add calculated column** | Add Column → Custom Column → name: `FullName`, formula: `[FirstName] & " " & [Surname]` | Handy for member lookup/display |

Verify:8 columns,200 rows.

---

## 6. restaurants (30 rows)

| Step | What to do | Why |
|---|---|---|
| **Rename** | `id→RestaurantID` | Standardise |
| **Rename** | `restaurant_name→RestaurantName`, `restaurant_type_id→RestaurantTypeID`, `city_id→CityID` | Must match restaurant_types / cities tables |
| **Rename** | **`income_persentage→IncomePercentage`** ⚠️ | Fix the typo so measures can reference it |
| **Set types** | RestaurantID/RestaurantTypeID/CityID = Whole Number, IncomePercentage = Decimal Number, RestaurantName = Text | IncomePercentage is a decimal share (0.075 = 7.5%) |

Verify:5 columns,30 rows.

---

## 7. meals (350 rows)

| Step | What to do | Why |
|---|---|---|
| **Rename** | `id→MealID` | Standardise |
| **Rename** | `restaurant_id→RestaurantID`, `serve_type_id→ServeTypeID`, `meal_type_id→MealTypeID` | Must match IDs in restaurants / serve_types / meal_types |
| **Rename** | `hot_cold→HotCold`, `meal_name→MealName`, `price→Price` | Clean names for visuals and measures |
| **Set types** | All IDs = Whole Number, HotCold/MealName = Text, Price = Decimal Number | Price must be Decimal for SUM/AVG |

Verify:7 columns,350 rows. HotCold = "Hot" and "Cold" only.

---

## 8. order_details (70,577 rows)

| Step | What to do | Why |
|---|---|---|
| **Rename** | `id→OrderDetailID`, `order_id→OrderID`, `meal_id→MealID` | Standardise |
| **Set types** | All three = Whole Number | Keys must match parent tables exactly |

Verify:3 columns,70,577 rows.

---

## 9. orders ⭐ (36,000 rows — the most important one)

| Step | What to do | Why |
|---|---|---|
| **Rename** | `id→OrderID`, `date→OrderDate`, `hour→OrderTime`, `member_id→MemberID`, `restaurant_id→RestaurantID`, `total_order→TotalOrder` | Standardise for relationships and measures |
| **Set types** | OrderID = Whole Number, OrderDate = **Date**, MemberID/RestaurantID = Whole Number, TotalOrder = **Decimal Number** | OrderDate must be Date (not DateTime) for time intelligence; TotalOrder must be Decimal for SUM |
| **Fix OrderTime** | ⚠️ Time column has 7-digit fractional seconds (`11:00:00.0000000`). Transform → **Extract → Text before delimiter** → delimiter `.` → replace column → set type to **Time** | Trimming to `HH:mm:ss` first makes Time parsing reliable |
| **Add OrderStatus** | Add Column → Custom Column → name: `OrderStatus`, formula: `if [TotalOrder] > 0 then "Completed" else "Not Completed"` → set type to Text | `total_order = 0` reliably means not completed (verified: 0 such rows have meal details). Powers your status measures |
| **Add HourOfDay** | Add Column → Custom Column → name: `HourOfDay`, formula: `Time.Hour([OrderTime])` → set type to Whole Number | Extracts hour as a number for time-of-day charts |

Verify:8 columns,36,000 rows. OrderStatus: "Completed" ≈30,782, "Not Completed" ≈5,218.

---

## 10. monthly_member_totals (1,200 rows)

| Step | What to do | Why |
|---|---|---|
| **Rename all** | `member_id→MemberID`, `first_name→FirstName`, `surname→Surname`, `sex→Sex`, `email→Email`, `city→City`, `year→Year`, `month→Month`, `order_count→OrderCount`, `meals_count→MealsCount`, `monthly_budget→MonthlyBudget`, `total_expense→TotalExpense`, `balance→Balance`, `commission→Commission` | PascalCase consistency |
| **Set types** | MemberID/Year/Month/OrderCount/MealsCount = Whole Number, text columns = Text, MonthlyBudget/TotalExpense/Balance/Commission = Decimal Number | This is a summary table — **don't connect it to the star schema** (double-count risk) |

Verify:14 columns,1,200 rows.

---

## After all10 tables

1. **Close & Apply** — watch for errors
2. **Row counts to verify:**

| Table | Expected | Table | Expected |
|---|---|---|---|
| orders | 36,000 | members | 200 |
| order_details | 70,577 | restaurants | 30 |
| meals | 350 | cities | 5 |
| meal_types | 4 | restaurant_types | 5 |
| serve_types | 3 | monthly_member_totals | 1,200 |

3. **Open Model View** — confirm relationships still valid. Check `Date[Date] → orders[OrderDate]` is **active** (right-click → Activate if greyed out).
4. **Next up:** Create the21 measures in `KPI Measures` table (see CONTINUE_GUIDE.md §4).
