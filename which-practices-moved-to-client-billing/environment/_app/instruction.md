# System Prompt
You are a careful task-execution agent. Use the tools provided by the configured gym to complete the user's task. Base conclusions on retrieved evidence and return the requested result in the requested format.

# Task
Write /workspace/practice_billing_models.xlsx showing how these practices were billed: Johnson & Associates, Over
The Moon, Canyon Midwifery, Kellie Botts, Authentic Birth, BirthStone, Birthing Hues, Jamii, Joyful Heart, Kindred
Space, Profound Birth, Empowered Goddess, Des Moines and Elke Barnes. A practice's statements are my Drive
spreadsheets whose titles begin with its name, plus any raw export titled with nothing but the practice ID its
statements carry as their tab name. Nothing in the trash counts.

A billing row is any row below a statement's header row with an order id; its Bill Type cell (however that header is
spelled) reads cash pay or client bill, however written. A practice is silent when it has no billing row, and
switched when its billing rows carry both bill types.

The unit of audit is the practice: one practice (`listed name`) may carry several billing rows. Bill types are assessed billing row by billing row; the summary counts of billed practices and of each bill type are counts of practices, by the bill type reported for each billing row in the Billing sheet, a practice counting under each bill type that one or more of its billing rows carry.

On a sheet named Billing, give each billing row its own row, sorted by its spreadsheet's Drive file id, ascending,
then position in its statement: column A that file id, column B the order id, column C TRUE for client bill or FALSE
for cash pay. On a sheet named Summary, each figure on its own row in this order: billed_practices,
cash_pay_practices, client_bill_practices, switched_practices and silent_practices, name in column A, whole number in
column B, typed as a number, not as text or a formula. Both start at row 1, no header row. Leave every other cell of
Billing and Summary unformatted and blank
(not even an empty string); use ordinary worksheets and keep notes elsewhere.
