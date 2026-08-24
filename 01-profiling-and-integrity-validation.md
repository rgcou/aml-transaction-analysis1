# Part I — Profiling and Integrity Validation

> The full code for this phase is in the accompanying notebook. This document explains the reasoning and findings.

## 1. Initial Exploration

### 1.1. Accounts Table — Distinct and Unique Values

I will start by exploring the composition of the table LI-Small_accounts, that I will onwards call "dfaccounts".

It is important to start by acknowledging that each row, this is, the unit of analysis, is a bank account, which can be deduced from the name of the file that contains the table and because its complementary table shows the transactions made with these bank accounts.

By counting rows I can see there are 712,688 rows, supposedly representative of 712,688 different bank accounts.

**Columns.** The table has 5 attributes:

- **Bank Name:** type string; it shows the name of the bank.
- **Bank ID:** type integer; it is a primary key, a unique number to identify each bank.
- **Account Number:** type string; it is another primary key, a unique string to identify each account.
- **Entity ID:** type string; it is also a primary key, a unique string to identify the account holder.
- **Entity Name:** type string; it is the name of the account holder.

I uploaded the table to PowerBI, which identified correctly the data type of all columns. All columns have 100% of valid values (i.e., their data type matches the identified data type), 0% of empty values (i.e. no null values) and 0% of errors (i.e. all values have been correctly converted to their data type).

### 1.2. Accounts Table — Cardinality Profile. First Observations

Below I comment on the column observations that have cardinal inconsistencies visible at first glance (marked in **bold** where an inconsistency appears).

- **Bank Name:** 27,652 distinct and 5,691 unique values. This means that we would be getting data out of 712,688 accounts from 27,652 different banks, which gives us an average of a bit more than 25 accounts per bank. This number can help us measure proportions when it comes to see how many accounts actually correspond to a certain bank.
- **Bank ID (inconsistency):** 41,815 distinct and 8,447 unique values. The id should be a unique number to identify a certain bank and should be repeated every time the same bank is repeated. We can see though, there are 41,815 distinct ids for 27,652 bank names, so as a first observation, we can see there might be something wrong that might have to be addressed during the data wrangling process.
- **Account Number (inconsistency):** 712,684 distinct and 712,680 unique values. We can see a difference with respect to the 712,688 rows and 712,680 uniques, which can indicate repeated values, for every row should represent a different account. This is not a problem per se, but I think it would be good to take a look at what is happening here.
- **Entity ID and Entity Name:** 224,931 distinct and 145,348 unique values. These two columns share the same amount of distinct and unique values, which is coherent, because every repeated entity name has to have its own repeated entity identification. It can be seen that there is more than one account per entity.

### 1.3. Transactions Table — Distinct and Unique Values

#### 1.3.1. Transactions Table Columns and Their Data Type

Each row of this table represents a monetary transaction made between two bank accounts out of the accounts represented in the accounts table I just described.

The table has 6,924,049 rows, so when I try to profile it based on the entire dataset with PowerBI, it is not responding due to the amount of data (about 9.7 times the size of the accounts table, which makes sense, since every account usually makes many transactions). I will therefore profile it using Pandas.

The transactions table has 11 columns:

- **Timestamp:** type date/time; it shows the date and time each bank transaction was made.
- **From Bank:** type integer; it shows the bank identification number the transaction comes from. It should be a foreign key pointing to "dfaccounts.Bank ID".
- **Account:** type string; it shows the account number from the accounts table from which the transaction has been made. It should also be a foreign key pointing to "dfaccounts.Account Number".
- **To Bank:** type integer; it shows the destination bank. It should also be a foreign key pointing to "dfaccounts.Bank ID".
- **Account.1:** type string; it is not clear from a simple look what this attribute means but it probably is the account number of the transaction destination, so it should also be a foreign key pointing to "dfaccounts.Account Number".
- **Amount Received:** it is the amount that has been transferred to the destination account.
- **Receiving Currency:** it is the currency in which the Amount Received value has been received.
- **Amount Paid:** it seems to be the value the paying entity is transferring to Account.1. I should check though if Amount Received and Amount Paid have different values and are not repeated columns.
- **Payment Currency:** it is the currency of the Amount Paid value.
- **Payment Format:** it is the means by which the money transfer has been made.
- **Is Laundering:** I will not work with this column, since the goal of this project is precisely to determine which transactions might be money laundering.

#### 1.3.2. Calculating Distinct and Unique Values of the Transactions Table

Counting distinct and unique values for each column of the transactions table (note that "unique" in Pandas equals "distinct" in PowerBI), the results are:

| Column | Distinct | Unique |
|---|---|---|
| Timestamp | 14,533 | 53 |
| From Bank | 41,814 | 4,714 |
| Account | 681,281 | 211,825 |
| To Bank | 21,588 | 5,156 |
| Account.1 | 576,176 | 157,062 |
| Amount Received | 1,194,921 | 615,389 |
| Receiving Currency | 15 | 0 |
| Amount Paid | 1,204,309 | 621,441 |
| Payment Currency | 15 | 0 |
| Payment Format | 7 | 0 |

### 1.4. Transactions Table — Cardinality Profile. First Observations

- **Timestamp:** 14,533 distinct values and 53 unique values.
- **From Bank:** 41,814 distinct values and 4,714 unique values. This column constitutes a foreign key pointing to "dfaccounts.Bank ID", so these 41,814 distinct values are over the 41,815 distinct bank IDs.
- **Account:** 681,281 distinct values, and 211,825 unique values. In the accounts table we had 712,684 distinct account numbers, so there is no inconsistency here: we have the transaction data of almost all accounts (more than 95%) and some accounts are not having any transaction.
- **To Bank:** 21,588 distinct values and 5,156 unique values. This column also constitutes a foreign key pointing to "dfaccounts.Bank ID", and, given we have 41,815 distinct bank IDs, there seems to be no inconsistency.
- **Account.1:** 576,176 distinct values and 157,062 unique values. It is different from the Account values, which makes sense if Account.1 are the destination accounts. Since in the accounts table we had 712,684 distinct account numbers, there is no evident inconsistency here.
- **Amount Received:** 1,194,921 distinct values and 615,389 unique values. If it was a real dataset, I would look if there is something wrong about having 1/7 of possible variations. But for a synthetic dataset I am accepting some things that would be suspicious in a real dataset.
- **Receiving Currency:** 15 distinct values, 0 unique values.
- **Amount Paid:** 1,204,309 distinct values, 621,441 unique values. Beforehand I suspected Amount Received and Amount Paid might be repeated columns, but if they were, they would have to have the same amount of distinct and unique values. If we combine, though, the tuples (Amount Paid, Payment Currency), (Amount Received, Receiving Currency) we can see there might be a currency conversion if Payment Currency ≠ Receiving Currency. In all those cases, Amount Paid ≠ Amount Received.
- **Payment Currency:** 15 distinct values, 0 unique values.
- **Payment Format:** 7 distinct values, 0 unique values.

---

## 2. Addressing the Cardinality Observations

### 2.1. Accounts Table

Now I am going to work on the attributes in which I identified possible cardinal inconsistencies.

**1) "dfaccounts.Bank ID": 41,815 distinct and 8,447 unique values.** The ID should be a unique number to identify a certain bank and should be repeated every time the same bank is repeated. We can see though, there are 41,815 distinct IDs for 27,652 bank names, so as a first observation, we can see there might be something wrong worth being addressed.

For this, I should first check if everytime a bank name is repeated, the same ID is assigned to that bank. In order to do it, I can group by Bank Names and check its ID to see if in the resulting dataframe I would get, there are rows with the same name and different ID.

I will do this by coding "dfaccounts.groupby("Bank Name")["Bank ID"].nunique()", which will group by bank names and then count how many id numbers each bank name has.

We can see there are 468 over 27,652 banks (468 / 27,652) with more than one ID assigned. Looking closer at "Arbor Bank", for example, I can see it effectively has 12 IDs assigned to the name.

This makes me think that there might be several different banks sharing a common name, differentiated by their ID number and not their name.

I will proceed then to check whether the relation Bank ID → Bank Name is a function. Indeed, if we do have repeated names identifying different banks that are differentiated by different ID numbers, each ID has to correlate with only one name, which would make sense. To do this, I am going to use the same code, but inverting attribute names.

As can be seen, there are no IDs related to more than one bank name, so **I can conclude that Bank Name is not a unique identifier — multiple banks can share a name — while Bank ID uniquely identifies each bank. The relational model will therefore use Bank ID as the bank's key.**

**2) "dfaccounts.Account Number": 712,684 distinct and 712,680 unique values.** We can see a difference between the amount of distinct values (712,684), rows (712,688) and unique values (712,680).

In mathematical terms, if there is a function rows → account numbers, there seems to be a non-injective function, for some rows are related to the same account number.

In order to see which rows are related to more than one account number, I am going to use the Pandas function ".value_counts()" on Account Number to see how many rows each one is related to, and leave just the results that are higher than 1. As a result, I can see four rows are each related to a same account number, which is consistent with the 712,684 distinct and 712,680 unique values.

Now, this is not inherently wrong and more information is needed to determine if there is a problem here. So I am going to select all the rows of the dataframe which have these repeated account numbers to see, essentially, if they have a repeated bank ID. Indeed, as long as those two attributes are not repeated, we can conclude there is just a coincidence in the account number of two different financial entities and that they can be uniquely identified combining these two different attributes.

As can be seen just by looking at those rows, there are no duplicated account numbers within the same bank ID, which means that, effectively, there are repeated account numbers but between different banks. To check computationally if this is right, I used the function "duplicated" to see if there was a duplication within these two essential attributes — and there was none.

Even if I discovered nothing is wrong here, what I have seen has a direct consequence for the relational model: **the account entity's primary key should be the composite Bank ID – Account Number.**

### 2.2. Transactions Table

**Tuples (Amount Paid, Payment Currency), (Amount Received, Receiving Currency):**

- Amount Paid: 1,204,309 distinct values, 621,441 unique values.
- Payment Currency: 15 distinct values, 0 unique values.
- Amount Received: 1,194,921 distinct values and 615,389 unique values.
- Receiving Currency: 15 distinct values, 0 unique values.

If we combine the tuples (Amount Paid, Payment Currency), (Amount Received, Receiving Currency) we can see there possibly is a currency conversion if Payment Currency ≠ Receiving Currency and they do not have the same value, which is highly improbable in reality.

So I will assume there will be a difference in amount paid/received if Payment Currency ≠ Receiving Currency and check specifically if there is a contradictory case.

This analysis shows that these attributes represent, effectively, an amount paid in one currency and its conversion to another currency and, other than 18 observed negligible values (rounding at amounts equal or lower than 0.03, not material to the analysis), the data is consistent with this hypothesis.

> **Relevance to AML:** currency conversion across transactions is relevant to AML analysis, as it is a known layering technique used to obscure the origin of funds.

---

## 3. Primary Key

In this section I will check if attributes that are supposed to be a primary key within each table, that is, a distinctive row identifier, are not repeated, serving their purpose.

### 3.1. Accounts Table Primary Key

The primary key in this table is the tuple (Bank ID, Account Number). Checking whether it is unique in every row, there are no results when we look for repeated values, meaning there are no repeated rows regarding these two attributes.

### 3.2. Transactions Table Primary Key

Here I see there is no natural combination of attributes that can be used to uniquely identify a row. For instance, there could be more than one transaction made within the same minute, with the same account, to the same account, with the same conversion and the same value for the rest of the attributes.

Thus it needs a surrogate key, which has not automatically been created by Pandas when I created the dataframe using "pd.read_csv()". So I will create an explicit transaction ID column to serve as a stable primary key, since the default pandas index is positional and not guaranteed to persist across transformations.

---

## 4. Functional Dependency Within Each Table

Here I will verify if there is coherence between values that are supposed to be coherent throughout the whole dataframe. For example, if ID 314693 maps to "China Bank #2820", then every row with ID 314693 must show that same name. If it turns out there is a different name, then there would be an incoherence between two values in columns that have a functional dependency.

### 4.1. Accounts Table

**Bank ID → Bank Name:** it has already been shown there is a coherent relation between these two columns. Grouping per Bank ID and counting distinct names, no ID returns more than 1, meaning there is no ID related to more than one bank name.

**Entity ID → Entity Name:** every ID is related to one Entity Name, which is what we suspected just by looking at the first distinct and unique values observations. Thus, we can see it is a function and that every time a row has a certain entity ID with a certain entity name, that combination is repeated throughout the whole dataset.

**(Bank ID, Account Number) → Entity ID:** I have already checked the tuple (Bank ID, Account Number) is a valid primary key for "dfaccounts". Thus, as there will be one Entity ID related to every single tuple, it is trivial to check this relation's coherence.

### 4.2. Transactions Table

I do not see any functional dependency inconsistency worth checking at this stage.

---

## 5. Cross-Table Referential Integrity

So we have a parent table I have named "dfaccounts" and a child table I have named "dftransactions". I am going to check now if the expected correspondences between them are correct. I think this is essential because it is part of the basic consistency needed to analyse transactions.

**1) ("dftransactions.From Bank", "dftransactions.Account") → ("dfaccounts.Bank ID", "dfaccounts.Account Number"):**

I should check if every pair (From Bank, Account) in the transactions table is, effectively, a pair in the accounts table, for every transaction is a transaction from an account existing in the accounts table.

Translated to set theory language, I should check here if all the elements made by tuples in the subset B = {("dftransactions.From Bank", "dftransactions.Account")} are included in the set made by tuples A = {("dfaccounts.Bank ID", "dfaccounts.Account Number")}.

There are several options through which I could check this, but the most intuitive to me is through a merge. So I will merge both tables and select just these attributes. To do the merge I will use a left join with "dftransactions" to the left and see if there is an orphan tuple to the right, which would mean that there is a row in "dftransactions" not related to any row in "dfaccounts". There are no orphan tuples.

**2) ("dftransactions.To Bank", "dftransactions.Account.1") → ("dfaccounts.Bank ID", "dfaccounts.Account Number"):** analogous to (1), and also with no orphan tuples.

---

## 6. Conclusion

This phase validated the structural integrity of both tables before modeling. Key findings:

1. Bank Name is not unique — multiple banks share names — so Bank ID is the bank identifier.
2. Account numbers repeat across banks, so the account key is the composite (Bank ID, Account Number).
3. The transactions table has no natural key and requires a surrogate.
4. Both foreign-key relationships (sender and receiver) into the accounts table hold zero orphan references.

The data is structurally sound and ready for normalization and typology analysis, which is the next phase.
