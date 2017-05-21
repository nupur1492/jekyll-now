---
layout: post
title: Implementing Apriori Algorithm in R
---

There are a bunch of blogs out there posted that show how to implement apriori algorithm in R. However, when I was working on the same, I hit a roadblock since the data was neither in single format, nor in basket(Step 2 explains what a basket format is). I spent quite some time converting the data into the required format to be able to find the association rules.   
So, here goes…

### Step 1: Read the data

Read the ‘Groceries_dataset’ csv file. [Here is a link] (https://github.com/nupur1492/RProjects/tree/master/MarketBasketAnalysis) to the csv file.

```r
df_groceries <- read.csv("Groceries_dataset.csv")
```
The data consists of three columns:   
Member_number: An ID that can help distinguish different purchases by different customers.   
Date: The date of transaction . 
ItemDescription: The description of the actual item that was bought.   

### Step 2: Data cleaning and manipulations using R

The data required for Apriori must be in the following basket format:

| TID | Items |
| --- | ----- |
| 100 | ACD |
| 200 | BCE |
| 300 | ABCE |
| 400 | BE |

The basket format must have first column as a unique identifier of each transaction, something like a unique receipt number. The second columns consists of the items bought in that transaction, separated by spaces or commas or some other separator.   

However, the data we have is something like this:  

| Member number | Date | Item description |
| ------------- | ---- | ---------------- |
| 1688122020199	| 12/26/2014	| Citrus fruit |
| 1688122020199 | 10/05/2011	| Whole milk |
| 1688122020199 |10/05/2011	| chocolates|
| 1618090368299	| 03/29/2011	| dishes |

Since the structure of the data is not in the format necessary to find association rules, we have to perform some data manipulations before finding the relationships.  

Lets first make sure that the Member numbers are of numeric data type and then sort the dataframe based on the Member_number.  
```r
df_sorted <- df_groceries[order(df_groceries$Member_number),]  
df_sorted$Member_number <- as.numeric(df_sorted$Member_number)  
```

Learn more about [vectors, matrices and data frames](https://datascienceplus.com/how-to-create-data-frames-in-r/) in R, or check those [videos](https://datascienceplus.com/learn-r-from-scratch-part-1/).

Now, we have to convert the dataframe into transactions format such that we have all the items bought at the same time in one row. For this, we use a function called ddply, offered by package plyr.

```r
install.packages(“plyr”, dependencies= TRUE)
```

Make sure that you do not have package ‘dplyr’ attached to the session. You might end up getting something like this:

```
You have loaded plyr after dplyr - this is likely to cause problems.
If you need functions from both plyr and dplyr, please load plyr first, then dplyr:
library(plyr); library(dplyr)
```

Hence, detach dplyr package first and then load the package

```
if(sessionInfo()['basePkgs']=="dplyr" | sessionInfo()['otherPkgs']=="dplyr"){
  detach(package:dplyr, unload=TRUE)
}
library(plyr)
```

The next step is to actually convert the dataframe into basket format, based on the Member_number and Date of transaction     

```r
df_itemList <- ddply(df_groceries,c("Member_number","Date"), 
                       function(df1)paste(df1$itemDescription, 
                       collapse = ","))
```

The above function ddply() checks the date and member number and pivots the item descriptions with same date and same member number in one line, separated by commas.     

Something like this:    

| Member number | Date | Item description |
| ------------- | ---- | ---------------- |
| 1688122020199	| 12/26/2014	| Citrus fruit |
| 1688122020199 | 10/05/2011	| Whole milk |
| 1688122020199 |10/05/2011	| chocolates|
| 1618090368299	| 03/29/2011	| dishes |

becomes:

| Member number | Date | Item description |
| ------------- | ---- | ---------------- |
| 1688122020199	| 12/26/2014	| Citrus fruit |
| 1688122020199 | 10/05/2011	| Whole milk, chocolates |
| 1618090368299	| 03/29/2011	| dishes |

Notice how member number 1688122020199 bought Whole milk and dishes on the same date; which means they were bought together.    Thus we group them together in one row, separated by commas.   

Thus, we now have the data in the necessary basket format. We can now implement Apriori on this data. The ddply function works pretty well even with larger datasets, I have tried it with a million rows and it takes only a few minutes to pivot the table.  
Once we have the transactions, we no longer need the date and member numbers in our analysis. Go ahead and delete those columns.   

```r
df_itemList$Member_number <- NULL
df_itemList$Date <- NULL

#Rename column headers for ease of use
colnames(df_itemList) <- c("itemList")
```

Write the resulting table to a csv file. The reason we do this is, when we write a dataframe to a .csv file, it attaches a row number by default. (unless, of course you were to explicitly tell it not to, by using the argument “row.names=FALSE” in the write.csv function).   
We can simply use these row numbers as transaction IDs, as they would be unique to each transaction. Convenient?   

Write dataframe to a csv file using write.csv() .  
```r
write.csv(df_itemList,"ItemList.csv", qoute = FALSE, row.names = TRUE)
```

### Step 3: Find the association rules

Read the csv file u just saved and you will automatically get the transaction IDs in the dataframe   
Run algorithm on ItemList.csv to find relationships among the items. Apriori find these relations based on the frequency of items bought together.    

For implementation in R, there is a package called ‘arules’ available that provides functions to read the transactions and find association rules.   

So, install and load the package:   
```r
install.packages(“arules”, dependencies=”TRUE”)
library(arules)
```

Using the read.transactions() functions, we can read the file ItemList.csv and convert it to a transaction format   

```r
txn = read.transactions(file="ItemList.csv", rm.duplicates= TRUE, format="basket",sep=",",cols=1);
```

Parameters: Transaction file: ItemList.csv   
rm.duplicates : to make sure that we have no duplicate transaction entried   
format : basket (row 1: transaction ids, row 2: list of items)    
sep: separator between items, in this case commas   
cols : column number of transaction IDs   

Quotes are introduced in transactions, which are unnecessary and result in some incorrect results. So, we must get rid of them:  

```r
txn@itemInfo$labels <- gsub("\"","",txn@itemInfo$labels)
```

Finally, run the apriori algorithm on the transactions by specifying minimum values for support and confidence.   
```r
basket_rules <- apriori(txn,parameter = list(sup = 0.01, conf = 0.5,target="rules"));
```

Print the association rules. To print the association rules, we use a function called inspect(). However, if you have package ‘tm’ attached in the session, it creates a conflict with the arules package. Thus, we need to check and detach the package.

```r
if(sessionInfo()['basePkgs']=="tm" | sessionInfo()['otherPkgs']=="tm"){
    detach(package:tm, unload=TRUE)
  }

inspect(basket_rules)

#Alternative to inspect() is to convert rules to a dataframe and then use View()
df_basket <- as(basket_rules,"data.frame")
View(df_basket)
```
Plot a few graphs that can help you visualize the rules. Install and load the ‘arulesViz’ library for association rules specific visualizations:   
```r
library(arulesViz)
plot(basket_rules)
plot(basket_rules, method = "grouped", control = list(k = 5))
plot(basket_rules, method="graph", control=list(type="items"))
plot(basket_rules, method="paracoord",  control=list(alpha=.5, reorder=TRUE))
plot(basket_rules,measure=c("support","lift"),shading="confidence",interactive=T)
```
Graph to display top 5 items
```r
itemFrequencyPlot(txn, topN = 5)
```
Thats’s all Folks! I hope it was simple to understand and implement. I also have my code on [github](https://github.com/nupur1492/RProjects/tree/master/MarketBasketAnalysis) if you dont want to type everything.   
A special thanks to [this blogpost](https://www.r-bloggers.com/association-rule-learning-and-the-apriori-algorithm/), where I first learned the basics of implementing apriori in R. Also, this is my first attempt at writing a blog. Please feel free to reach out if you have any suggestions and comments.!   
Thank you.   
<style>
table{
    border-collapse: collapse;
    border-spacing: 0;
    border:2px solid #ff0000;
}

th{
    border:2px solid #000000;
}

td{
    border:1px solid #000000;
}
</style>
<table>
<colgroup>
<col width="30%" />
<col width="70%" />
</colgroup>
<thead>
<tr class="header">
<th>Field</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td markdown="span">First column **fields**</td>
<td markdown="span">Some descriptive text. This is a markdown link to [Google](http://google.com). Or see [some link][mydoc_tags].</td>
</tr>
<tr>
<td markdown="span">Second column **fields**</td>
<td markdown="span">Some more descriptive text.
</td>
</tr>
</tbody>
</table>


