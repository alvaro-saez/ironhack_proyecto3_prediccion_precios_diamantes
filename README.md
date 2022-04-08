# ih_datamadpt1121_project_m3
This Kaggle competition of Ironhack consists of predicting the price of a data set of Diamonds. 

For it, we have to understand the different data sets, prepare and clean the data and find the best model that targets the lowest deviation from the predicted price.

## **STEPS:**
<ol>
  <li>Importing Libraries</li>
  
  <li>Load the Data available</li>

  <li>Inspect it</li>

  <li>Exploratory Data Analysis (EDA)</li>

  <li>Create the train and the test dataframe</li>

  <li>Test different models</li>

  <li>Optimized the hyperparameters of the model chosen</li>

  <li>Make the prediction</li>

  <li>Evaluate the error of the prediction</li>
</ol>

## **1 - Importing Libraries:**
```
import pandas as pd
import numpy as np
from sqlalchemy import create_engine
import plotly.express as px
import seaborn as sns

from sklearn.model_selection import RandomizedSearchCV
from sklearn.ensemble import RandomForestRegressor

from sklearn.model_selection import train_test_split
from sklearn.model_selection import cross_val_score
from sklearn.metrics import mean_squared_error
```

## **2 - Load the Data available:**

We have two different data sources:

  **A ) DATABASE FILE:** “diamonds_train.db”, which contains 40455 entries to train the model.  

¿How to read a “db file”?. We will use SQLite, a relational database management system, and SQL to read it. 
```
 def conection(database):
    connection = create_engine("sqlite:///"+ database).connect()
    return connection
con = conection("../files/diamonds_train.db")
```

```
def query_diamond(query):
    df_diamonds_full_train = pd.read_sql_query(query,con)
    return df_diamonds_full_train

query_diamonds_full_train = '''
SELECT 
    d_prop.index_id,
    
    d_tran.carat,
    d_cut.cut,
    d_col.color,
    d_clar.clarity,

    d_dim.depth,
    d_dim.'table',
    d_dim.x,
    d_dim.y,
    d_dim.z,
    
    d_city.city,

    d_tran.price
    
FROM diamonds_properties as d_prop
    INNER JOIN diamonds_clarity as d_clar ON d_prop.clarity_id = d_clar.clarity_id
    INNER JOIN diamonds_color as d_col ON d_prop.color_id = d_col.color_id
    INNER JOIN diamonds_cut as d_cut ON d_prop.cut_id = d_cut.cut_id
    INNER JOIN diamonds_dimensions as d_dim ON d_prop.index_id = d_dim.index_id
    INNER JOIN diamonds_transactional as d_tran ON d_prop.index_id = d_tran.index_id
    INNER JOIN diamonds_city as d_city ON d_tran.city_id = d_city.city_id
'''

df_diamonds_train = query_diamond(query_diamonds_full_train)
```

**B ) CSV FILE:** “diamonds_test.csv”, which contains 13485 entries to make the prediction:

¿How to read a CSV file?. With a Pandas method (pd.read_csv())
```
def import_csv_diamonds(location):
    df_diamonds_test = pd.read_csv(location)
    return df_diamonds_test

df_diamonds_test = import_csv_diamonds("../files/diamonds_test.csv")
```

#### FEATURES TO STUDY:
- price in US dollars ($326--$18,823). This will be our the target column. It is only in the tarin dataframe.

**The 4 Cs of Diamonds:**

- carat (0.2--5.01) The carat is the weight of the diamond.  One carat equals 1/5 gram and is subdivided into 100 points. Carat weight is the most important feature of the 4Cs to predict the price. 

- cut (Fair, Good, Very Good, Premium, Ideal) It is the quality of the cut. A well-cut diamond will direct more light through the crown. A diamond with a depth that's too shallow or too deep will allow light to escape through the sides or the bottom of the stone.

- color, from J (worst) to D (best). The diamond’s color ranges from an icy white colorless to a light yellow. Colorless is the most rare and therefore the most expensive. Yellow is the least rare and therefore the least expensive.

- clarity (I1 (worst), SI2, SI1, VS2, VS1, VVS2, VVS1, IF (best)) The clarity refers to the diamond's tiny markings. Flawless (IF) is the most rare and therefore the most expensive. Included (I) is the least rare and therefore the least expensive.

**Dimensions:**

- x length in mm (0--10.74)

- y width in mm (0--58.9)

- z depth in mm (0--31.8)

**Other Features:**
- depth: total depth percentage = z / mean(x, y) = 2 * z / (x + y) (43--79)

- table: width of top of diamond relative to widest point (43--95)

- city: city where the diamonds is reported to be sold.

## **3 - Inspect our data**

REMEMBER OUR GOAL --> To predict the price, so we are going to study how the different features are correlated with the price. I have use:
- TABLEAU
- SEABORN.

**A) Correlation coeficient:**
```
def correlation_diamonds(df_diamonds_enriched):
    df_diamonds_corr = df_diamonds_enriched.corr().reset_index()
    return df_diamonds_corr
df_diamonds_corr = correlation_diamonds(df_diamonds_train)
```

```
def correlation_visualization(df_diamonds_enriched):
    df_diamonds_corr2 = df_diamonds_enriched.corr()
    sns.set(rc={'figure.figsize':(12,8)})
    return sns.heatmap(df_diamonds_corr2, annot=True);
corr_graf = correlation_visualization(df_diamonds_corr)
```
<p align="center"><img src="https://github.com/alvaro-saez/ih_datamadpt1121_project_m3/blob/main/images/CORRELATION.png"></p>

So we see a clearly positive correlation between carat, x, y, z with the price. Let´s go to see it with Tableau:
<p align="center"><img src="https://github.com/alvaro-saez/ih_datamadpt1121_project_m3/blob/main/images/carat.png"></p>
<p align="center"><img src="https://github.com/alvaro-saez/ih_datamadpt1121_project_m3/blob/main/images/x.png"></p>
<p align="center"><img src="https://github.com/alvaro-saez/ih_datamadpt1121_project_m3/blob/main/images/y.png"></p>
<p align="center"><img src="https://github.com/alvaro-saez/ih_datamadpt1121_project_m3/blob/main/images/z.png"></p>
<p align="center"><img src="https://github.com/alvaro-saez/ih_datamadpt1121_project_m3/blob/main/images/depth.png"></p>
<p align="center"><img src="https://github.com/alvaro-saez/ih_datamadpt1121_project_m3/blob/main/images/table.png"></p>

We are going to use this information to ponder those features giving more importance to carat, x, y and z.

## **4 - Exploratory Data Analysis (EDA)**

We need to prepare our dataframes to fit it to the best model.

**a) Missing and cero values:** The is no any nulls, but we have different raws with "0" in "x", "y" and "z" columns. There are different ways to see it, but we use the next code (we have to reapeat it for each column):

```
df_diamonds_train[df_diamonds_train["x"]<1]
```

For our case, we are going to drop the "Z" column (making different tests it decrease the RMSE oof our predictions).

¿How to deal with the cero values? There are different techniques. We are going to use the cero values in our prediction beacause I have done many tests and the one with the lowest RMSE was using "0" values, but the next picture is the best way for this data frame to deal with those cero values:
<p align="center"><img src="https://github.com/alvaro-saez/ih_datamadpt1121_project_m3/blob/main/images/mean.png"></p>

**b) Drop columns**
**AN ADVISE!**
in a EDA process is very common to drop columns, so you can create a function to save your time. Here you have an example:
```
def delete_column(df, column_name):
    del df[column_name]
    return "column deleted"
```

I have delete the next columns:
 - Z
 - index id
 - id

**c) Remove outliers** In this case, all the raws are usefull for our model, but here you have a function if you want to remove outliers of a normal distribution.
```
def remove_outlier(df_in, col_name):
    q1 = df_in[col_name].quantile(0.30)
    q3 = df_in[col_name].quantile(0.70)
    iqr = q3-q1 #Interquartile range
    fence_low  = q1-1.5*iqr
    fence_high = q3+1.5*iqr
    df_out = df_in.loc[(df_in[col_name] > fence_low) & (df_in[col_name] < fence_high)]
   return df_out
```
**d) Create new columns**: It could be counterproductive, because it is a good practise to remove columns highly correlated (Feature selection"), but for our model (you will see it later) will be usefull.
 - one new feature multiplying "x" and "y"
 - another feature dividing depth and table

**d) Encoding the categorical features**:  I have test the next methods:

- One hot encoding (discarded because of the model peculiarity): using the pd.get_dummmies() method of Pandas

- Label encoding (which give us a better core in our model)
```
X_train_cat[["cut","color","clarity","city"]] = X_train_cat[["cut","color","clarity","city"]].astype("category")

X_train_cat["cut"] = X_train_cat["cut"].cat.codes
X_train_cat["color"] = X_train_cat["color"].cat.codes
X_train_cat["clarity"] = X_train_cat["clarity"].cat.codes
X_train_cat["city"] = X_train_cat["city"].cat.codes
```
