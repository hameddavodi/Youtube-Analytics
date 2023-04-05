## Exploratory Data Analysing for Persian Rap Reaction Youtubers

Explanatory Data Analysis (EDA) is a crucial step in data analysis that helps to understand the data and extract meaningful insights from it. It is a process of summarizing the main characteristics of a dataset using visual methods such as histograms, scatter plots, box plots, etc., and identifying patterns, trends, and relationships between variables.

In this project, I plan to explore the following:

  - Learn about Youtube API and how to obtain video data.
  - Analyze video data and verify different common “myths” about   what makes a video do well on Youtube. For example:
  - Does the number of likes and comments matter for a video to get more views?
  - Does the video duration matter for views and interaction (likes/ comments)?
  - Does title length matter for views?
  - How many tags do good performing videos have? What are the common tags among these videos?
  - Across all the creators I take into consideration, how often do they upload new videos? On which days in the week?
  - Explore the trending topics using NLP techniques
  - Which popular topics are being covered in the videos (e.g. using wordcloud for video titles)?
  - Which questions are being asked in the comment sections in the videos?

Also pre-requisites libraries for this project are:

```python 


import pandas as pd
import numpy as np
from dateutil import parser
import isodate

# Data visualization libraries
import matplotlib.pyplot as plt
import matplotlib.ticker as ticker
import seaborn as sns
sns.set(style="darkgrid", color_codes=True)

# Google API
from googleapiclient.discovery import build

#NLP libraries

import nltk
from nltk.corpus import stopwords
from nltk.tokenize import word_tokenize
nltk.download('stopwords')
nltk.download('punkt')
from wordcloud import WordCloud
```
## Dataset Collection and API connection:

Step 1: create a project on Google Developers Console.
Step 2: Request an authorization credential (API key).
Step 3: Enable API.
Step 4: Collect channel IDs to start research on them. 


```python 

# Connecting to the DATA v3 API key
# Parameters: 
   # youtube: the build object from googleapiclient.discovery
   # channels_ids: list of channel IDs
    
api_key = 'AIzaSyB-4NIQtecQPbRX7TWKphThkb9_Brh2wL4' 


channel_ids = ['UCJk9mjbfuZ3k7oiWLlbKYFA', # Persian Rap Reaction
               'CB0qQXpkEjVK2_yd5w-Khgw', # RealPelmx
               'UCoOjmdECYvybtOOKtlwvsAw', # RadioActive Zone
               'UCV_bYCslgWyIZWH_Jita9rw', # MA2YAR TV
              ]

youtube = build('youtube', 'v3', developerKey=api_key)



```
Then to collect data from selected channels I can write following functions:

```python 

# Channel stats
def get_channel_stats(youtube, channel_ids):

    all_data = []
    request = youtube.channels().list(
                part='snippet,contentDetails,statistics',
                id=','.join(channel_ids))
    response = request.execute() 
    
    for i in range(len(response['items'])):
        data = dict(channelName = response['items'][i]['snippet']['title'],
                    subscribers = response['items'][i]['statistics']['subscriberCount'],
                    views = response['items'][i]['statistics']['viewCount'],
                    totalVideos = response['items'][i]['statistics']['videoCount'],
                    playlistId = response['items'][i]['contentDetails']['relatedPlaylists']['uploads'])
        all_data.append(data)
    
    return pd.DataFrame(all_data)
    
# Returns:
   # Dataframe containing the channel statistics for all channels in the provided list: title, subscriber count, view count, video count, upload playlist.


def get_video_ids(youtube, playlist_id):
    
    request = youtube.playlistItems().list(
                part='contentDetails',
                playlistId = playlist_id,
                maxResults = 50)
    response = request.execute()
    
    video_ids = []
    
    for i in range(len(response['items'])):
        video_ids.append(response['items'][i]['contentDetails']['videoId'])
        
    next_page_token = response.get('nextPageToken')
    more_pages = True
    
    while more_pages:
        if next_page_token is None:
            more_pages = False
        else:
            request = youtube.playlistItems().list(
                        part='contentDetails',
                        playlistId = playlist_id,
                        maxResults = 50,
                        pageToken = next_page_token)
            response = request.execute()
    
            for i in range(len(response['items'])):
                video_ids.append(response['items'][i]['contentDetails']['videoId'])
            
            next_page_token = response.get('nextPageToken')
        
    return video_ids

def get_video_details(youtube, video_ids):

        
    all_video_info = []
    
    for i in range(0, len(video_ids), 50):
        request = youtube.videos().list(
            part="snippet,contentDetails,statistics",
            id=','.join(video_ids[i:i+50])
        )
        response = request.execute() 

        for video in response['items']:
            stats_to_keep = {'snippet': ['channelTitle', 'title', 'description', 'tags', 'publishedAt'],
                             'statistics': ['viewCount', 'likeCount', 'favouriteCount', 'commentCount'],
                             'contentDetails': ['duration', 'definition', 'caption']
                            }
            video_info = {}
            video_info['video_id'] = video['id']

            for k in stats_to_keep.keys():
                for v in stats_to_keep[k]:
                    try:
                        video_info[v] = video[k][v]
                    except:
                        video_info[v] = None

            all_video_info.append(video_info)
            
    return pd.DataFrame(all_video_info)

def get_comments_in_videos(youtube, video_ids):
    """
    Get top level comments as text from all videos with given IDs (only the first 10 comments due to quote limit of Youtube API)

    all_comments = []
    
    for video_id in video_ids:
        try:   
            request = youtube.commentThreads().list(
                part="snippet,replies",
                videoId=video_id
            )
            response = request.execute()
        
            comments_in_video = [comment['snippet']['topLevelComment']['snippet']['textOriginal'] for comment in response['items'][0:10]]
            comments_in_video_info = {'video_id': video_id, 'comments': comments_in_video}

            all_comments.append(comments_in_video_info)
            
        except: 
            # When error occurs - most likely because comments are disabled on a video
            print('Could not get comments for video ' + video_id)
        
    return pd.DataFrame(all_comments) 
```
To get channel statistics:

```python 

channel_data = get_channel_stats(youtube, channel_ids)
channel_data

```

