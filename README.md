# AI-Payment-Reconciliation-Copilot

import pandas as pd
import streamlit as st

st.set_page_config(
    page_title="AI Payment Reconciliation Copilot",
    page_icon="💳",
    layout="wide"
)

st.title("💳 AI Payment Reconciliation Copilot")
st.write(
    "Compare bank transactions against settlement records "
    "and identify discrepancies."
)

st.info(
    "Portfolio demo only. Use synthetic data. "
    "This application does not move money or modify bank records."
)


def reconcile(bank, settlement):
    bank = bank.copy()
    settlement = settlement.copy()

    bank.columns = bank.columns.str.strip().str.lower()
    settlement.columns = settlement.columns.str.strip().str.lower()

    required = {"transaction_id", "amount"}

    if not required.issubset(bank.columns):
        raise ValueError(
            "Bank CSV must contain transaction_id and amount."
        )

    if not required.issubset(settlement.columns):
        raise ValueError(
            "Settlement CSV must contain transaction_id and amount."
        )

    for df in (bank, settlement):
        df["transaction_id"] = (
            df["transaction_id"].astype(str).str.strip()
        )
        df["amount"] = pd.to_numeric(
            df["amount"], errors="coerce"
        ).round(2)

        for column in ("currency", "status"):
            if column not in df.columns:
                df[column] = ""
            df[column] = df[column].fillna("").astype(str).str.strip()

    bank_duplicates = set(
        bank.loc[
            bank["transaction_id"].duplicated(keep=False),
            "transaction_id"
        ]
    )
    settlement_duplicates = set(
        settlement.loc[
            settlement["transaction_id"].duplicated(keep=False),
            "transaction_id"
        ]
    )

    bank_map = {
        key: group for key, group
        in bank.groupby("transaction_id", sort=False)
    }
    settlement_map = {
        key: group for key, group
        in settlement.groupby("transaction_id", sort=False)
    }

    transaction_ids = list(dict.fromkeys(
        bank["transaction_id"].tolist()
        + settlement["transaction_id"].tolist()
    ))

    results = []

    for transaction_id in transaction_ids:
        b = bank_map.get(transaction_id)
        s = settlement_map.get(transaction_id)

        bank_row = b.iloc[0] if b is not None else None
        settlement_row = s.iloc[0] if s is not None else None

        if (
            transaction_id in bank_duplicates
            or transaction_id in settlement_duplicates
        ):
            status = "Potential duplicate"

        elif bank_row is None:
            status = "Missing in bank"

        elif settlement_row is None:
            status = "Missing in settlement"

        elif pd.isna(bank_row["amount"]) or pd.isna(
            settlement_row["amount"]
        ):
            status = "Invalid amount"

        elif bank_row["amount"] != settlement_row["amount"]:
            status = "Amount mismatch"

        elif (
            bank_row["currency"]
            and settlement_row["currency"]
            and bank_row["currency"].upper()
            != settlement_row["currency"].upper()
        ):
            status = "Currency mismatch"

        elif (
            bank_row["status"]
            and settlement_row["status"]
            and bank_row["status"].lower()
            != settlement_row["status"].lower()
        ):
            status = "Status mismatch"

        else:
            status = "Matched"

        next_steps = {
            "Matched": "Retain reconciliation evidence.",
            "Missing in bank": "Check bank ingestion and posting delays.",
            "Missing in settlement": "Check settlement file coverage.",
            "Amount mismatch": "Investigate fees, reversals and adjustments.",
            "Currency mismatch": "Verify currency mapping and conversion.",
            "Status mismatch": "Check payment lifecycle status.",
            "Potential duplicate": "Review all duplicate source rows.",
            "Invalid amount": "Investigate the source amount."
        }

        results.append({
            "transaction_id": transaction_id,
            "status": status,
            "bank_amount": (
                None if bank_row is None else bank_row["amount"]
            ),
            "settlement_amount": (
                None if settlement_row is None
                else settlement_row["amount"]
            ),
            "suggested_next_step": next_steps[status]
        })

    return pd.DataFrame(results)


# Synthetic sample records for testing
sample_bank = pd.DataFrame([
    {"transaction_id": "TX1001", "amount": 1200,
     "currency": "INR", "status": "SETTLED"},
    {"transaction_id": "TX1002", "amount": 500,
     "currency": "INR", "status": "SETTLED"},
    {"transaction_id": "TX1003", "amount": 800,
     "currency": "INR", "status": "SETTLED"},
    {"transaction_id": "TX1004", "amount": 300,
     "currency": "INR", "status": "PENDING"},
    {"transaction_id": "TX1005", "amount": 250,
     "currency": "INR", "status": "SETTLED"},
])

sample_settlement = pd.DataFrame([
    {"transaction_id": "TX1001", "amount": 1200,
     "currency": "INR", "status": "SETTLED"},
    {"transaction_id": "TX1002", "amount": 500,
     "currency": "INR", "status": "SETTLED"},
    {"transaction_id": "TX1003", "amount": 780,
     "currency": "INR", "status": "SETTLED"},
    {"transaction_id": "TX1004", "amount": 300,
     "currency": "INR", "status": "SETTLED"},
    {"transaction_id": "TX1006", "amount": 450,
     "currency": "INR", "status": "SETTLED"},
])

st.subheader("1. Choose your data")

use_sample = st.checkbox("Use built-in sample data", value=True)

if use_sample:
    bank = sample_bank
    settlement = sample_settlement
else:
    left, right = st.columns(2)

    with left:
        bank_file = st.file_uploader(
            "Upload bank CSV", type=["csv"]
        )

    with right:
        settlement_file = st.file_uploader(
            "Upload settlement CSV", type=["csv"]
        )

    bank = (
        pd.read_csv(bank_file)
        if bank_file is not None else None
    )
    settlement = (
        pd.read_csv(settlement_file)
        if settlement_file is not None else None
    )

if st.button("Run reconciliation", type="primary"):
    if bank is None or settlement is None:
        st.error("Please upload both CSV files.")
    else:
        try:
            st.session_state["results"] = reconcile(
                bank, settlement
            )
        except (ValueError, KeyError) as error:
            st.error(str(error))

results = st.session_state.get("results")

if results is not None:
    st.subheader("2. Reconciliation results")

    total = len(results)
    matched = int((results["status"] == "Matched").sum())
    exceptions = total - matched

    c1, c2, c3 = st.columns(3)
    c1.metric("Transaction IDs", total)
    c2.metric("Matched", matched)
    c3.metric("Exceptions", exceptions)

    st.bar_chart(results["status"].value_counts())

    st.dataframe(results, use_container_width=True)

    st.subheader("3. Investigate exceptions")

    for _, row in results[
        results["status"] != "Matched"
    ].iterrows():
        with st.expander(
            f"{row['transaction_id']} — {row['status']}"
        ):
            st.write(row["suggested_next_step"])

    st.download_button(
        "Download reconciliation report",
        data=results.to_csv(index=False).encode("utf-8"),
        file_name="reconciliation_report.csv",
        mime="text/csv"
    )

st.caption(
    "Prototype: exact transaction ID matching. "
    "Review exceptions before marking them resolved."
)
