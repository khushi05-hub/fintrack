# fintrack
python project 

import customtkinter as ctk
from tkinter import ttk, messagebox, filedialog
from datetime import datetime
import matplotlib
matplotlib.use("TkAgg")
from matplotlib.figure import Figure
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
import csv
import os
import json
import uuid

# =====================================================
# FINTRACK 4.1 - SMART EXPENSE TRACKER
# =====================================================

ctk.set_appearance_mode("dark")
ctk.set_default_color_theme("blue")

BASE = os.path.dirname(os.path.abspath(__file__))
CSV_FILE = os.path.join(BASE, "finance_data.csv")
SETTINGS_FILE = os.path.join(BASE, "settings.json")

# MIDNIGHT LAVENDER THEME

BG = "#100D1C"
SIDEBAR = "#19142A"
CARD = "#211B35"
CARD2 = "#302747"
LAVENDER = "#B69CFF"
GREEN = "#55D6BE"
RED = "#FF718A"
GOLD = "#FFC978"
WHITE = "#F7F2FF"
MUTED = "#B0A7C3"
BORDER = "#493B61"

CATEGORIES = [
    "Food", "Travel", "Shopping", "Education",
    "Entertainment", "Bills", "Healthcare",
    "Salary", "Freelancing", "Other"
]

records = []
budget = 10000.0
search_job = None


# =====================================================
# DATA STORAGE
# =====================================================

def load_data():
    global records, budget

    records = []

    if os.path.exists(CSV_FILE):
        try:
            with open(CSV_FILE, "r", newline="", encoding="utf-8") as f:
                reader = csv.DictReader(f)

                for row in reader:
                    try:
                        amount = float(row["Amount"])
                        datetime.strptime(row["Date"], "%d-%m-%Y")

                        if amount <= 0 or row["Type"] not in ["Income", "Expense"]:
                            continue

                        records.append(row)

                    except (ValueError, KeyError, TypeError):
                        continue

        except (OSError, csv.Error):
            messagebox.showerror("Error", "Unable to load transaction data.")

    if os.path.exists(SETTINGS_FILE):
        try:
            with open(SETTINGS_FILE, "r", encoding="utf-8") as f:
                budget = float(json.load(f).get("budget", 10000))

            if budget <= 0:
                budget = 10000.0

        except (OSError, ValueError, TypeError, json.JSONDecodeError):
            budget = 10000.0


def save_data():
    fields = ["ID", "Date", "Type", "Category", "Amount", "Description"]

    with open(CSV_FILE, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=fields)
        writer.writeheader()
        writer.writerows(records)


def save_settings():
    with open(SETTINGS_FILE, "w", encoding="utf-8") as f:
        json.dump({"budget": budget}, f, indent=4)


def money(value):
    return f"₹{value:,.2f}"


def get_totals(data):
    income = sum(float(r["Amount"]) for r in data if r["Type"] == "Income")
    expense = sum(float(r["Amount"]) for r in data if r["Type"] == "Expense")
    return income, expense


def monthly_records():
    current_month = datetime.now().strftime("%m-%Y")
    result = []

    for r in records:
        try:
            record_month = datetime.strptime(
                r["Date"], "%d-%m-%Y"
            ).strftime("%m-%Y")

            if record_month == current_month:
                result.append(r)

        except (ValueError, KeyError):
            continue

    return result


# =====================================================
# MAIN WINDOW
# =====================================================

app = ctk.CTk()
app.title("FINTRACK | Smart Personal Finance")
app.geometry("1350x850")
app.minsize(1100, 720)
app.configure(fg_color=BG)

load_data()


# =====================================================
# HELPER FUNCTIONS
# =====================================================

def make_label(parent, text, size=13, color=WHITE, weight="normal"):
    return ctk.CTkLabel(
        parent,
        text=text,
        font=("Segoe UI", size, weight),
        text_color=color
    )


def make_button(parent, text, command, color=LAVENDER, width=120):
    text_color = BG if color in [LAVENDER, GREEN, GOLD] else WHITE

    return ctk.CTkButton(
        parent,
        text=text,
        command=command,
        width=width,
        height=39,
        corner_radius=10,
        fg_color=color,
        hover_color="#59477A",
        text_color=text_color,
        font=("Segoe UI", 12, "bold")
    )


def make_entry(parent, placeholder, width=250):
    return ctk.CTkEntry(
        parent,
        placeholder_text=placeholder,
        width=width,
        height=40,
        corner_radius=9,
        fg_color=CARD2,
        border_color=BORDER,
        text_color=WHITE,
        placeholder_text_color=MUTED
    )


# =====================================================
# ADD / EDIT TRANSACTION
# =====================================================

def open_transaction(edit=False):

    existing = None

    if edit:
        selected = tree.selection()

        if not selected:
            messagebox.showwarning("Selection", "Select a transaction first.")
            return

        selected_id = selected[0]

        existing = next(
            (r for r in records if r["ID"] == selected_id),
            None
        )

        if not existing:
            return

    win = ctk.CTkToplevel(app)
    win.title("Transaction Details")
    win.geometry("480x680")
    win.resizable(False, False)
    win.configure(fg_color=BG)
    win.transient(app)
    win.grab_set()

    make_label(
        win,
        "EDIT TRANSACTION" if edit else "NEW TRANSACTION",
        25, LAVENDER, "bold"
    ).pack(pady=(25, 5))

    make_label(
        win, "Track every rupee with confidence.", 12, MUTED
    ).pack(pady=(0, 15))

    form = ctk.CTkFrame(
        win, fg_color=CARD, corner_radius=18,
        border_width=1, border_color=BORDER
    )
    form.pack(fill="both", expand=True, padx=25, pady=10)

    make_label(
        form, "TRANSACTION TYPE", 11, MUTED, "bold"
    ).pack(anchor="w", padx=22, pady=(18, 7))

    type_menu = ctk.CTkSegmentedButton(
        form,
        values=["Expense", "Income"],
        selected_color=LAVENDER,
        selected_hover_color="#59477A",
        unselected_color=CARD2,
        text_color=WHITE
    )
    type_menu.pack(fill="x", padx=22)
    type_menu.set(existing["Type"] if existing else "Expense")

    def create_field(title, placeholder, default=""):
        make_label(
            form, title, 12, MUTED, "bold"
        ).pack(anchor="w", padx=22, pady=(14, 5))

        entry = make_entry(form, placeholder)
        entry.pack(fill="x", padx=22)

        if default:
            entry.insert(0, default)

        return entry

    amount_entry = create_field(
        "AMOUNT (₹)", "Enter amount",
        existing["Amount"] if existing else ""
    )

    make_label(
        form, "CATEGORY", 12, MUTED, "bold"
    ).pack(anchor="w", padx=22, pady=(14, 5))

    category_menu = ctk.CTkOptionMenu(
        form,
        values=CATEGORIES,
        fg_color=CARD2,
        button_color=LAVENDER,
        height=40
    )
    category_menu.pack(fill="x", padx=22)
    category_menu.set(existing["Category"] if existing else "Food")

    date_entry = create_field(
        "DATE (DD-MM-YYYY)", "Enter date",
        existing["Date"] if existing else datetime.now().strftime("%d-%m-%Y")
    )

    description_entry = create_field(
        "DESCRIPTION", "What was this transaction for?",
        existing["Description"] if existing else ""
    )

    def save_transaction():

        try:
            amount = float(amount_entry.get().strip())

            if amount <= 0:
                raise ValueError

        except ValueError:
            messagebox.showerror(
                "Invalid Amount",
                "Enter a valid amount greater than zero.",
                parent=win
            )
            return

        try:
            datetime.strptime(date_entry.get().strip(), "%d-%m-%Y")

        except ValueError:
            messagebox.showerror(
                "Invalid Date",
                "Use DD-MM-YYYY format.",
                parent=win
            )
            return

        transaction = {
            "ID": existing["ID"] if existing else str(uuid.uuid4()),
            "Date": date_entry.get().strip(),
            "Type": type_menu.get(),
            "Category": category_menu.get(),
            "Amount": f"{amount:.2f}",
            "Description": description_entry.get().strip()
        }

        old_records = [r.copy() for r in records]

        if edit:
            for i, r in enumerate(records):
                if r["ID"] == existing["ID"]:
                    records[i] = transaction
                    break
        else:
            records.append(transaction)

        try:
            save_data()

        except OSError:
            records[:] = old_records
            messagebox.showerror(
                "Error", "Unable to save transaction.", parent=win
            )
            return

        win.destroy()
        refresh_all()

        messagebox.showinfo(
            "Success",
            "Transaction updated!" if edit else "Transaction added successfully!"
        )

    make_button(
        form,
        "UPDATE TRANSACTION ✓" if edit else "SAVE TRANSACTION ✓",
        save_transaction,
        GREEN,
        230
    ).pack(pady=27)


# =====================================================
# DELETE TRANSACTION
# =====================================================

def delete_transaction():

    selected = tree.selection()

    if not selected:
        messagebox.showwarning("Selection", "Select a transaction first.")
        return

    if not messagebox.askyesno(
        "Confirm Delete",
        "Are you sure you want to delete selected transactions?"
    ):
        return

    old_records = [r.copy() for r in records]
    ids = set(selected)

    records[:] = [r for r in records if r["ID"] not in ids]

    try:
        save_data()

    except OSError:
        records[:] = old_records
        messagebox.showerror("Error", "Unable to save changes.")
        return

    details_label.configure(
        text="Select a transaction to view its details."
    )

    refresh_all()


# =====================================================
# TRANSACTION DETAILS
# =====================================================

def show_details(event=None):

    selected = tree.selection()

    if not selected:
        return

    record = next(
        (r for r in records if r["ID"] == selected[0]),
        None
    )

    if record:
        details_label.configure(
            text=(
                f"{record['Type']}  |  {record['Category']}\n"
                f"Amount: {money(float(record['Amount']))}\n"
                f"Date: {record['Date']}\n"
                f"Description: {record['Description'] or 'No description'}"
            )
        )


# =====================================================
# SET MONTHLY BUDGET
# =====================================================

def set_budget():

    global budget

    win = ctk.CTkToplevel(app)
    win.title("Monthly Budget")
    win.geometry("400x290")
    win.configure(fg_color=BG)
    win.transient(app)
    win.grab_set()

    make_label(
        win, "SET YOUR MONTHLY BUDGET", 21, GOLD, "bold"
    ).pack(pady=(30, 8))

    make_label(
        win, "Plan your spending. Save more.", 12, MUTED
    ).pack(pady=5)

    entry = make_entry(win, "Enter monthly budget", 280)
    entry.pack(pady=18)
    entry.insert(0, str(budget))

    def save():

        global budget

        try:
            value = float(entry.get())

            if value <= 0:
                raise ValueError

        except ValueError:
            messagebox.showerror(
                "Invalid Budget",
                "Enter a positive budget.",
                parent=win
            )
            return

        old_budget = budget
        budget = value

        try:
            save_settings()

        except OSError:
            budget = old_budget
            messagebox.showerror(
                "Error", "Unable to save budget.", parent=win
            )
            return

        win.destroy()
        refresh_all()

    make_button(
        win, "SAVE BUDGET", save, GOLD, 170
    ).pack(pady=12)


# =====================================================
# SEARCH AND FILTER
# =====================================================

def refresh_table():

    for item in tree.get_children():
        tree.delete(item)

    search = search_entry.get().lower().strip()
    category = filter_category.get()
    transaction_type = filter_type.get()

    count = 0

    for r in reversed(records):

        if category != "All Categories" and r["Category"] != category:
            continue

        if transaction_type != "All Types" and r["Type"] != transaction_type:
            continue

        searchable = (
            r["Description"] + " " + r["Category"]
        ).lower()

        if search and search not in searchable:
            continue

        tree.insert(
            "",
            "end",
            iid=r["ID"],
            values=(
                r["Date"],
                r["Type"],
                r["Category"],
                money(float(r["Amount"])),
                r["Description"]
            ),
            tags=(r["Type"],)
        )

        count += 1

    record_count.configure(
        text=f"{count} transactions found"
    )


def clear_filters():
    search_entry.delete(0, "end")
    filter_category.set("All Categories")
    filter_type.set("All Types")
    refresh_table()


def search_changed(event=None):

    global search_job

    if search_job is not None:
        app.after_cancel(search_job)

    search_job = app.after(150, refresh_table)


# =====================================================
# EXPORT CSV
# =====================================================

def export_csv():

    if not records:
        messagebox.showinfo("No Data", "No transactions available.")
        return

    path = filedialog.asksaveasfilename(
        defaultextension=".csv",
        filetypes=[("CSV Files", "*.csv")],
        initialfile="FINTRACK_Report.csv"
    )

    if not path:
        return

    try:
        fields = [
            "ID", "Date", "Type",
            "Category", "Amount", "Description"
        ]

        with open(path, "w", newline="", encoding="utf-8") as f:
            writer = csv.DictWriter(f, fieldnames=fields)
            writer.writeheader()
            writer.writerows(records)

        messagebox.showinfo(
            "Success", "Finance report exported successfully!"
        )

    except OSError:
        messagebox.showerror("Error", "Could not export report.")


# =====================================================
# ANALYTICS REFRESH - FIXED
# =====================================================

def refresh_analytics():

    # This function updates the analytics summary.
    # It is safe to call whenever transactions change.

    if "analytics_summary" not in globals():
        return

    data = monthly_records()
    income, expense = get_totals(data)
    balance = income - expense

    if income > 0:
        savings = (balance / income) * 100
    else:
        savings = 0

    analytics_summary.configure(
        text=(
            f"MONTHLY INCOME: {money(income)}     |     "
            f"MONTHLY EXPENSES: {money(expense)}     |     "
            f"NET BALANCE: {money(balance)}     |     "
            f"SAVINGS: {savings:.1f}%"
        )
    )


# =====================================================
# FINANCIAL ANALYTICS CHART
# =====================================================

def show_chart():

    data = monthly_records()

    if not data:
        messagebox.showinfo(
            "No Data",
            "Add transactions for this month to view analytics."
        )
        return

    income_categories = {}
    expense_categories = {}

    for r in data:

        category = r["Category"]
        amount = float(r["Amount"])

        target = (
            income_categories
            if r["Type"] == "Income"
            else expense_categories
        )

        target[category] = target.get(category, 0) + amount

    win = ctk.CTkToplevel(app)
    win.title("FINTRACK | Financial Analytics")
    win.geometry("950x680")
    win.configure(fg_color=BG)
    win.transient(app)

    make_label(
        win, "YOUR FINANCIAL INSIGHTS", 25, LAVENDER, "bold"
    ).pack(pady=18)

    figure = Figure(figsize=(9, 5.5), dpi=100)
    figure.patch.set_facecolor(BG)

    ax1 = figure.add_subplot(121)
    ax2 = figure.add_subplot(122)

    figure.subplots_adjust(
        left=0.06,
        right=0.95,
        top=0.85,
        bottom=0.22,
        wspace=0.4
    )

    if expense_categories:

        ax1.pie(
            list(expense_categories.values()),
            labels=list(expense_categories.keys()),
            autopct="%1.1f%%",
            startangle=90,
            colors=[
                "#B69CFF", "#55D6BE", "#FF718A",
                "#FFC978", "#D0B5FF", "#83B8FF",
                "#F59CC8", "#80C8A8"
            ],
            textprops={"color": WHITE, "fontsize": 8}
        )

    else:
        ax1.text(
            0.5, 0.5, "No expenses",
            ha="center", color=WHITE
        )

    ax1.set_title(
        "Expense Distribution",
        color=WHITE,
        fontsize=12
    )

    categories = sorted(
        set(income_categories) | set(expense_categories)
    )

    x = list(range(len(categories)))
    width = 0.36

    income_values = [
        income_categories.get(c, 0) for c in categories
    ]

    expense_values = [
        expense_categories.get(c, 0) for c in categories
    ]

    ax2.bar(
        [v - width / 2 for v in x],
        income_values,
        width,
        label="Income",
        color=GREEN
    )

    ax2.bar(
        [v + width / 2 for v in x],
        expense_values,
        width,
        label="Expense",
        color=RED
    )

    ax2.set_xticks(x)
    ax2.set_xticklabels(
        categories,
        rotation=35,
        ha="right",
        color=WHITE,
        fontsize=8
    )

    ax2.tick_params(axis="y", colors=MUTED)
    ax2.set_title("Income vs Expenses", color=WHITE, fontsize=12)
    ax2.legend(facecolor=CARD, labelcolor=WHITE)
    ax2.set_facecolor(CARD)
    ax2.grid(axis="y", alpha=0.15)

    for spine in ax2.spines.values():
        spine.set_visible(False)

    canvas = FigureCanvasTkAgg(figure, master=win)
    canvas.draw()

    canvas.get_tk_widget().pack(
        fill="both",
        expand=True,
        padx=15,
        pady=10
    )

    income, expense = get_totals(data)

    make_label(
        win,
        f"NET MONTHLY BALANCE: {money(income - expense)}",
        17,
        GREEN if income >= expense else RED,
        "bold"
    ).pack(pady=15)


# =====================================================
# DASHBOARD REFRESH
# =====================================================

def refresh_all():

    income, expense = get_totals(records)
    balance = income - expense

    month_income, month_expense = get_totals(
        monthly_records()
    )

    total_income.configure(text=money(income))
    total_expense.configure(text=money(expense))
    total_balance.configure(text=money(balance))

    savings = (balance / income * 100) if income > 0 else 0

    savings_value.configure(text=f"{savings:.1f}%")

    monthly_income_label.configure(text=money(month_income))
    monthly_spending.configure(text=money(month_expense))

    percentage = month_expense / budget

    budget_progress.set(min(percentage, 1))

    if percentage >= 1:
        budget_progress.configure(progress_color=RED)
        budget_status.configure(
            text="⚠ Budget exceeded!",
            text_color=RED
        )

    elif percentage >= 0.8:
        budget_progress.configure(progress_color=GOLD)
        budget_status.configure(
            text="⚠ Approaching your budget limit",
            text_color=GOLD
        )

    else:
        budget_progress.configure(progress_color=GREEN)
        budget_status.configure(
            text="✓ You are within your budget",
            text_color=GREEN
        )

    budget_text.configure(
        text=f"{money(month_expense)} spent out of {money(budget)}"
    )

    refresh_table()

    # FIX: Now the function is properly defined.
    refresh_analytics()


# =====================================================
# SIDEBAR
# =====================================================

sidebar = ctk.CTkFrame(
    app,
    width=235,
    fg_color=SIDEBAR,
    corner_radius=0,
    border_width=1,
    border_color=BORDER
)

sidebar.pack(side="left", fill="y")
sidebar.pack_propagate(False)

make_label(
    sidebar, "✦ FINTRACK", 28, LAVENDER, "bold"
).pack(pady=(35, 5))

make_label(
    sidebar, "YOUR MONEY. YOUR CONTROL.", 9, MUTED, "bold"
).pack(pady=(0, 40))

make_label(
    sidebar, "WORKSPACE", 11, MUTED, "bold"
).pack(anchor="w", padx=22, pady=10)


# =====================================================
# MAIN CONTENT
# =====================================================

main = ctk.CTkFrame(
    app,
    fg_color=BG,
    corner_radius=0
)

main.pack(side="right", fill="both", expand=True)

header = ctk.CTkFrame(
    main,
    fg_color="transparent"
)

header.pack(fill="x", padx=28, pady=22)

make_label(
    header, "Your Finance Dashboard", 26, WHITE, "bold"
).pack(side="left")


# =====================================================
# TABS
# =====================================================

tabs = ctk.CTkTabview(
    main,
    fg_color=BG,
    segmented_button_fg_color=CARD,
    segmented_button_selected_color=LAVENDER,
    segmented_button_selected_hover_color="#59477A",
    segmented_button_unselected_color=CARD2,
    text_color=WHITE,
    corner_radius=14
)

tabs.pack(
    fill="both",
    expand=True,
    padx=22,
    pady=(0, 18)
)

dashboard_tab = tabs.add("Overview")
transactions_tab = tabs.add("Transactions")
analytics_tab = tabs.add("Analytics")


# =====================================================
# SIDEBAR NAVIGATION
# =====================================================

make_button(
    sidebar, "▦   Overview",
    lambda: tabs.set("Overview"), CARD
).pack(fill="x", padx=15, pady=7)

make_button(
    sidebar, "＋   Add Transaction",
    lambda: open_transaction(False), CARD2
).pack(fill="x", padx=15, pady=7)

make_button(
    sidebar, "◷   Transactions",
    lambda: tabs.set("Transactions"), CARD2
).pack(fill="x", padx=15, pady=7)

make_button(
    sidebar, "◉   Analytics",
    lambda: tabs.set("Analytics"), CARD2
).pack(fill="x", padx=15, pady=7)

make_button(
    sidebar, "⚙   Monthly Budget",
    set_budget, CARD2
).pack(fill="x", padx=15, pady=7)

ctk.CTkFrame(
    sidebar, fg_color="transparent"
).pack(fill="both", expand=True)

make_label(
    sidebar, "SMART MONEY MANAGEMENT", 9, MUTED, "bold"
).pack(pady=5)

make_label(
    sidebar, "PREMIUM EDITION  •  4.1", 10, LAVENDER, "bold"
).pack(pady=(5, 25))


# =====================================================
# OVERVIEW TAB
# =====================================================

make_label(
    dashboard_tab,
    "Your financial performance",
    20, WHITE, "bold"
).pack(anchor="w", padx=15, pady=(18, 20))

cards = ctk.CTkFrame(
    dashboard_tab,
    fg_color="transparent"
)

cards.pack(fill="x", padx=5)


def create_card(parent, title, color):

    card = ctk.CTkFrame(
        parent,
        fg_color=CARD,
        corner_radius=15,
        height=115,
        border_width=1,
        border_color=BORDER
    )

    card.pack(
        side="left",
        fill="x",
        expand=True,
        padx=6
    )

    card.pack_propagate(False)

    make_label(
        card, title, 11, MUTED, "bold"
    ).pack(anchor="w", padx=15, pady=(17, 5))

    value = make_label(
        card, "₹0.00", 21, color, "bold"
    )

    value.pack(anchor="w", padx=15)

    return value


total_income = create_card(cards, "TOTAL INCOME", GREEN)
total_expense = create_card(cards, "TOTAL EXPENSES", RED)
total_balance = create_card(cards, "REMAINING BALANCE", LAVENDER)
savings_value = create_card(cards, "SAVINGS RATE", GOLD)


# =====================================================
# BUDGET CARD
# =====================================================

budget_card = ctk.CTkFrame(
    dashboard_tab,
    fg_color=CARD,
    corner_radius=16,
    border_width=1,
    border_color=BORDER
)

budget_card.pack(fill="x", padx=7, pady=22)

make_label(
    budget_card, "✦  MONTHLY BUDGET TRACKER",
    16, LAVENDER, "bold"
).pack(anchor="w", padx=20, pady=(18, 8))

budget_text = make_label(
    budget_card, "", 12, MUTED
)

budget_text.pack(anchor="w", padx=20)

budget_progress = ctk.CTkProgressBar(
    budget_card,
    height=14,
    corner_radius=10,
    fg_color=CARD2,
    progress_color=GREEN
)

budget_progress.pack(fill="x", padx=20, pady=14)
budget_progress.set(0)

budget_status = make_label(
    budget_card, "", 12, GREEN, "bold"
)

budget_status.pack(anchor="w", padx=20, pady=(0, 17))


# =====================================================
# MONTHLY SUMMARY
# =====================================================

summary_frame = ctk.CTkFrame(
    dashboard_tab,
    fg_color="transparent"
)

summary_frame.pack(fill="x", padx=5, pady=5)

monthly_income_label = create_card(
    summary_frame, "THIS MONTH'S INCOME", GREEN
)

monthly_spending = create_card(
    summary_frame, "THIS MONTH'S SPENDING", RED
)

make_button(
    dashboard_tab,
    "＋  ADD NEW TRANSACTION",
    lambda: open_transaction(False),
    LAVENDER,
    230
).pack(pady=22)


# =====================================================
# TRANSACTIONS TAB
# =====================================================

make_label(
    transactions_tab,
    "TRANSACTION MANAGEMENT",
    20, WHITE, "bold"
).pack(anchor="w", padx=12, pady=15)

search_frame = ctk.CTkFrame(
    transactions_tab,
    fg_color="transparent"
)

search_frame.pack(fill="x", padx=10, pady=8)

search_entry = ctk.CTkEntry(
    search_frame,
    placeholder_text="⌕  Search transactions...",
    width=260,
    height=40,
    fg_color=CARD2,
    border_color=BORDER
)

search_entry.pack(side="left", padx=5)

filter_category = ctk.CTkOptionMenu(
    search_frame,
    values=["All Categories"] + CATEGORIES,
    fg_color=CARD2,
    button_color=LAVENDER,
    width=150,
    command=lambda choice: refresh_table()
)

filter_category.pack(side="left", padx=5)
filter_category.set("All Categories")

filter_type = ctk.CTkOptionMenu(
    search_frame,
    values=["All Types", "Income", "Expense"],
    fg_color=CARD2,
    button_color=LAVENDER,
    width=130,
    command=lambda choice: refresh_table()
)

filter_type.pack(side="left", padx=5)
filter_type.set("All Types")

make_button(
    search_frame, "Clear",
    clear_filters, CARD2, 75
).pack(side="right", padx=5)

search_entry.bind("<KeyRelease>", search_changed)


# =====================================================
# TRANSACTION TABLE
# =====================================================

table_frame = ctk.CTkFrame(
    transactions_tab,
    fg_color=CARD,
    corner_radius=13,
    border_width=1,
    border_color=BORDER
)

table_frame.pack(fill="both", expand=True, padx=10, pady=10)

style = ttk.Style()
style.theme_use("clam")

style.configure(
    "Treeview",
    background="#19142A",
    foreground=WHITE,
    fieldbackground="#19142A",
    rowheight=39,
    font=("Segoe UI", 10),
    borderwidth=0
)

style.configure(
    "Treeview.Heading",
    background=CARD2,
    foreground=LAVENDER,
    font=("Segoe UI", 10, "bold"),
    padding=10
)

style.map(
    "Treeview",
    background=[("selected", "#59477A")],
    foreground=[("selected", "white")]
)

columns = ("Date", "Type", "Category", "Amount", "Description")

tree = ttk.Treeview(
    table_frame,
    columns=columns,
    show="headings",
    selectmode="extended"
)

for col in columns:
    tree.heading(col, text=col.upper())

tree.column("Date", width=110, anchor="center")
tree.column("Type", width=100, anchor="center")
tree.column("Category", width=130, anchor="center")
tree.column("Amount", width=140, anchor="center")
tree.column("Description", width=260, anchor="w")

tree.tag_configure("Income", foreground=GREEN)
tree.tag_configure("Expense", foreground=RED)

scrollbar = ttk.Scrollbar(
    table_frame,
    orient="vertical",
    command=tree.yview
)

tree.configure(yscrollcommand=scrollbar.set)

tree.pack(side="left", fill="both", expand=True, padx=8, pady=8)
scrollbar.pack(side="right", fill="y")

tree.bind("<<TreeviewSelect>>", show_details)

details_label = make_label(
    transactions_tab,
    "Select a transaction to view its details.",
    12, MUTED
)

details_label.pack(anchor="w", padx=15, pady=8)

action_frame = ctk.CTkFrame(
    transactions_tab,
    fg_color="transparent"
)

action_frame.pack(fill="x", padx=10, pady=10)

record_count = make_label(
    action_frame, "0 transactions found", 11, MUTED
)

record_count.pack(side="left")

make_button(
    action_frame, "✎ EDIT",
    lambda: open_transaction(True), LAVENDER, 100
).pack(side="right", padx=5)

make_button(
    action_frame, "DELETE",
    delete_transaction, RED, 100
).pack(side="right", padx=5)

make_button(
    action_frame, "EXPORT CSV",
    export_csv, CARD2, 110
).pack(side="right", padx=5)


# =====================================================
# ANALYTICS TAB
# =====================================================

make_label(
    analytics_tab,
    "YOUR FINANCIAL INSIGHTS",
    21, LAVENDER, "bold"
).pack(anchor="w", padx=15, pady=20)

# IMPORTANT: Create this label BEFORE refresh_all() is called.

analytics_summary = make_label(
    analytics_tab,
    "Monthly financial summary will appear here.",
    12, GREEN, "bold"
)

analytics_summary.pack(
    anchor="w",
    padx=15,
    pady=10
)

chart_frame = ctk.CTkFrame(
    analytics_tab,
    fg_color=CARD,
    corner_radius=16,
    border_width=1,
    border_color=BORDER
)

chart_frame.pack(
    fill="both",
    expand=True,
    padx=12,
    pady=12
)

make_label(
    chart_frame,
    "Understand your spending habits.",
    17, WHITE, "bold"
).pack(pady=(65, 10))

make_label(
    chart_frame,
    "Visualize income, expenses and category-wise spending.",
    12, MUTED
).pack(pady=5)

make_button(
    analytics_tab,
    "✦  OPEN FINANCIAL ANALYSIS",
    show_chart,
    LAVENDER,
    260
).pack(pady=18)


# =====================================================
# INITIALIZE APPLICATION
# =====================================================

refresh_all()

app.mainloop()
