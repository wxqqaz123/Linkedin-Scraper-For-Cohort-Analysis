# Linkedin Scraper

This project aims to scrape Linkedin profiles returned by search engines based on the keywords the users are interested in and to upload the result sheets to AWS S3 and DynamoDB through some essential ETL work initiated by Glue and Lambda. Keywords can be role type/description but more so can be typical family names from across different cultural and ethnic backgrounds. The cleansed dataset is ideal for cohort analysis.   


## Scraping

Given that Linkedin discourage the usage of automation tools, it is highly recommended to build your own proxies/accounts pool or at least to use premium-tier account to avoid getting banned. Before running code, make sure you have Chrome Webdriver installed as selenium is used.

```Python
import requests
from bs4 import BeautifulSoup
import re
from selenium import webdriver
from selenium.webdriver.common.keys import Keys
import time
import random
import requests
import pandas as pd
import os
def Linkedin_Scraper(Username,Password,Keywords,Companies,Country,Location,profile_photo_path=None):
    # Initiate Webdriver and Login your Linkedin Account
    driver = webdriver.Chrome()
    driver.get('https://www.linkedin.com/login?fromSignIn=true&trk=guest_homepage-basic_nav-header-signin')
    username=driver.find_element_by_xpath("/html/body/div/main/div[2]/form/div[1]/input")
    username.send_keys(Username)
    password=driver.find_element_by_xpath("/html/body/div/main/div[2]/form/div[2]/input")
    password.send_keys(Password)
    submit=driver.find_element_by_xpath("/html/body/div/main/div[2]/form/div[3]/button")
    time.sleep(1.5)
    submit.click()
    details=[]
    for Company in Companies:
        # Creating staging lists for profile urls and locations.
        urls=[]
        loc=[]
        # Creating superlists for qualified urls and names (Linkedin IDs)
        g_qualified=[]
        g_names=[]
        for Keyword in Keywords:
            query = f"site:linkedin.com/in {Company} AND {Keyword} AND {Country}"
            query = query.replace(' ', '+')
            l=[1,51]
            USER_AGENT = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.169 Safari/537.36"
            headers = {"user-agent" : USER_AGENT}
            for i in l:
                # Acquiring Linkedin profile urls from Bing search results, please note that Bing only shows up to 50 results per page
                # In this case, I'd like to scrape first 100 profiles which is 2 pages equivalent, page is splitted by starting index
                # This is why we need a for loop against [1,51]
                URL = f"https://bing.com/search?q={query}&count=50&first={i}&FORM=PERE"
                resp = requests.get(URL, headers=headers)
                soup = BeautifulSoup(resp.text)
                links=[i.get_text() for i in soup.findAll("div",{"class":"b_attribution"})]
                locations=[i.get_text() for i in soup.findAll("div",{"class":"b_caption"})]
                # Preprocessing the scraped urls and locations, please note that only a url containing 'linkedin.com/in' is 
                # considered a valid profile page. In most cases the corresponding location to a profile is also available in
                # Bing search results that we can capture straight away. if it is not there, we can use the default location in
                # the args as a proxy for the time being
                for i in links:
                    if r"linkedin.com/in" in i:
                        urls.append(i)
                for i in locations:
                    if r"linkedin.com/in" in i:
                        if "Location:" in i:
                            loc.append(i.split("Location:")[1])
                        else:
                            loc.append(Location)
                # Filtering out unwanted profiles by location and keyword
                qualified=[i for i,j in zip(urls,loc) if Location in j]
                qualified=[i for i in qualified if Keyword.lower() in i.split("/in")[1]]
                qualified=list(set(qualified))
                names=[i.split("com/in/")[1] for i in qualified]
                names=list(set(names))
                g_names.extend(names)
                g_qualified.extend(qualified)
                urls.clear()
                loc.clear()
        if profile_photo_path:
            subfolder=profile_photo_path+Company
            if not os.path.exists(subfolder):
                os.mkdir(subfolder)
        for i,j in zip(g_qualified,g_names):
            try:
                driver.get(i)
                pg_source=driver.page_source
                soup2 = BeautifulSoup(pg_source)
                current_company = soup2.findAll("span",{"class":"text-align-left ml2 t-14 t-black t-bold full-width lt-line-clamp lt-line-clamp--multi-line ember-view"})[0].get_text()
                if Company in current_company: 
                    if profile_photo_path:
                        subfolder=profile_photo_path+Company
                        if not os.path.exists(subfolder):
                            os.mkdir(subfolder)
                        img = driver.find_element_by_xpath('/html/body/div[7]/div[3]/div/div/div/div/div[2]/main/div[1]/section/div[2]/div[1]/div[1]/div/div/img')
                        src = img.get_attribute('src')
                        try:
                            r=requests.get(src)
                            with open(subfolder+"\\"+f"{j}"+".jpg","wb") as f:
                                f.write(r.content)
                        except:
                            pass
                    full_name = soup2.findAll("li",{"class":"inline t-24 t-black t-normal break-words"})[0].get_text().strip()
                    current_position = soup2.findAll("h2",{"class":"mt1 t-18 t-black t-normal break-words"})[0].get_text().strip()
                    current_position = current_position.split(" at")[0]
                    # Scrape estimated years of working experience
                    experience_yrs=soup2.findAll("span",{"pv-entity__bullet-item-v2"})
                    experience_yrs=[i.get_text().strip() for i in experience_yrs]
                    experience_yrs_str=" ".join(experience_yrs)
                    yrs=re.findall(r"\d+\syrs?",experience_yrs_str)
                    mons=re.findall(r"\d+\smos?(?!r)",experience_yrs_str)
                    yrsn=[int(i.split()[0]) for i in yrs]
                    monsn=[int(i.split()[0]) for i in mons]
                    total_months=sum(yrsn)*12+sum(monsn)
                    yrs_experience=f"{total_months//12} Years {total_months%12} Months"
                    details.append([full_name,current_position,Company,yrs_experience,i])   
            except Exception as e:
                print(e)
            time.sleep(random.uniform(1.7,3.1))
        print("Company Batch Done")
        time.sleep(3)
    results=pd.DataFrame(details,columns=["Name","Current Position","Company","Years of Experience","Linkedin URL"])
    results=results[results["Current Position"]!="at"]
    return results

if __name__ == "__main__":
    Username = "your username"
    Password = "your password"
    Keywords = ["keyword1","keyword2","keyword3"]
    Companies = ["Company1","Company2"]
    Country = "The country you'd like to get results from"
    Location = "The city you'd like to get results from"
    profile_photo_path = r"C:your path\\"
    df = Linkedin_Scraper(Username,Password,Keywords,Companies,Country,Location,profile_photo_path=None)
```

Ideally, you will get a dataframe as below:
![Pic](https://github.com/wxqqaz123/Linkedin-Scraper-For-Cohort-Analysis/blob/main/img/demo.png)




## ETL Example With AWS Glue

Data munging through Pyspark to add more dimensions to the raw dataset

```Python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from pyspark.sql.functions import *
from pyspark.sql.types import *

glueContext = GlueContext(SparkContext.getOrCreate())

df = spark.read.option("header","true").option("sep",",").csv(r"S3 path of your scraped dataset")

exp_to_split = split(df["Years of Experience"], " ")

df = df.withColumn("Total Months",exp_to_split.getItem(0).cast(IntegerType())*12 + 
                   exp_to_split.getItem(2).cast(IntegerType()))
df = df.withColumn("Seniority",when(df["Total Months"]>=120,"Highly Experienced")
                   .when(df["Total Months"]>60,"Experienced")
                   .otherwise("Entry Level")).drop("_c0")
df2 = DynamicFrame.fromDF(df, glueContext, "nested")

glueContext.write_dynamic_frame.from_options(
       frame = df2,
       connection_type = "s3",
       connection_options = {"path": "s3://xxxxxxx  your output S3 path"},
       format = "csv")
```

The transformed dataset will be like:
![Pic](https://github.com/wxqqaz123/Linkedin-Scraper-For-Cohort-Analysis/blob/main/img/demo2.png)

## Load to DynamoDB with Lambda
Finally we load transformed data to DynamoDB by leveraging Lambda
```Python
import boto3
import csv

def lambda_handler(event, context):
    region='us-east-1'
    s3=boto3.client('s3')            
    obj= s3.get_object(Bucket='your bucket', Key='key to the transformed dataset')
    data = obj['Body'].read().decode('utf-8').splitlines()
    items=[]
    csv_reader = csv.reader(data, delimiter=',', quotechar='"')
    next(csv_reader,None)
    for row in csv_reader:
          data = {}
          data['Name'] = row[0]
          data['Current Position']=row[1]
          data['Company']=row[2]
          data['Years of Experience']=row[3]
          data['Linkedin URL']=row[4]
          data['Seniority']=row[6]
          items.append(data)
    dynamodb = boto3.resource('dynamodb')
    db = dynamodb.Table('Linkedin')
    with db.batch_writer() as batch:
         for item in items:
             batch.put_item(Item=item)
```
The Items uploaded to DynamoDB will be like:

![Pic](https://github.com/wxqqaz123/Linkedin-Scraper-For-Cohort-Analysis/blob/main/img/demo3.png)
