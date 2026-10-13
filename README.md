# task-8
SELECT
    strftime('%Y-%m', sale_date) AS month,
    SUM(quantity) AS total_quantity
FROM sales
GROUP BY strftime('%Y-%m', sale_date)
ORDER BY month;
SELECT
    strftime('%Y-%m', s.sale_date) AS month,
    SUM(s.quantity * p.price) AS total_revenue
FROM sales AS s
JOIN products AS p
    ON s.product_id = p.product_id
GROUP BY strftime('%Y-%m', s.sale_date)
ORDER BY month;
SELECT
    p.category,
    SUM(s.quantity * p.price) AS total_revenue
FROM sales AS s
JOIN products AS p
    ON s.product_id = p.product_id
GROUP BY p.category
ORDER BY total_revenue DESC;
SELECT
    s.city,
    SUM(s.quantity * p.price) AS total_revenue
FROM sales AS s
JOIN products AS p
    ON s.product_id = p.product_id
GROUP BY s.city
ORDER BY total_revenue DESC;v
WITH product_city_revenue AS (
    SELECT
        s.city,
        p.product_id,
        p.product_name,
        SUM(s.quantity * p.price) AS revenue
    FROM sales AS s
    JOIN products AS p
        ON s.product_id = p.product_id
    GROUP BY s.city, p.product_id, p.product_name
),
ranked_products AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY city
            ORDER BY revenue DESC, product_id
        ) AS rn
    FROM product_city_revenue
)
SELECT city, product_name, revenue
FROM ranked_products
WHERE rn = 1
ORDER BY revenue DESC;
SELECT
    p.category,
    AVG(s.quantity * p.price) AS average_sale_value
FROM sales AS s
JOIN products AS p
    ON s.product_id = p.product_id
GROUP BY p.category
ORDER BY average_sale_value DESC;
# generate_report.py

import sqlite3
from pathlib import Path

import pandas as pd
import matplotlib.pyplot as plt


# ==========================================
# 1. إعداد الملفات والاتصال بقاعدة البيانات
# ==========================================

BASE_DIR = Path(__file__).resolve().parent
DB_PATH = BASE_DIR / "sales.db"
EXCEL_PATH = BASE_DIR / "sales_report.xlsx"

if not DB_PATH.exists():
    raise FileNotFoundError(
        f"Database not found: {DB_PATH}\n"
        "Place sales.db next to this script."
    )

conn = sqlite3.connect(DB_PATH)


# ==========================================
# 2. تعريف الاستعلامات
# ==========================================

query1 = """
SELECT
    strftime('%Y-%m', sale_date) AS month,
    SUM(quantity) AS total_quantity
FROM sales
WHERE sale_date IS NOT NULL
GROUP BY strftime('%Y-%m', sale_date)
ORDER BY month;
"""

query2 = """
SELECT
    strftime('%Y-%m', s.sale_date) AS month,
    SUM(s.quantity * p.price) AS total_revenue
FROM sales AS s
JOIN products AS p
    ON s.product_id = p.product_id
WHERE s.sale_date IS NOT NULL
GROUP BY strftime('%Y-%m', s.sale_date)
ORDER BY month;
"""

query3 = """
SELECT
    p.category,
    SUM(s.quantity * p.price) AS total_revenue
FROM sales AS s
JOIN products AS p
    ON s.product_id = p.product_id
GROUP BY p.category
ORDER BY total_revenue DESC;
"""

query4 = """
SELECT
    s.city,
    SUM(s.quantity * p.price) AS total_revenue
FROM sales AS s
JOIN products AS p
    ON s.product_id = p.product_id
GROUP BY s.city
ORDER BY total_revenue DESC;
"""

query5 = """
WITH product_city_revenue AS (
    SELECT
        s.city,
        p.product_id,
        p.product_name,
        SUM(s.quantity * p.price) AS revenue
    FROM sales AS s
    JOIN products AS p
        ON s.product_id = p.product_id
    GROUP BY s.city, p.product_id, p.product_name
),
ranked_products AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY city
            ORDER BY revenue DESC, product_id
        ) AS rn
    FROM product_city_revenue
)
SELECT
    city,
    product_name,
    revenue
FROM ranked_products
WHERE rn = 1
ORDER BY revenue DESC;
"""

query6 = """
SELECT
    p.category,
    AVG(s.quantity * p.price) AS average_sale_value
FROM sales AS s
JOIN products AS p
    ON s.product_id = p.product_id
GROUP BY p.category
ORDER BY average_sale_value DESC;
"""

query7 = """
SELECT
    SUM(s.quantity * p.price) AS total_revenue
FROM sales AS s
JOIN products AS p
    ON s.product_id = p.product_id;
"""


# ==========================================
# 3. تنفيذ الاستعلامات وإنشاء DataFrames
# ==========================================

try:
    monthly_quantity_df = pd.read_sql_query(query1, conn)
    monthly_revenue_df = pd.read_sql_query(query2, conn)
    category_revenue_df = pd.read_sql_query(query3, conn)
    city_revenue_df = pd.read_sql_query(query4, conn)
    top_product_by_city_df = pd.read_sql_query(query5, conn)
    average_sale_value_df = pd.read_sql_query(query6, conn)
    total_revenue_df = pd.read_sql_query(query7, conn)

finally:
    conn.close()


# ==========================================
# 4. رسم الاتجاه الشهري
# ==========================================

def plot_monthly_trend(df, date_column, value_column,
                       title, y_label, file_name):
    if df.empty:
        print(f"No data available for chart: {title}")
        return

    plot_df = df.sort_values(date_column)

    plt.figure(figsize=(10, 5))
    plt.plot(
        plot_df[date_column],
        plot_df[value_column],
        marker="o",
        linewidth=2
    )

    plt.title(title)
    plt.xlabel("Month")
    plt.ylabel(y_label)
    plt.xticks(rotation=45)
    plt.grid(True, alpha=0.3)
    plt.tight_layout()
    plt.savefig(BASE_DIR / file_name, dpi=150)
    plt.show()
    plt.close()


plot_monthly_trend(
    monthly_quantity_df,
    "month",
    "total_quantity",
    "Monthly Sales Quantity",
    "Total Quantity Sold",
    "monthly_quantity.png"
)

plot_monthly_trend(
    monthly_revenue_df,
    "month",
    "total_revenue",
    "Monthly Sales Revenue",
    "Total Revenue",
    "monthly_revenue.png"
)


# ==========================================
# 5. مقارنة أعلى شهر من حيث الكمية والإيراد
# ==========================================

if not monthly_quantity_df.empty:
    highest_quantity_month = monthly_quantity_df.loc[
        monthly_quantity_df["total_quantity"].idxmax()
    ]
else:
    highest_quantity_month = None

if not monthly_revenue_df.empty:
    highest_revenue_month = monthly_revenue_df.loc[
        monthly_revenue_df["total_revenue"].idxmax()
    ]
else:
    highest_revenue_month = None


print("\n========== MONTHLY ANALYSIS ==========")

if highest_quantity_month is not None:
    print(
        "Highest quantity month:",
        highest_quantity_month["month"],
        "| Quantity:",
        highest_quantity_month["total_quantity"]
    )

if highest_revenue_month is not None:
    print(
        "Highest revenue month:",
        highest_revenue_month["month"],
        "| Revenue:",
        round(highest_revenue_month["total_revenue"], 2)
    )


# ==========================================
# 6. بناء صفحة Summary
# ==========================================

total_revenue = (
    total_revenue_df.loc[0, "total_revenue"]
    if not total_revenue_df.empty
    else None
)

summary_rows = [
    {
        "Metric": "Total Revenue",
        "Value": total_revenue
    },
    {
        "Metric": "Highest Revenue Month",
        "Value": (
            highest_revenue_month["month"]
            if highest_revenue_month is not None
            else "No data"
        )
    },
    {
        "Metric": "Highest Revenue Category",
        "Value": (
            category_revenue_df.iloc[0]["category"]
            if not category_revenue_df.empty
            else "No data"
        )
    },
    {
        "Metric": "Highest Revenue City",
        "Value": (
            city_revenue_df.iloc[0]["city"]
            if not city_revenue_df.empty
            else "No data"
        )
    },
    {
        "Metric": "Highest Quantity Month",
        "Value": (
            highest_quantity_month["month"]
            if highest_quantity_month is not None
            else "No data"
        )
    }
]

summary_df = pd.DataFrame(summary_rows)


# ==========================================
# 7. تصدير جميع النتائج إلى Excel
# ==========================================

with pd.ExcelWriter(
    EXCEL_PATH,
    engine="openpyxl"
) as writer:

    summary_df.to_excel(
        writer,
        sheet_name="Summary",
        index=False
    )

    monthly_quantity_df.to_excel(
        writer,
        sheet_name="Monthly Quantity",
        index=False
    )

    monthly_revenue_df.to_excel(
        writer,
        sheet_name="Monthly Revenue",
        index=False
    )

    category_revenue_df.to_excel(
        writer,
        sheet_name="Category Revenue",
        index=False
    )

    city_revenue_df.to_excel(
        writer,
        sheet_name="City Revenue",
        index=False
    )

    top_product_by_city_df.to_excel(
        writer,
        sheet_name="Top Product by City",
        index=False
    )

    average_sale_value_df.to_excel(
        writer,
        sheet_name="Average Sale Value",
        index=False
    )

    # تنسيق رؤوس الأعمدة وضبط عرض الأعمدة
    for worksheet in writer.sheets.values():
        worksheet.freeze_panes = "A2"
        worksheet.auto_filter.ref = worksheet.dimensions

        for cell in worksheet[1]:
            cell.font = cell.font.copy(bold=True)

        for column_cells in worksheet.columns:
            max_length = max(
                len(str(cell.value or ""))
                for cell in column_cells
            )
            column_letter = column_cells[0].column_letter
            worksheet.column_dimensions[
                column_letter
            ].width = min(max_length + 3, 35)


# ==========================================
# 8. طباعة النتائج النهائية
# ==========================================

print("\n========== CATEGORY REVENUE ==========")
print(category_revenue_df.to_string(index=False))

print("\n========== CITY REVENUE ==========")
print(city_revenue_df.to_string(index=False))

print("\n========== TOP PRODUCT BY CITY ==========")
print(top_product_by_city_df.to_string(index=False))

print("\n========== AVERAGE SALE VALUE ==========")
print(average_sale_value_df.to_string(index=False))

print("\n========== EXECUTIVE SUMMARY ==========")
print(summary_df.to_string(index=False))

print(f"\nExcel report saved to: {EXCEL_PATH}")
print("Monthly charts saved as PNG files.")
