---
layout: post
title: The Bay vs LA Part Two
hero: /assets/images/pipes.jpg
date: 2021-05-13 12:00:00 -0700
summary: "This is the second half of the earlier data analysis project I did using Yelp data on boba locations.
For this portion I built an ETL pipeline to automate the data tasks."
---

Welcome to the long awaited part 2 of the boba project. Originally I was hoping to complete
the pipeline portion of the project immediately after "Part 1", but my schedule filled up and
caused a delay. Feel free to look at the raw code [here](https://github.com/nebulis33/google_cloud_functions). 

Also, an important caveat is that while
I have Tableau Desktop connected directly to BigQuery, since I don't have a Tableau Server subscription,
the only way to share the viz is through Tableau Public which doesnt allow server connections. So that part
of the pipeline will not be 'live' on the public facing end unfortunately.

## Goals

To recap, the goal was to build a batch ETL pipeline as follows:

1. Use Google pub/sub to schedule an event once a month
2. A Google Cloud Function (GCF) triggered by a pub/sub topic will run, pulling the raw json data from Yelp and saving it to
a Google Cloud Storage (GCS) bucket
3. Another GCF will be triggered by the save and will clean and transform the data into a csv file, saving this
to another GCS bucket
4. This triggers a final GCF to take the saved csv and upload it to a table in BigQuery
5. Tableau is linked to the BigQuery table and thus updates automatically

![google data pipeline flow](/assets/images/gcp_serverless_large.png)

## Process

I decided the best bet was to work backwards, so my first step was to make two GCFs: one that triggers on creation
of a new file in the bucket "boba_bay_bucket_clean" and loads the file to the "clean_bay_boba" table in BigQuery,
and one that does the same for the socal data. This took forever because it was my first time using GCF in any way,
so it was all learn as you go, and quite frankly, the Goolge Cloud documentation isn't that great. Eventually, I figured
out how to use the bigquery module of the cloud library and everything fell into place from there. [This site](https://www.ternarydata.com/news/use-python-and-google-cloud-to-schedule-a-file-download-and-load-into-bigquery-3p3aw) was a big help in figuring out the correct implementation. A key tip is that the function GCF calls (hello_gcs by default) must accept a data payload and a context payload (which is metadata). The data payload (or whatever you name it) is then used to get things like bucket, file name, etc. The payloads don't
have to be used, but the function must accept both as arguments or else it wil crash.

The next step was to load dirty data from the dirty bucket, transform it into a clean csv, and upload the new file to the
clean bucket. Here I decided to diverge from the original plan a little bit. Instead of saving the raw JSON from the
API call as dirty then processing that, I decided it was eaiser to just convert it to CSV immediately, and save
the unfiltered CSV as the dirty file. This was partly because I realized I don't really have a use for JSON data but
also because I already had this functionality written. I also decided it was easiest to just get rid of any records with the "foodtrucks"
category, even though there was one I had identified before as transitioning to a regular location. To clean the data I
used pandas and made data frames that removed duplicates, foodtrucks, and Panda Express locations. 

<code>
    df.drop_duplicates(subset=None, inplace=True)
    np = df[~df.city.isin(prohibited)]
    nf = np[np.categories.str.contains("foodtrucks")==False]
    new_df = nf[nf.alias.str.contains("panda-express")==False]
    new_df.to_csv('/tmp/temp_data.csv', index=False, encoding='utf-8')
</code>

For the Bay Area I also removed several cities that don't belong in the data, like Sacramento. One thing I'm sure could be improved here is
the usage of dataframes. I'm not that well versed in pandas so I used a new dataframe for every transformation, but I'm sure
with better knowledge multiple transformations could be done in the same dataframe. A stumbling point here was that pandas has the ability to
read_csv() directly from a GCS bucket if passed the path (gs://bucket/file.csv), but for some reason it can't save directly to a bucket.
I orginally tried using StringIO but it lead to errors since the data has UTF-8 character in it (some names have Chinese characters),
so saving to the tmp directory and uploading from there was the simplest option.

The next step was downloading the actual data from the Yelp API into the dirty buckets. I considered just keeping this process on my local
machine, but it would have made the pipeline sort of useless if it just transforms and loads. Luckily this was one of the
quickest steps. I was able to basically just copy over the original script I had written and add only minimal functionality
to facilitate uploading to GCS. I used the tmp directory again to save the CSV file for upload to GCS, because GCS
doesn't allow direct writing to a file sored in a bucket - you must upload a whole blob.
The use of /tmp will probably become an issue when my free trial runs out since tmp storage counts towards memory usage, but thats an
issue for later.

Finally, the easiest step: creating Cloud Scheduler jobs. All of the cloud functions are triggered by the creation of a
new file in their respective buckets, except the API call functions. I want them to run on the first of each month, so they
are triggered by a Cloud Scheduler job. [This site](https://crontab.guru/every-month) is a great resource for creating cron schedules.

## Conclusion

Overall this project took a little longer than expected since it was my first experience with Google Cloud. Besides that
it was enjoyable, the value of cloud functions is obvious as now I don't have to run code and play with data every month,
I can just look at the end result and correct any one-off mistakes while having a somewhat live dashboard. There are a few
things I would like to add, primarily the ability to get an SMS when the BigQuery tables load, but nothing major.