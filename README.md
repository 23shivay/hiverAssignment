# PART B

This project classifies customer support emails using a two-stage pipeline:

1. **Stage 1:** The LLM predicts a universal category.
2. **Stage 2:** The system maps that category to the correct customer-specific tag.

This ensures accuracy, consistency, and strict customer isolation.

---

## Approach

The system uses a **universal-first classification workflow**:

### ✔ Stage 1 — Universal Category  
The LLM reads both **subject + body** and predicts one standardized category from a controlled list.  
This avoids customer bias and keeps predictions stable.

### ✔ Stage 2 — Customer Tag Mapping  
Each customer has their own taxonomy.  
The model does *not* learn customer tags.  
Instead, we map the universal category → customer tag **only in code**.

This prevents leakage of one customer’s tags to another.

---

##  Model & Prompt

### Model Used
llama-3.1-8b-instant

prompt=You are an email classifier for customer support tickets.

Classify the email into EXACTLY one of these categories:
{UNIVERSAL_CATEGORIES}

Rules:
- Use both subject and body.
- Pick the most accurate universal category.
- Only choose "other_issue" if none of the categories fit.

Email:
Customer: {customer}
Subject: {subject}
Body: {body}

Return ONLY the category name.


### Arch Flow
                 ┌───────────────────────────────┐
                 │        Incoming Email          │
                 │  (customer_id, subject, body)  │
                 └───────────────────────────────┘
                                  │
                                  ▼
                ┌─────────────────────────────────┐
                │         STAGE 1 (LLM)           │
                │  Classify into UNIVERSAL label  │
                ├─────────────────────────────────┤
                │  Example output: "workflow"     │
                └─────────────────────────────────┘
                                  │
                                  ▼
        ┌────────────────────────────────────────────────────┐
        │            STAGE 2 (Code Mapping Layer)            │
        │  Map universal label → customer-specific tag       │
        │                                                    │
        │  CUSTOMER_TAGGING[customer_id][universal_label]    │
        └────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌────────────────────────┐
                    │  Final Customer Tag     │
                    │ e.g., "workflow_bug"    │
                    └────────────────────────┘



## How Customer Isolation Is Ensured

Customer isolation is handled through design-level controls rather than only prompt instructions.

### ✔ 1. Universal Categories Prevent Cross-Customer Leakage
The model predicts **only high-level universal categories** that are the same across all customers.  
This ensures the LLM never directly assigns customer-specific tags.

### ✔ 2. Customer-Specific Tags Mapped in Code 
Each customer maintains its own tagging dictionary:

```python
CUSTOMER_TAGGING = {
    "CUST_A": { ... },
    "CUST_B": { ... },
    "CUST_C": { ... }
}

