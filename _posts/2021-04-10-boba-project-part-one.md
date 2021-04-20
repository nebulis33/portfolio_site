---
layout: post
title: The Bay vs LA - Boba Edition
hero: /assets/images/boba_comparison.PNG
date: 2021-04-10 12:00:00 -0700
summary: "This project was originally inspired by a partial data set I found that had boba shops and their ratings, 
locations, etc pulled from the Yelp API. It got me thinking, what if I could create a data pipeline that uses more up to date information to compare boba between The Bay and LA?"
---

Hello and welcome to my first blog post! This is part one in a two part series; part two will be
about the data pipeline aspect of the project.

This is a pretty extensive post so feel free to read the full writeup below, or
        <a href="#findings">skip to the findings</a>. The tableau viz is also
        available [here](https://public.tableau.com/profile/colton4314#!/vizhome/boab_viz/Dashboard1).

As a note, I use boba, bubble tea, and milk tea to all refer to the same drink.

## Background

This project was originally inspired by a partial data set I found that had boba shops and their ratings, 
locations, etc pulled from the Yelp API. It got me thinking, what if I could create a data pipeline that 
follows a path like this:

1. A Python script pulls the complete data from the Yelp API 
2. Data is saved as a JSON file to Google Cloud Storage (GCS)
3. A Google Cloud Function (GCF) gets triggered by the upload 
4. Another script pulls the relevant info from the JSON file and saves it in a CSV file (of course it could be saved as CSV from the start but I wanted to play with GCF more)
5. The CSV file is then loaded to a data warehouse (BigQuery in this case) where duplicates are dealt with and any current records are updated appropriately
6. Tableau pulls the data set down from BigQuery and visualizes the data

I could then use the findings to decide which region has the better boba. My (biased) hypothesis was that the
Bay Area has higher ratings, but due to population SoCal has more locations.

## Process

Easily the hardest part is linking everything in the Google Cloud ecosystem. There are so many different services and I am not really familiar with any of them besides BigQuery. So, I did most of the project manually and as mentioned before, will
make another post when I have the pipeline fully built out detailing that process.

### Gathering Data
The first step was picking cities to include in each region. Originally, I was going to use every zipcode in the SF Bay Area census region, but this ran into roadblocks discussed later. I ended up using [this site](http://www.bayareacensus.ca.gov/cities/cities.htm) to get a list of city names, although I didn’t submit a search query for every one. While it would not really have been any more difficult or costly to include every city, I figured it was basically pointless as the smaller cities often overlapped almost entirely with their larger neighbors (i.e. Daly City and San Francisco). So, I chose the ‘big name’ ones, and also tried to make sure my choices were evenly distributed geographically. I also begrudgingly included cities like Fairfield and Gilroy which I don’t consider part of the Bay Area, in an attempt to keep my bias out of the results as much as possible.

Heres the list I used in the Python script:

<code>
    bay_cities = ['Santa Rosa, CA', 'Novato, CA', 'San Rafael, CA', 'San Francisco, CA', 'San Mateo, CA',
        'Palo Alto, CA', 'San Jose, CA', 'Fremont, CA', 'Hayward, CA', 'Oakland, CA',
        'Richmond, CA', 'Concord, CA', 'Vallejo, CA', 'Napa, CA', 'Pleasanton, CA',
        'Walnut Creek, CA', 'Santa Rosa, CA', 'Fairfield, CA', 'Vacaville, CA', 'Gilroy, CA', 'Antioch, CA']
</code>

‘SoCal’ area cities were a little trickier. Not being a native of SoCal it was hard to discern where SoCal started and where it ended. I knew that San Diego and LA shouldn't be considered together, but what about Orange County? Unfortunately, there’s not really a clear region like there is in the Bay Area. I ended up going with “Greater Los Angeles” and using the Los Angeles-Long Beach-Santa Ana-Riverside area. Basically from Thousand Oaks to Irvine, and Long Beach to San Bernardino. Google Maps came in handy to see which areas seemed geographically separate.

<code>
la_cities = ['Los Angeles, CA', 'Long Beach, CA', 'Irvine, CA', 'Santa Ana, CA', 'Anaheim, CA', 'Santa Monica, CA',
        'Burbank, CA', 'Malibu, CA', 'Thousand Oaks, CA', 'Santa Clarita, CA', 'Ontaria, CA', 'West Covina, CA',
        'Riverside, CA', 'San Bernardino, CA', 'Pasadena, CA', 'Huntington Beach, CA']
</code>

<!-- returns (sorry, not alphabetized):

|   |   |   |   |   |
|---|---|---|---|---|
|Mira Loma|Anaheim|Rowland Heights|West Hollywood|Tustin|
|Los Angeles|Gardena|La Verne|Santa Clarita|Wilmington|
|Glendale|Brea|Ontario|Running Springs|San Dimas|
|Riverside|Fullerton|Montclair|Pasadena|Signal Hill|
|Hacienda Heights|South Gate|Seal Beach|Huntington Beach|Simi Valley|
|Diamond Bar|Long Beach|Pomona|Whittier|Monrovia|
|Temple City|Irvine|Culver City|Laguna Hills|Upland|
|Buena Park|Thousand Oaks|Carson|La Habra|Montrose|
|Cypress|San Bernardino|Rosemead|Montebello|South Pasadena|
|Westminster|Van Nuys|Downey|Norwalk|Tujunga|
|Mission Viejo|La Mirada|Torrance|Santa Monica|Colton|
|Santa Ana|Pico Rivera|Walnut|Azusa|Redondo Beach|
|Baldwin Park|Fountain Valley|West Covina|Stanton|Winnetka|
|Claremont|Irwindale|Lynwood|Chino|Highland|
|Cerritos|Chino Hills|Alhambra|Covina|Loma Linda|
|South El Monte|Artesia|Garden Grove|La Crescenta|Foothill Ranch|
|Redlands|-|-|-|-| -->


The script was pretty simple, I considered using Ruby but went with Python because it really feels unmatched in simplicity and elegance for these types of projects. The hard part here was figuring out the Yelp API. On paper, it’s a simple API: you send a GET request to the API with the parameters you want to search by (location, category, term, etc) and their values, and a header with your API key.

However, I quickly hit a major tripping point: there’s a serious lack of granularity. The API and indeed Yelp itself doesn’t really provide a way to restrict or filter results on location (zip, city, geofence, etc). Well it does, but it will still return results outside of the boundary, mixed with results inside. As mentioned earlier this made it difficult to cleanly perform my original search be by zipcode. In the end, using the major cities in each region lead to plenty of overlap. For example, the Bay Area search returned 2030 raw results but only 638 unique entries. Nothing a bit of SQL (or even Excel in this case) can’t fix, but I would have prefered cleaner raw data.

### Cleaning
This time around I manually cleaned the data using Excel, but ideally in the future this will be a part of the pipeline. I removed restaurants that had category “foodtrucks” and any Panda Express entries. Not sure why Yelp assigns Panda Express the Bubble Tea category, but I don’t consider it a boba shop. A major caveat of this data is that it's difficult to define “boba shop”. Many places don't have “boba” in their names yet serve strictly bubble tea, while some only offer it as a drink in a full resturant setting. Going strictly by tag doesn't clarify things either as many bubble tea shops serve food and are thus assigned the same tags as resturants that happen to serve boba. Because of this, there are undoubtedly a handful of entries that are simply restaurants that happen to serve milk tea, but I believe there are too few of these to taint the data.

### On The Cloud
I uploaded the cleaned file to GCS into respective buckets (processed Bay Area data and processed LA data). In BigQuery I created a dataset and uploaded both files as tables. After running a few queries to check for accuracy, I simply connected the BigQuery dataset to Tableau and started visualizing.

## <a id="findings">Findings</a>
Of course the most important finding was that the LA area has a higher average rating than the Bay Area (noooo!). Debate abounds as to whether this means they have better quality boba, they are just more generous with ratings, or they have a low bar. Beyond the differences in averages, there were a few unexpected findings. 

First, the difference in the number of locations. LA has only 770 locations VS 638 in the Bay Area. That's only a 21% difference, despite there being almost 2.5 times as many people living in the LA area (7.75m vs 18.71m).

![graph of population vs boba shops](/assets/images/pop_v_boba.png)

Second, the chain Quickly was consistently rated very poorly.
Using:

<code>
SELECT<br>
    &nbsp;name,<br>
    &nbsp;rating,<br>
    &nbsp;ROUND(AVG(rating) OVER (PARTITION BY name), 2) AS chain_avg<br>
FROM (<br>
    &nbsp;SELECT<br>
        &nbsp;&nbsp;id,<br>
        &nbsp;&nbsp;LEFT(name, 7) AS name,<br>
        &nbsp;&nbsp;rating<br>
    &nbsp;FROM<br>
        &nbsp;&nbsp;bay_area_boba<br>
    &nbsp;WHERE<br>
        &nbsp;&nbsp;name LIKE '%Quickly%') sub
</code>

We get Bay Area Results showing the average rating of the chain is 2.98 over 32 locations (yes I used a window function
just to show I know how to, sue me).
Additionally, there are several locations with a sub 3 star rating with the minimum rating accross all locaitons being 1.5 stars, which is pretty bad.

SoCal quickly fares better but is still well below the regional average: the chain scores 3.38 stars over 8 locations, this time with a minimum of 2.5 stars.

This is a somewhat surprising finding as not only is Quickly a fairly widespread chain, but they were one of the early pioneers of the boba shop in California. Between their experience and what is clearly enough popularity to maintain dozens of locations, I would have expected better. Indeed, in the Bay Area the average Quickly rating is actually 2 SD from the mean, which is a particularly poor performance.

![the mean is 3.86 and the SD is 0.55](/assets/images/boba_bay_stats.png)

Finally, I was surprised to see the difference in the quantity of ratings. San Francisco had a
total of 35,256 ratings across its locations and San Jose, the most populous city in the region, had 30,717. Compare this to LA with only 23,113 followed
by Irvine with 16,175. Like before, this is pretty surprising gven the difference in population. Even with fewer shops per capita I would have expected LA to have more ratings than SF simply because their populaiton dwarfs ours.

## Conclusion

The findings unfortunately refuted my original hypothesis. 
However, they seem to indicate that there is a significant difference in 'boba culture' between the Bay Area
and Southern California. This possibly explains the divergence in numbers of ratings and locations between
the two regions. In a region where boba is more deeply ingrained in the culture there is demand to
support more locations and more visits lead to more Yelp reviews. It may also explain why SoCal has a higher average rating: without a deeply entrenched boba culture people are less harsh with ratings.

I didn't compare this to granular demographic data, but my hypothesis for
this divergence in culture is there is a larger population of fresh immigrants from Asia in the Bay Area. Since boba
is so big in places like Hong Kong and Taiwan (having originated there), the high concentration of immigrants from these locations
likely offers a dual benefit. Immigrants coming from boba rich areas have higher consumption and therefore are
able to sustain more shops, while at the same time they are bringing new innovations, flavors,
and supply chain connections from their home country leading to a fresher and more vibrant
boba scene.

Overall this was an enjoyable project. Heep an eye out for the second post in this two part series.
Additionally, I will be implementing a live tracker on this
site that compares rating averages between the two regions... at some point... Keep an eye out for that.

Thanks for reading!
