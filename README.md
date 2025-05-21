# streamlit_app.py
# Streamlit ≥ 1.32 -- run with:  streamlit run streamlit_app.py
# Author: <your name or clinic>, 2025
# Sources: WHO Child Growth Standards (2006), WHO/FAO/UNU EER (2004),
#          Peruvian MoH energy PAL tables (2017), ISPAD CPG 2022 ch. 10,
#          ADA Exchange Lists for Diabetes, 2023.

import streamlit as st
import pandas as pd
from math import ceil

# --------------------- CONSTANTS ------------------------

WHO_EER_GIRLS_3_10 = lambda w: 22.5 * w + 499                          # kcal · day⁻¹
WHO_EER_BOYS_3_10  = lambda w: 22.7 * w + 495

PAL_OPTIONS = {  # Peruvian MoH (or change to your context)
    "Sedentary (1.4)": 1.4,
    "Low active (1.6)": 1.6,
    "Active (1.8)": 1.8,
    "Very active (2.0)": 2.0,
}

MEAL_SPLIT = {
    "Breakfast": 0.20,
    "Mid-morning": 0.10,
    "Lunch": 0.25,
    "Mid-afternoon": 0.10,
    "Dinner": 0.25,
    "After-dinner": 0.10,
}

# Fixed food-group targets (exchanges or servings per day)
FIXED_FOOD_GROUPS = {
    "Fruit (15 g/serv)": 3,      # 3 exchanges ↔ 45 g CHO
    "Cooked veg (5 g/serv)": 2,  # 2 servings ↔ 10 g CHO
    "Dairy (12 g/serv)": 2,      # 2 servings ↔ 24 g CHO
}
# CHO grams contributed by fixed groups
FIXED_CHO_G = 45 + 10 + 24

EXCH_G = 15  # 1 ADA exchange = 15 g carbohydrate

# --------------------- SIDEBAR INPUTS -------------------

st.sidebar.header("Child information")

age = st.sidebar.number_input("Age (years)", min_value=3.0, max_value=18.0, value=7.0, step=0.5)
sex = st.sidebar.selectbox("Sex", options=["Female", "Male"])
weight = st.sidebar.number_input("Weight (kg)", min_value=8.0, max_value=120.0, value=22.0, step=0.1)
height = st.sidebar.number_input("Height (cm)", min_value=50.0, max_value=200.0, value=120.0, step=0.1)
pal_label = st.sidebar.selectbox("Physical-activity level", options=list(PAL_OPTIONS.keys()))
carb_pct = st.sidebar.slider("Percent of energy from carbohydrates (%, ISPAD 45–55)", 40, 60, 50)

st.sidebar.markdown("---")
show_example = st.sidebar.checkbox("🔎  Load ISPAD example (7-yo girl)", value=False)

if show_example:
    age, sex, weight, height, pal_label, carb_pct = 7, "Female", 22.0, 120.0, "Low active (1.6)", 50
    st.sidebar.success("Example loaded!")

# --------------------- CALCULATIONS ---------------------

# 1) Total energy requirement
eer_fn = WHO_EER_GIRLS_3_10 if sex == "Female" else WHO_EER_BOYS_3_10
bmr = eer_fn(weight)
total_kcal = bmr * PAL_OPTIONS[pal_label]

# 2) Total daily carbohydrate grams & exchanges
total_carb_g = total_kcal * carb_pct / 100 / 4
total_exchanges = total_carb_g / EXCH_G

# 3) Meal-by-meal carbohydrate grams
meal_grams = {meal: total_carb_g * frac for meal, frac in MEAL_SPLIT.items()}

# 4) Fixed food-group CHO allocation
remaining_carb_g = max(total_carb_g - FIXED_CHO_G, 0)
starchy_exchanges = remaining_carb_g / EXCH_G

# 5) Build final output table
rows = []
for meal, frac in MEAL_SPLIT.items():
    g = meal_grams[meal]
    rows.append({
        "Meal": meal,
        "CHO (g)": round(g, 1),
        "Exchanges": round(g / EXCH_G, 1),
    })
output_df = pd.DataFrame(rows).set_index("Meal")

summary_df = pd.DataFrame({
    "Item": ["Total kcal", "Total CHO (g)", "Total exchanges",
             "Fruit exch.", "Veg serv.", "Dairy serv.", "Starchy exch."],
    "Value": [round(total_kcal, 0), round(total_carb_g, 1), round(total_exchanges, 1),
              FIXED_FOOD_GROUPS["Fruit (15 g/serv)"],
              FIXED_FOOD_GROUPS["Cooked veg (5 g/serv)"],
              FIXED_FOOD_GROUPS["Dairy (12 g/serv)"],
              round(starchy_exchanges, 1)]
}).set_index("Item")

# --------------------- DISPLAY -------------------------

st.title("Meal-plan calculator for children & adolescents with type 1 diabetes")
st.caption("↳ Based on WHO growth standards, WHO/FAO/UNU energy equations, Peruvian PAL tables, "
           "ISPAD 2022 Ch. 10, and ADA Exchange Lists.")

st.subheader("Daily summary")
st.table(summary_df)

st.subheader("Meal-by-meal carbohydrate targets")
st.table(output_df)

st.download_button(
    label="📄 Download printable table (CSV)",
    data=output_df.to_csv(index=True).encode("utf-8"),
    file_name="carbohydrate_plan.csv",
    mime="text/csv",
)

st.markdown("---")
st.markdown("**How to use this tool**  \n"
            "1. Verify or adjust weight, height, and activity level.  \n"
            "2. Hand-check that the calculated kcal matches the child’s clinical picture.  \n"
            "3. Teach the family to count exchanges in each food group.  \n"
            "4. Update the diet plan at every clinic visit or growth crossing.")

st.info("ℹ️ For full WHO z-score assessment you can add the `who-child-growth-standards` "
        "Python package or connect to your EMR’s anthropometry module.")

# --------------------- END ------------------------------
