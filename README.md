![customer-segmentation](https://github.com/user-attachments/assets/f8e029f3-b75e-44d0-ba19-8f723292f8fe)
# [Python] RFM Customer Segmentation Analysis

## Project Overview

This project applies RFM (Recency, Frequency, Monetary) analysis to segment customers based on purchasing behavior.
The goal is to identify high-value customers, at-risk customers, and growth opportunities, then translate insights into actionable marketing strategies.

This analysis is commonly used in CRM, retention marketing, and customer analytics.

## Objectives

* Segment customers using RFM scoring

* Identify key customer groups (Champions, At Risk, Lost, Potential Loyalists, etc.)

* Understand behavioral patterns across segments

* Propose data-driven marketing actions to improve retention and revenue

## Methodology

* Recency: How recently a customer made a purchase

* Frequency: How often they purchased

* Monetary: How much they spent

### Steps:

* EDA
```
<class 'pandas.core.frame.DataFrame'>
RangeIndex: 541909 entries, 0 to 541908
Data columns (total 8 columns):
 #   Column       Non-Null Count   Dtype         
---  ------       --------------   -----         
 0   InvoiceNo    541909 non-null  object        
 1   StockCode    541909 non-null  object        
 2   Description  540455 non-null  object        
 3   Quantity     541909 non-null  int64         
 4   InvoiceDate  541909 non-null  datetime64[ns]
 5   UnitPrice    541909 non-null  float64       
 6   CustomerID   406829 non-null  float64       
 7   Country      541909 non-null  object        
dtypes: datetime64[ns](1), float64(2), int64(1), object(4)
```

* Data Wrangling

```
df['CustomerID']= df['CustomerID'].dropna()
df['CustomerID']= df['CustomerID'].astype('Int64')
df['InvoiceDate']= pd.to_datetime(df['InvoiceDate']).dt.date
df
```
```
df = df[~df['InvoiceNo'].astype(str).str.startswith('C')]
df = df[df['Quantity'] > 0]
df = df[df['UnitPrice'] > 0]
df
```


* Calculate R, F, M metrics for each customer under R-score, F-score, M-score
  *   M-score = [Qanitity] * [Unit Price] , grouped by Customer
  *   F-score = distinct count [InvoiceNo] , grouped by Customer
```
df['TotalPrice']= df['Quantity'] * df['UnitPrice']
df_rfm= df.groupby('CustomerID').agg({'InvoiceNo': 'nunique', 'TotalPrice': 'sum','InvoiceDate':'max'}).reset_index()
df_rfm.columns = ['CustomerID', 'Frequency', 'Monetary','MaxInvoiceDate']
df_rfm
```
  * R-score
      * Extract last order date [MaxInvoiceDate] of every customer
      * [Max Date of the data (2011-12-31)] - [MaxInvoiceDate]
```
df_rfm['MaxInvoiceDate'] = pd.to_datetime(df_rfm['MaxInvoiceDate'], errors='coerce')
snapshot_date= pd.to_datetime('2011-12-31')
df_rfm['Recency'] = (snapshot_date - df_rfm['MaxInvoiceDate']).dt.days
del df_rfm['MaxInvoiceDate']
df_rfm
```
* Assign scores (1–5) using quantiles

```
df_rfm['F_score']= pd.qcut(df_rfm['Frequency'].rank(method='first'), q=5, labels=[1,2,3,4,5]).astype(int)
df_rfm['M_score']= pd.qcut(df_rfm['Monetary'].rank(method='first'), q=5, labels=[1,2,3,4,5]).astype(int)
df_rfm['R_score']= pd.qcut(df_rfm['Recency'].rank(method='first'), q=5, labels=[5,4,3,2,1]).astype(int)
df_rfm
```

* Combine scores into RFM segments

```
segmentation_dict = {
    "Champions": ["555","554","544","545","454","455","445"],
    "Loyal": ["543","444","435","355","354","345","344","335"],
    "Potential Loyalist": [
        "553","551","552","541","542","533","532","531",
        "452","451","442","441","431","453","433","432",
        "423","353","352","351","342","341","333","323"],
    "New Customers": ["512","511","422","421","412","411","311"],
    "Promising": [
        "525","524","523","522","521","515","514","513",
        "425","424","413","414","415","315","314","313"],
    "Need Attention": ["535","534","443","434","343","334","325","324"],
    "About To Sleep": ["331","321","312","221","213","231","241","251"],
    "At Risk": [
        "255","254","245","244","253","252","243","242",
        "235","234","225","224","153","152","145","143",
        "142","135","134","133","125","124"],
    "Cannot Lose Them": ["155","154","144","214","215","115","114","113"],
    "Hibernating customers": ["332","322","233","232","223","222","132","123","122","212","211"],
    "Lost customers": ["111","112","121","131","141","151"]}

segment_lookup = {segment: name
    for name, segments in segmentation_dict.items()
    for segment in segments}

df_rfm['Segment'] = df_rfm['RFM_score'].map(segment_lookup)

```

* Analyze segment distribution and business impact

## Results

<img width="630" height="470" alt="mtv-rfm-1" src="https://github.com/user-attachments/assets/90c31969-582a-4be2-b7fd-3a85c521941b" />

<img width="1057" height="547" alt="mtv-rfm-2" src="https://github.com/user-attachments/assets/2c337811-1e47-47a8-bc3e-fdd5772754f5" />

<img width="1057" height="547" alt="mtv-rfm-3" src="https://github.com/user-attachments/assets/208fa2ad-1bb9-4022-aaad-4fd105c1fdc4" />

<img width="1057" height="547" alt="mtv-rfm-4" src="https://github.com/user-attachments/assets/7a5ef1ea-c5d3-4e5c-b6da-80985e88d461" />

## Analysis

1. Champions = R = 4–5, F = 4–5, M = 4–5.
* These customers are concentrated at high scores → They are the profit engine.

3. Hibernating & Lost Customers are concentrated at R = 1–2
* Low recency = they haven’t returned for a long time.
* Lost Customers: heavily concentrated at R = 1 (490), with low M and low F.
* Hibernating: R = 2–3, but absent from high M groups.
→ This is a dormant segment: not purchasing, with a high risk of churning permanently.

3. At Risk customers are at R = 1–2 but have relatively high F and M
* They used to purchase frequently.
* They spent a meaningful amount.
* But they haven’t returned recently.
→ Losing them means revenue loss.

4. Potential Loyalists are spread across R = 3–4 with medium M
→ This group is ready to convert into Loyal Customers or Champions.

5. New Customers (R = 4–5 but low F & M)
→ The store is successfully acquiring customers, but has not yet developed them enough to increase LTV.

## CTA
1. Focus on Loyal and Champions because these types are ready to spend on the company
* Cross-sell, upsell
* Bundle discount
* Early access new products

2. Winback At risk group
* Reminder campaigns (email, SMS)
* Personal discount based on last category purchased
* “We miss you” offers
* Extended warranty / free shipping to push a comeback purchase

3. Attract Potential Loyalist
* Promote membership program
* Lower-bar offers ("Buy again, get 10%")
