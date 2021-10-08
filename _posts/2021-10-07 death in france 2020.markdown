---
layout: post
title: Death in France in 2020
date: 20221-10-07 13:32:20 +0300
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: age_pyramid.png # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [Data analysis, Personal project, Public data]
---
## Deaths in France in 2020

* The [National Institute of Statistics and Economic Studies](https://www.insee.fr/en/accueil) 
releases public available data on deaths in France every year. It is a popular datafile to work with and there are many projects using and tools available to explore the file at [data.gouv.fr](https://www.data.gouv.fr/fr/datasets/fichier-des-personnes-decedees/)

* I tried my hand at organizing the data and use a few vizualization techniques I will expose here.

The raw file lacks for most of the entries a country of birth and I filled it accordingly to the birth location matching how the coding for it evolved over time. 
The process can be replicated after collecting the required csv files available at 
[Insee files](https://www.insee.fr/fr/information/2560452) using [clean_INSEE_csv.py.py](https://github.com/GreLeBr/deces_2020/blob/master/deces_2020/clean_INSEE_csv.py)


### Age pyramid of deaths in France

* Deaths are categorized in group spanning a range of 5 years by using a bins variable and cutting the 
dataframe accordingly. Age of the person deceased is calculated by substrating day of death to day of birth, the results is left as years for easier classfication. The dataframe is further groupby using the sex variable.  

![Age pyramid](age_pyramid.png)


```python
# Building categories for age group
bins = pd.IntervalIndex.from_tuples([(0, 5), (5, 10), (10, 15), (15,20),(20,25),
 (25,30), (30,35), (35,40), (40,45), (45,50), (50, 55),(55, 60), (60,65),
  (65,70), (70,75), (75, 80), (80,85), (85, 90), (90, 95), (95,100),
   (105,110), (110,115) ])
# Starting by making temp columns
df["birthdate1"]=df["datenaiss"].astype("str")
df["deathdate1"]=df["datedeces"].astype("str")
# Assigning "NaN" to missing dates
df["birthdate1"]=df["birthdate1"].apply(lambda x: np.nan if "00" in
 x[-2:] or "00" in x[-4:-2] or "0000" in x[:-4] or x=="0" else x)
df["deathdate1"]=df["deathdate1"].apply(lambda x: np.nan if "00" in
 x[-2:] or "00" in x[-4:-2] or "0000" in x[:-4] or x=="0" else x)
# Dropping missing values
df.dropna(inplace=True)
# Dropping temp columns
df.drop(columns=["deathdate1", "birthdate1"], inplace=True)
# Converting date str to date values
df["birthdate"]=pd.to_datetime(df["datenaiss"],  format='%Y%m%d')
df["deathdate"]=pd.to_datetime(df["datedeces"],  format='%Y%m%d')
# Assigning age group to rows
df['Range']=df.groupby("sexe", as_index=False)[["lifespan"]]
.transform(lambda x: pd.cut(x, bins) )

# Grouping values by sex
data=df.groupby(["sexe", "Range"], as_index=False)[["lifespan"]].count().copy()

# Inverting data for woman to plot them alongside male data 
data["lifespan"][22:]=data["lifespan"][22:].apply(lambda x: -x)

# Drawing figure

plt.figure(figsize=(20,15))
plt.xlim((-90000, 90000))

# Creating a line separating left from right on 0
plt.axvline(x=0, linestyle='--', color="black")

# Plotting men data, the categories are reverted to display age range from bottom to top
bar_plot = sns.barplot(x='lifespan', y='Range', data=data[22:],  order=data["Range"][21::-1])

# To add a label on each row the text is placed accordingly to the value of the data with a bit of 
# an offset

for ytick in bar_plot.get_yticks():
    bar_plot.text(data["lifespan"][21-ytick] + 100+ data["lifespan"][21-ytick]* 0.05,
    ytick,  data["lifespan"][21-ytick], verticalalignment="center",  fontsize=14 )

# Plotting woman data with similar techniques 
bar_plot = sns.barplot(x='lifespan', y='Range', data=data[:22],  order=data["Range"][21::-1])
for ytick in bar_plot.get_yticks():
    bar_plot.text(data["lifespan"][43-ytick] -7000 + data["lifespan"][43-ytick]* 0.05, ytick,
    -data["lifespan"][43-ytick], verticalalignment="center", ha="left", fontsize=14)  

bar_plot.tick_params( axis='x',          # changes apply to the x-axis
    which='both',      # both major and minor ticks are affected
    bottom=False,      # ticks along the bottom edge are off
    top=False,         # ticks along the top edge are off
    labelbottom=False) # labels along the bottom edge are off

bar_plot.set_xlabel(xlabel="Number of deaths in 2020",fontsize=18 )
bar_plot.set_ylabel(ylabel="Age ranges",fontsize=18 )

# adding boxes for description
props = dict(boxstyle='round', facecolor='yellow', alpha=0.5)

bar_plot.text(50000, 20, "Men", fontsize=25,bbox=props )
bar_plot.text(-50000, 20, "Women", fontsize=25, bbox=props )

``` 

### What is the average age of people dying according to their first or last name?

Following popular blog [Le prénom : catégorie sociale](http://coulmont.com/bac/index.html) ,
I plotted first_name by age of deaths as well as last_name by age of death.

HOWEVER !!! These vizualisations need to be taken with a big grain of salt as first_names are not all equivalent. Some names were popular in the early 20th century while other are more recents and it has a direct effect on the average age of death associated with a first_name. 
I will describe this issue in more details further down the page. 

Using first name:
{% include prenom_france.html %}
<!-- [First_name](https://chart-studio.plotly.com/~GreLeBr/13) -->


For people born abroad: 
{% include prenom_horsfrance.html %}
<!-- [First_name_abroad](https://chart-studio.plotly.com/~GreLeBr/3) -->

Using last name:
{% include nom.html %}
<!-- [Last_name](https://chart-studio.plotly.com/~GreLeBr/11) -->

and included the gender with the last name:
{% include nom_sex.html %}


Code is as follow:

```python
# Separating first and last names
df["nomprenomsplit"]=df["nomprenom"].apply(lambda x: x.split("*"))
df["nom"]=df["nomprenomsplit"].apply(lambda x: x[0])
df["prenoms"]=df["nomprenomsplit"].apply(lambda x: x[1])
df["prenom_split"]=df["prenoms"].apply(lambda x: x.split(" "))
df["first_name"]=df["prenom_split"].apply(lambda x: x[0])
df["first_name"]=df["first_name"].apply(lambda x: x.replace("/", ""))

# Grouping data according to first name, gender and country of birth
data4=df.groupby(['sexe',"first_name", "paysnaiss"], as_index=False)[["sexe", "lifespan"]]
.agg({"sexe": "mean", "lifespan": ["mean", "std", "count"]})

# Joining the multi layered columns
data4.columns = ['_'.join(col) for col in data4.columns]

# Droping the index
data4.reset_index(inplace=True)

# Renaming columns
data4.rename(columns={"first_name_":"prenom", "paysnaiss_": "paysnaiss","sexe_mean":"sex", "lifespan_mean":"Average_Age", "lifespan_std":"Age_Std", "lifespan_count":"count"}, inplace=True)

# Taking a subset of the first 500 results according to count and average age
data5=data4.sort_values(["count","Average_Age"], ascending=False).head(500)

# Transforming the sex column as string to only have two distinct categories in plotly
data5["sex"]=data5["sex"].astype("str")

# Ploting plotly figure
fig=px.scatter(data5, x="Average_Age", y="count", log_y=True, color="sex",size="count", size_max=75,  hover_data={"prenom":True, "Average_Age":':.2f', "Age_Std":':.2f'})  
fig.show()

# for people born abroad
fig=px.scatter(data5[data5["paysnaiss"]!="FRANCE"], x="Average_Age", y="count", log_y=True, color="sex",size="count", hover_data={"prenom":True, "Average_Age":':.2f', "Age_Std":':.2f', "paysnaiss":True})  
fig.show()

# Similarly grouping by name
data3=df.groupby(['sexe',"nom",  "paysnaiss"] , as_index=False)[["sexe", "lifespan"]].agg ({"sexe": "mean", "lifespan": ["mean", "std", "count"]})
data3.columns = ['_'.join(col) for col in data3.columns]
data3.reset_index(inplace=True)
data3.drop(columns="index", inplace=True)
data3.rename(columns={"nom_":"nom", "paysnaiss_": "paysnaiss","sexe_mean":"sex", "lifespan_mean":"Average_Age", "lifespan_std":"Age_Std", "lifespan_count":"count"}, inplace=True)
data2=data3.sort_values(["count","Average_Age"], ascending=False).head(500)
data2["sex"]=data2["sex"].astype("str")
fig=px.scatter(data2, x="Average_Age", y="count", log_y=True, color="sex",size="count", hover_data=[data2["nom"]])
fig.show()

# Not including gender
no_sex=df.groupby(["nom",  "paysnaiss"] , as_index=False)[["lifespan"]].agg ( ["mean", "std", "count"])
no_sex.columns = ['_'.join(col) for col in no_sex.columns]
no_sex.reset_index(inplace=True)
no_sex.rename(columns={ "lifespan_mean":"Average_Age", "lifespan_std":"Age_Std", "lifespan_count":"count"}, inplace=True)
no_sex_500=no_sex.sort_values(["count","Average_Age"], ascending=False).head(500)
fig=px.scatter(no_sex_500, x="Average_Age", y="count", log_y=True, size="count", hover_data=[no_sex_500["nom"]])
fig.show()
```

### How old is a person according to their name

As mentioned above not all first name are equal and some were more popular in the early 1900s and are not that popular anymore, some are more recents and others are experiencing a revival. 

* A website to visualize this is available here [Predict age by name](https://www.ekintzler.com/projects/age-prediction/) . 

* I will recreate the process in python. You can play with it using my mini heroku app 
-> [prenom_age](https://prenomage2020.herokuapp.com/)

# Method:

INSEE makes available the number of people born each year since 1900 according to their first name , INSEE also keep track of age expectancy every year. 
[First names in France](https://www.insee.fr/fr/statistiques/2540004?sommaire=4767262)
[Mortality tables](https://www.insee.fr/fr/statistiques/3311422?sommaire=3311425#consulter-sommaire)

* The mortality tables only calculate life expectancy up to 105 years while the first name database does not report deaths and would technically keep track of people up to 121 years old. For simplification purposes, I assumed that from 105 years and over the percentage of people alive would be the same although it is much more likely to be decreasing drastically. 

To load the data I used the following code: 

``` python
# Reading the csvs
df_naissance=pd.read_csv("../raw_data/nat2020.csv", sep=";")
mortality=pd.read_csv("../raw_data/mortality.csv")
# Cleaning the mortality file and calculating percentage of people alive per year of birth
mortality["pop"]=[mortality["pop"][i].replace(" ", "") for i in range (mortality.shape[0])]
mortality["pop"]=mortality["pop"].astype("int64")
mortality["deaths"]=[mortality["deaths"][i].replace(" ", "") for i in range (mortality.shape[0])]
mortality["deaths"]=mortality["deaths"].astype("int64")
mortality["alive_percent"]=(mortality["pop"])/1000
# cleaning the original csv and calculating the virtual age
df_naissance.annais.replace("XXXX", np.nan, inplace=True)
df_naissance.preusuel.replace("_PRENOMS_RARES", np.nan, inplace=True)
df_naissance.dropna(inplace=True)
df_naissance["annais"]=df_naissance["annais"].astype("int64")
df_naissance["age_virtuel"]=2021-df_naissance["annais"]
```
* In order to calculate the expected number of people alive depending on their year of birth and the mortality tables, I used the following function: 

``` python
def funcA(d, nombre):
    if d >=105:
      return nombre*0.246/100
    return nombre*mortality.loc[d, "alive_percent"]/100
def funcB(d, nombre):
    if d >=105:
      return nombre*0.916/100
    return nombre*mortality.loc[d+104, "alive_percent"]/100

df_naissance['nombre_weight'] = df_naissance.apply(lambda x: funcA(x['age_virtuel'],x["nombre"]) if x['sexe'] == 1 else funcB(x['age_virtuel'], x["nombre"]), axis=1)
df_naissance["age_multi"]=df_naissance["age_virtuel"]*df_naissance["nombre_weight"]

```

And this lead for example with the first name Grégoire to display the average age:

``` python
plt.figure(figsize=(50,10))
plt.xticks(rotation='vertical')
sns.barplot(data=df_naissance[(df_naissance["preusuel"]=="GRÉGOIRE") & (df_naissance["sexe"]==1) ], x="age_virtuel", y="nombre_weight" ,dodge=False)
```
![Gregoire](gregoire.png)


* It is possible to estimate the average age of someone according to their name as well as the number of person who would be expected to die this year according to how many were born since 1900 but the approximation is a bit off.  

Using the following code: 

``` python
import unidecode
# Calculating the difference in number of people if it was the next year
df_naissance['nombre_weight_plus1'] = df_naissance.apply(lambda x: funcA(x['age_virtuel']+1,x["nombre"]) if x['sexe'] == 1 else funcB(x['age_virtuel']+1, x["nombre"]), axis=1)

# grouping data by first name
estimation=df_naissance.groupby(["preusuel", "sexe"], as_index=False).agg({"sexe":"mean", "annais":"mean","nombre":"sum","age_virtuel":["mean","std"], "nombre_weight": "sum", "age_multi":["sum","mean","std"],"nombre_weight_plus1":"sum",
 "nombre_weight_minus1":"sum","expected_adjusted_number":"sum"  }) 
estimation.columns = ['_'.join(col) for col in estimation.columns]
estimation.reset_index(inplace=True)

# Dropping accentuated and special characters to match the other csv entries
estimation["prenom"]=estimation["preusuel_"].apply(lambda x: unidecode.unidecode(x))
estimation["true_age_mean"]=estimation["age_multi_sum"]/estimation["nombre_weight_sum"]
estimation["expected_number_deaths"]=round((estimation["nombre_weight_sum"]-estimation["nombre_weight_plus1_sum"]), 0)
estimation2=estimation.sort_values("nombre_sum", ascending=False)
estimation2.drop_duplicates(subset="prenom", keep="first", inplace=True)
# Making dictionaries to match first name to estimated age and estimated deaths
prenom_dict={k:v for k,v in zip(test["prenom"], test["true_age_mean"])}
prenom_dict_deaths={k:v for k,v in zip(test2["prenom"], test2["expected_number_deaths"])}

# Passing the dictionaries to the subset data made earlier
data5["true_age_mean"]=data5["prenom"].apply(lambda x: prenom_dict[x] if x in prenom_dict.keys() else np.nan)
data5["expected_deaths"]=data5["prenom"].apply(lambda x: prenom_dict_deaths[x] if x in prenom_dict_deaths.keys() else np.nan)
data5["diff_age"]=data5["Average_Age"]-data5["true_age_mean"]
data5["diff_deaths"]=data5["count"]-data5["expected_deaths"]
data5[data5["paysnaiss"]=="FRANCE"].sort_values("count", ascending=False)
```
This is how the dataframe will look like:

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>index</th>
      <th>prenom</th>
      <th>paysnaiss</th>
      <th>sex</th>
      <th>Average_Age</th>
      <th>Age_Std</th>
      <th>count</th>
      <th>true_age_mean</th>
      <th>diff_age</th>
      <th>expected_deaths</th>
      <th>diff_deaths</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>9220</th>
      <td>9220</td>
      <td>JEAN</td>
      <td>FRANCE</td>
      <td>1</td>
      <td>81.427231</td>
      <td>9.981261</td>
      <td>26590</td>
      <td>71.496793</td>
      <td>9.930438</td>
      <td>31118.0</td>
      <td>-4528.0</td>
    </tr>
    <tr>
      <th>31809</th>
      <td>31809</td>
      <td>MARIE</td>
      <td>FRANCE</td>
      <td>2</td>
      <td>87.370109</td>
      <td>10.065417</td>
      <td>23119</td>
      <td>64.167322</td>
      <td>23.202787</td>
      <td>26032.0</td>
      <td>-2913.0</td>
    </tr>
    <tr>
      <th>12938</th>
      <td>12938</td>
      <td>MICHEL</td>
      <td>FRANCE</td>
      <td>1</td>
      <td>77.991489</td>
      <td>9.980199</td>
      <td>13106</td>
      <td>77.510527</td>
      <td>0.480962</td>
      <td>15077.0</td>
      <td>-1971.0</td>
    </tr>
    <tr>
      <th>1534</th>
      <td>1534</td>
      <td>ANDRE</td>
      <td>FRANCE</td>
      <td>1</td>
      <td>84.324293</td>
      <td>9.302870</td>
      <td>10340</td>
      <td>83.527396</td>
      <td>0.796897</td>
      <td>11602.0</td>
      <td>-1262.0</td>
    </tr>
    <tr>
      <th>15072</th>
      <td>15072</td>
      <td>PIERRE</td>
      <td>FRANCE</td>
      <td>1</td>
      <td>83.212907</td>
      <td>10.771183</td>
      <td>10169</td>
      <td>74.677724</td>
      <td>8.535182</td>
      <td>11746.0</td>
      <td>-1577.0</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>32121</th>
      <td>32121</td>
      <td>MARIE-LAURE</td>
      <td>FRANCE</td>
      <td>2</td>
      <td>60.753077</td>
      <td>12.772656</td>
      <td>91</td>
      <td>49.978790</td>
      <td>10.774287</td>
      <td>86.0</td>
      <td>5.0</td>
    </tr>
    <tr>
      <th>25843</th>
      <td>25843</td>
      <td>FELICIE</td>
      <td>FRANCE</td>
      <td>2</td>
      <td>90.725333</td>
      <td>7.664114</td>
      <td>90</td>
      <td>15.246389</td>
      <td>75.478945</td>
      <td>100.0</td>
      <td>-10.0</td>
    </tr>
    <tr>
      <th>1026</th>
      <td>1026</td>
      <td>ALEX</td>
      <td>FRANCE</td>
      <td>1</td>
      <td>64.708222</td>
      <td>20.002979</td>
      <td>90</td>
      <td>24.057174</td>
      <td>40.651049</td>
      <td>126.0</td>
      <td>-36.0</td>
    </tr>
    <tr>
      <th>23496</th>
      <td>23496</td>
      <td>CLEMENTINE</td>
      <td>FRANCE</td>
      <td>2</td>
      <td>90.814944</td>
      <td>12.831586</td>
      <td>89</td>
      <td>25.975736</td>
      <td>64.839207</td>
      <td>106.0</td>
      <td>-17.0</td>
    </tr>
    <tr>
      <th>31910</th>
      <td>31910</td>
      <td>MARIE-ANTOINETTE</td>
      <td>FRANCE</td>
      <td>2</td>
      <td>83.652360</td>
      <td>12.100183</td>
      <td>89</td>
      <td>67.762314</td>
      <td>15.890046</td>
      <td>86.0</td>
      <td>3.0</td>
    </tr>
  </tbody>
</table>
<p>427 rows × 11 columns</p>
</div>


### Mapping age of deaths to location in France


It is possible to map the average age death to geographical division in France.
Here I associated it with the communes location, although the result is a bit too dense to be easily readable.

{% include choropleth.html %}

Code is as follow: 
```python
# Loading the geojson file I previously made using the June 2021 IGN data
with open('../raw_data/data.json', 'w') as f:
    json.dump(commu, f)

# Adding the geojson file that has the "arrondissements" for Paris, Lyon and Marseille, there is some overlap sadly
# I strip the file of its properties for easier manipulation
with open("../raw_data/arrondissement_simp.json") as json_file:
  arron = json.load(json_file)
for items in arron["features"]:
    items["properties"].pop("DATE_CREAT", None)
    items["properties"].pop("ID", None)
    items["properties"].pop("DATE_MAJ", None)
    items["properties"].pop("DATE_CONF", None)
    items["properties"].pop("DATE_APP", None)
    items["properties"].pop("ID_AUT_ADM", None)
    items["properties"].pop("ID_CH_LIEU", None)
    items["properties"]["INSEE_COM"]=items["properties"]["INSEE_ARM"]

# Merging the two geojson together to display all locations
commu["features"].extend(arron["features"])

# Previously I ran a function to update the communes name of birth to their current equivalent using
communes=pd.read_csv("../raw_data/commune2021.csv")
# Making a dictionary of the new value
dict_value={k:v for k,v in zip(movement["COM_AV"], movement["COM_AP"])}
# Applying the update
df["lieunaiss_str"]=df["lieunaiss_str"].apply(lambda x: dict_value[x] if x in dict_value.keys() else x)

# Grouping the original dataset by communes
df_communes_group=df[df["paysnaiss"]=="FRANCE"].groupby(["lieunaiss_str"] , as_index=False)[["lifespan"]].agg(["mean", "std", "count"]) 
df_communes_group.columns = ['_'.join(col) for col in df_communes_group.columns]
df_communes_group["lieunaiss_str_col"]=[df_communes_group.index[i] for i in range (df_communes_group.shape[0])]
df_communes_group.reset_index(inplace=True)

# Ploting the figure
fig=px.choropleth_mapbox(df_canton_group, geojson=commu, locations="lieunaiss_str_col", featureidkey="properties.INSEE_COM", color="lifespan_mean",
                           color_continuous_scale="Viridis",
                           range_color=(55, 100),
                           mapbox_style="carto-positron",
                           center=dict(lat=46.2276, lon=2.2137), zoom=3,
                           opacity=0.5                    
                          )
fig.update_layout(margin={"r":0,"t":0,"l":0,"b":0})
fig.show()

```

And for a less busy figure, using departments only

{% include choropleth_dep.html %}

Code is as follow:

``` python
# Loading the geojson file for department
with open("../raw_data/departement.json") as json_file:
  deparjson = json.load(json_file)
# Making a dictionary for department equivalent to communes zip codes
communes_dept_dict={k:v for k,v in zip (communes["COM"],communes["DEP"])}
df["dep"]=df["lieunaiss_str"].apply(lambda x: communes_dept_dict[x] if x in communes_dept_dict.keys() else np.nan)
# Drawing the picture
fig=px.choropleth_mapbox(df_dep, geojson=deparjson, locations="dep", featureidkey="properties.INSEE_DEP", color="lifespan_mean",
                           color_continuous_scale="Viridis",
                           range_color=(75, 85),
                           mapbox_style="carto-positron",
                           center=dict(lat=46.2276, lon=2.2137), zoom=3,
                           opacity=0.5                    
                          )
fig.update_layout(margin={"r":0,"t":0,"l":0,"b":0})
fig.show()

```

[Insee files](https://www.insee.fr/fr/information/2560452)

