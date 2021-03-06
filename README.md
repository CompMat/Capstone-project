# Capstone-project
Peer graded assignment


#Battle of the Neighbourhoods

# Import libraries
import numpy as np
import pandas as pd 
pd.set_option('display.max_columns', None)
pd.set_option('display.max_rows', None)

from geopy.geocoders import Nominatim 
import folium
import requests
import json

import matplotlib.cm as cm
import matplotlib.colors as colors
import matplotlib.pyplot as plt
import seaborn as sns

from sklearn.metrics import silhouette_score
from sklearn.cluster import KMeans


# wards of kyoto url
ward_url = 'https://en.wikipedia.org/wiki/Wards_of_Kyoto'

# More concise method than requests and Beautiful soup
dataframe_list = pd.read_html(ward_url, flavor='bs4')
df_raw = dataframe_list[1]
df_raw

# Data processing - refine and get relevant data
df_raw.columns = df_raw.columns.get_level_values(1)

df_raw

df_ward  = df_raw.iloc[: , :4]
df_ward.drop(index=11, inplace=True)
df_ward.tail(1)

# Check data types
df_ward.info()

print(f"df_ward has {df_ward.shape[0]} rows, {df_ward.shape[1]} columns")


# Furnish with Geographical Coordinates
# Define and name user_Agent as "Kyoto_wards".  
geolocator = Nominatim(user_agent="Kyoto_wards")

# Get the coordinates per ward name
df_ward['Name'].apply(geolocator.geocode).apply(lambda x: (x.latitude, x.longitude))


df_ward['Coord']= df_ward['Name'].apply(geolocator.geocode).apply(lambda x: (x.latitude, x.longitude))
df_ward.head(1)

# Split the coordinates into Lat and Long, drop the M
df_ward[['Latitude', 'Longitude']] = df_ward['Coord'].apply(pd.Series)
# drop the Coord column
df_ward.drop(['Coord'], axis=1, inplace=True)

df_ward.tail(1)

# Summary check on the datatypes, and null values
df_ward.info()

# Identify coordinates of Tokyo, define and name user_agent as "Kyoto_explorer"
place = 'Kyoto'
geolocator = Nominatim(user_agent="Kyoto_explorer")
location = geolocator.geocode(place)
kyoto_lat = location.latitude
kyoto_lon = location.longitude
print(f"Coordinates of {place} are {kyoto_lat}, {kyoto_lon}")


# Visualize the neighbourhoods
map_kyoto = folium.Map(location = [kyoto_lat, kyoto_lon], zoom_start = 12)

# Add markers
for lat, lng, label in zip(df_ward['Latitude'], df_ward['Longitude'], df_ward['Name']):
    folium.RegularPolygonMarker([lat, lng],
                                popup=label,
                                radius=5,
                                color='purple',
                                fill_color='blue',
                                fill_opacity=0.7).add_to(map_kyoto) 
# display map
map_kyoto


# Summary of the coordinates and wards
for lat, lng, label in zip(df_ward['Latitude'], df_ward['Longitude'], df_ward['Name']):
    print(lat,lng, label)


# Get the addresses for Kita-ku & Minami-ku
loc_kita = geolocator.geocode("Kita-ku")
loc_minami = geolocator.geocode("Minami-ku")
print(f"returned address of Kita-ku: {loc_kita.address}")
print(f"returned address of Minami-ku: {loc_minami.address}")

# Get the actual address of the kyoto wards
loc_kita_kyo = geolocator.geocode("Kita-ku, Kyoto")
loc_minami_kyo = geolocator.geocode("Minami-ku, Kyoto")
print(f"returned address of Kita-ku: {loc_kita_kyo.address}")
print(f"returned address of Minami-ku: {loc_minami_kyo.address}")

kita_kyo_lat = loc_kita_kyo.latitude
kita_kyo_lon = loc_kita_kyo.longitude
minami_kyo_lat = loc_minami_kyo.latitude
minami_kyo_lon = loc_minami_kyo.longitude
print(f"corrected coordinates of Kita-ku, Kyoto: {kita_kyo_lat}, {kita_kyo_lon}")
print(f"corrected coordinates of Minami-ku, Kyoto: {minami_kyo_lat}, {minami_kyo_lon}")


# Before coordinates replace
df_ward.loc[4]

# Update the coordinates in dataframe
df_ward.loc[3,'Latitude'] = kita_kyo_lat
df_ward.loc[3,'Longitude'] = kita_kyo_lon
df_ward.loc[4,'Latitude'] = minami_kyo_lat
df_ward.loc[4,'Longitude'] = minami_kyo_lon

# After coordinates replace
df_ward.loc[4]

# Replotting the map, 11 wards expected
map_kyoto = folium.Map(location = [kyoto_lat, kyoto_lon], zoom_start = 12)

# Add markers
for lat, lng, label in zip(df_ward['Latitude'], df_ward['Longitude'], df_ward['Name']):
    folium.RegularPolygonMarker([lat, lng],
                                popup=label,
                                radius=5,
                                color='purple',
                                fill_color='blue',
                                fill_opacity=0.7).add_to(map_kyoto) 
# display map
map_kyoto


#Explore the First Neighbourhood in Kyoto
# Remember to remove prior to release
CLIENT_ID = ''
CLIENT_SECRET = '' # your Foursquare Secret
VERSION = '20210624' # Foursquare API version
LIMIT = 100 # A default Foursquare API limit value

print('Your credentails:')
print('CLIENT_ID: ' + CLIENT_ID)
print('CLIENT_SECRET:' + CLIENT_SECRET)

# Name of first Kyoto ward in dataframe
nb_name = df_ward.loc[0,'Name'] 
print(f"First ward in Kyoto wards dataframe: {nb_name}")

# Get this neighbourhood's lat and lon
nb_lat = df_ward.loc[0, 'Latitude'] 
nb_lon = df_ward.loc[0, 'Longitude'] 
nb_name = df_ward.loc[0,'Name'] 

print(f"{nb_name}'s latitude and longitude: {nb_lat} & {nb_lon}.")


# Create the url
limit = 100
radius = 500
url = url='http://api.foursquare.com/v2/venues/explore?client_id={}&client_secret={}&v={}&ll={},{}&radius={}&limit={}'.format(CLIENT_ID,
                                                                                                                       CLIENT_SECRET,
                                                                                                                       VERSION,
                                                                                                                       nb_lat,
                                                                                                                       nb_lon,
                                                                                                                       radius,
                                                                                                                       limit)

# check url
url

# send the GET request and review the results
results = requests.get(url).json()
results


# function that extracts the category of the venue
def get_category_type(row):
    try:
        categories_list = row['categories']
    except:
        categories_list = row['venue.categories']
        
    if len(categories_list) == 0:
        return None
    else:
        return categories_list[0]['name']


venues = results['response']['groups'][0]['items']
    
nearby_venues = pd.json_normalize(venues) # flatten JSON

# filter columns
filtered_columns = ['venue.name', 'venue.categories', 'venue.location.lat', 'venue.location.lng']
nearby_venues =nearby_venues.loc[:, filtered_columns]

# filter the category for each row
nearby_venues['venue.categories'] = nearby_venues.apply(get_category_type, axis=1)

# clean column headers
nearby_venues.columns = [col.split(".")[-1] for col in nearby_venues.columns]

nearby_venues.head()


# Number of venues returned by Foursquare
print(f"number of venues returned via Foursquare API: {nearby_venues.shape[0]}")

# Number of unique categories in the neighbourhood
print(f"number of unique venue categories: {len(nearby_venues['categories'].unique())}")

# venues by category count
nearby_venues['categories'].value_counts()

#Explore the Neighbourhoods in Kyoto

def getNearbyVenues(names, latitudes, longitudes, radius=500):
    
    venues_list=[]
    for name, lat, lng in zip(names, latitudes, longitudes):
        print(name)
            
        # create the API request URL
        url = 'https://api.foursquare.com/v2/venues/explore?&client_id={}&client_secret={}&v={}&ll={},{}&radius={}&limit={}'.format(
            CLIENT_ID, 
            CLIENT_SECRET, 
            VERSION, 
            lat, 
            lng, 
            radius, 
            LIMIT)
            
        # make the GET request
        results = requests.get(url).json()["response"]['groups'][0]['items']
        
        # return only relevant information for each nearby venue
        venues_list.append([(
            name, 
            lat, 
            lng, 
            v['venue']['name'], 
            v['venue']['location']['lat'], 
            v['venue']['location']['lng'],  
            v['venue']['categories'][0]['name']) for v in results])

    nearby_venues = pd.DataFrame([item for venue_list in venues_list for item in venue_list])
    nearby_venues.columns = ['Neighborhood', 
                  'Neighborhood Latitude', 
                  'Neighborhood Longitude', 
                  'Venue', 
                  'Venue Latitude', 
                  'Venue Longitude', 
                  'Venue Category']
    
    return(nearby_venues)

kyoto_venues = getNearbyVenues(names=df_ward['Name'],
                               latitudes=df_ward['Latitude'],
                               longitudes=df_ward['Longitude'])


kyoto_venues_restr = kyoto_venues[kyoto_venues['Venue Category'].str.contains('Restaurant')].reset_index(drop=True)
# set index to start from 1
kyoto_venues_restr.index = np.arange(1, len(kyoto_venues_restr)+1)
kyoto_venues_restr.head()

print(f"Types of restaurants in kyoto: {len(kyoto_venues_restr['Venue Category'].unique())}")

# create a dataframe with the venue category and counts
df_counts = kyoto_venues_restr['Venue Category'].value_counts().to_frame(name='counts')
df_counts = df_counts.reset_index()
df_counts.rename(index=str, columns={"index": "venue_category"}, inplace=True)

df_counts

fig = plt.figure(figsize=(10,6))
fig = sns.barplot(x='venue_category',y='counts',data=df_counts[0:10],color='steelblue')
plt.title('10 Most Frequent Restaurant Types in Kyoto', fontsize=14)
plt.xlabel("Venue Category", fontsize=12)
plt.ylabel ("Counts", fontsize=12)
plt.xticks(rotation=45,  horizontalalignment='right')

for bar in fig.patches:
    # passing the coordinates where the annotation shall be done
    # x-coordinate: bar.get_x() + bar.get_width() / 2
    # y-coordinate: bar.get_height()
    # free space to be left to make graph pleasing: (0, 8)
    # ha and va stand for the horizontal and vertical alignment
    fig.annotate(format(bar.get_height(), '.0f'), 
                 xy=(bar.get_x() + bar.get_width() / 2, bar.get_height()),
                 ha='center', 
                 va='center',
                 size=10, xytext=(0, 8),
                 textcoords='offset points')

plt.savefig("10 Most Frequent Restaurant Types in Kyoto.png")

plt.show()


kyoto_venues_restr.groupby(['Neighborhood'])['Venue Category'].apply(lambda x: x[x.str.contains('Restaurant')].count()).sort_val

#Analyze Each Neighborhood
# one hot encoding
kyoto_onehot = pd.get_dummies(kyoto_venues_restr[['Venue Category']], prefix="", prefix_sep="")

# add neighborhood column back to dataframe
kyoto_onehot['Neighborhood'] = kyoto_venues_restr['Neighborhood'] 

# move neighborhood column to the first column
fixed_columns = [kyoto_onehot.columns[-1]] + list(kyoto_onehot.columns[:-1])
kyoto_onehot = kyoto_onehot[fixed_columns]

kyoto_onehot.head()

# check dataframe shape
print(f'new dataframe has {kyoto_onehot.shape[0]} rows, {kyoto_onehot.shape[1]} columns')

kyoto_grouped = kyoto_onehot.groupby('Neighborhood').mean().reset_index()
kyoto_grouped

# check dataframe shape
print(f'grouped dataframe has {kyoto_grouped.shape[0]} rows, {kyoto_grouped.shape[1]} columns')

num_top_venues = 5

for ward in kyoto_grouped['Neighborhood']:
    print("----"+ward+"----")
    temp = kyoto_grouped[kyoto_grouped['Neighborhood'] == ward].T.reset_index()
    temp.columns = ['venue','freq']
    temp = temp.iloc[1:]
    temp['freq'] = temp['freq'].astype(float)
    temp = temp.round({'freq': 2})
    print(temp.sort_values('freq', ascending=False).reset_index(drop=True).head(num_top_venues))
    print('\n')

# Function to sort values in descending order
def return_most_common_venues(row, num_top_venues):
    row_categories = row.iloc[1:]
    row_categories_sorted = row_categories.sort_values(ascending=False)
    
    return row_categories_sorted.index.values[0:num_top_venues]

num_top_venues = 10

indicators = ['st', 'nd', 'rd']

# create columns according to number of top venues
columns = ['Neighborhood']
for ind in np.arange(num_top_venues):
    try:
        columns.append('{}{} Most Common Venue'.format(ind+1, indicators[ind]))
    except:
        columns.append('{}th Most Common Venue'.format(ind+1))

# create a new dataframe
neighborhoods_venues_sorted = pd.DataFrame(columns=columns)
neighborhoods_venues_sorted['Neighborhood'] = kyoto_grouped['Neighborhood']

for ind in np.arange(kyoto_grouped.shape[0]):
    neighborhoods_venues_sorted.iloc[ind, 1:] = return_most_common_venues(kyoto_grouped.iloc[ind, :], num_top_venues)

neighborhoods_venues_sorted.head()

#Examine the Clusters

#cluster 1
 
kyoto_merged.loc[kyoto_merged['Cluster Labels'] == 0, 
                 kyoto_merged.columns[[0] + list(range(4, kyoto_merged.shape[1]))]]

#cluster 2
kyoto_merged.loc[kyoto_merged['Cluster Labels'] == 1, 
                 kyoto_merged.columns[[0] + list(range(4, kyoto_merged.shape[1]))]]

#cluster 3
kyoto_merged.loc[kyoto_merged['Cluster Labels'] == 2, 
                 kyoto_merged.columns[[1] + list(range(5, kyoto_merged.shape[1]))]]
