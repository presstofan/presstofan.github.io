---
title:  "Create Facebook friend network graph using Python"
date:   2018-10-13 22:51:03 +0100
categories:
  - Python
tags:
  - Python
  - Facebook
  - Social Network
featured_image:
header:
  teaser: assets/images/facebook_graph_teaser_500x300.png
---

I have recently been asked to generate a social network graph to show how friends on Facebook are connected. This type of network graph can be used to identify communities as well as key persons in the network. It used to be much easier to generate graph like this (through some 3rd part apps or the innate API) but Facebook is getting increasingly strict about what data can be accessed through the API. The `all_mutual_friends`, `mutual_friends`, and `three_degree_mutual_friends` context edges of the Facebook Social Context API were deprecated on April 4, 2018 and immediately started returning empty data sets. They have now been fully removed. This post presents an alternative way to get and analyses Facebook friend network by the help of Python and Selenium Webdriver.

## Step 1: Retrieve friend and mutual friend information from a Facebook account

In order to make the script work, we will need to set up Selenium ChromeDriver first. ChromeDriver is a web automation framework that allows you to control the behaviours of the Chrome browswer using a programming launguage (Python in this case). I found [this guide from Chris Kenst](https://www.kenst.com/2015/03/installing-chromedriver-on-mac-osx) particularly useful for making the ChromeDriver work on macOs.

Once making sure the ChromeDriver, the following script by [Lucas Allen](https://github.com/lgallen/twitter-graph) and modified by [Eliot Andres](https://github.com/EliotAndres/facebook-friend-graph) can be used to scrap the friend network. First, we will need to load a couple of useful packages:

`html.parser`: for parsing HTML
`webdriver`: for controlling the ChromeDriver
`tqdm`: for showing a progress bar while scrapping
`pickle`: for backing up the checkpoint and storing the scrped friend information
`getpass`: for the login prompt

```python
from html.parser import HTMLParser
import re
import time
from selenium import webdriver
from selenium.webdriver.common.keys import Keys
import os
from tqdm import tqdm
import pickle
import getpass
```

Now we are introducing several functions that help to navigate through Facebook pages. Facebook pages are not static and many of them are using AJAX (Asynchronous JavaScript And XML), which allows web pages to be updated asynchronously by exchanging data with a web server behind the scenes. This means that it is possible to update parts of a web page, without reloading the whole page. For example, when visiting the friend page, only the first few friends are listed. But more friends will be loaded once the user scroll down to the bottom of the page. The `get_fb_page` function does exactly that. Given a page, the function will determine height of the body to calculate the scrolling distance, scroll down to the bottom, wait for a while for the AJAX to load new content, calculate new scrolling distance and scroll down again. It will stop when no new content is loaded.

```python
def get_fb_page(url):
    time.sleep(5)
    driver.get(url)

    # Get scroll height
    last_height = driver.execute_script("return document.body.scrollHeight")

    while True:
        # Scroll down to bottom
        driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")

        # Wait to load page
        time.sleep(SCROLL_PAUSE_TIME)

        # Calculate new scroll height and compare with last scroll height
        new_height = driver.execute_script("return document.body.scrollHeight")
        if new_height == last_height:
            break
        last_height = new_height
    html_source = driver.page_source
    return html_source
```

`MyHTMLParser` is a class which help us to find the URLs to friends' home page. `find_friend_from_url` function simply direct us to a friend's home page by a given URL.

```python
def find_friend_from_url(url):
    if re.search('com\/profile.php\?id=\d+\&', url) is not None:
        m = re.search('com\/profile.php\?id=(\d+)\&', url)
        friend = m.group(1)
    else:
        m = re.search('com\/(.*)\?', url)
        friend = m.group(1)
    return friend

class MyHTMLParser(HTMLParser):
    urls = []

    def error(self, message):
        pass

    def handle_starttag(self, tag, attrs):
        # Only parse the 'anchor' tag.
        if tag == "a":
            # Check the list of defined attributes.
            for name, value in attrs:
                # If href is defined, print it.
                if name == "href":
                    if re.search('\?href|&href|hc_loca|\?fref', value) is not None:
                        if re.search('.com/pages', value) is None:
                            self.urls.append(value)
```

Once we have loaded all the functions and packages above, we can use the following script to log in to Facebook. It will prompt you to input username and password for the login.

```python
username = input("Facebook username:")
password = getpass.getpass('Password:')

chrome_options = webdriver.ChromeOptions()
prefs = {"profile.default_content_setting_values.notifications": 2}
chrome_options.add_experimental_option("prefs", prefs)
driver = webdriver.Chrome(chrome_options=chrome_options)

driver.get('http://www.facebook.com/')

# authenticate to facebook account
elem = driver.find_element_by_id("email")
elem.send_keys(username)
elem = driver.find_element_by_id("pass")
elem.send_keys(password)
elem.send_keys(Keys.RETURN)
time.sleep(5)
print("Successfully logged in Facebook!")

SCROLL_PAUSE_TIME = 2
```

The ChromeDriver will open a new Chrome instance and log in Facebook. Once successfully log in. Execute the codes below to get the URLs to your friends' homepage. It will also store the friend list into a Pickle object `uniq_urls.pickle`, so we don't need to replace the same process when network failure occurs.

```python
my_url = 'http://www.facebook.com/' + username + '/friends'

UNIQ_FILENAME = 'uniq_urls.pickle'
if os.path.isfile(UNIQ_FILENAME):
    with open(UNIQ_FILENAME, 'rb') as f:
        uniq_urls = pickle.load(f)
    print('We loaded {} uniq friends'.format(len(uniq_urls)))
else:
    friends_page = get_fb_page(my_url)
    parser = MyHTMLParser()
    parser.feed(friends_page)
    uniq_urls = set(parser.urls)

    print('We found {} friends, saving it'.format(len(uniq_urls)))

    with open(UNIQ_FILENAME, 'wb') as f:
        pickle.dump(uniq_urls, f)
```

Next, we will use the URLs collected above to access the mutual friend page of each friend and retrieve the list of mutual friends. This is a slightly tricky part as Facebook are not happy with repetitive activities even you check your own friend list. It could temporarily ban you from accessing friend pages if you request more than 100 pages in one go. As such, the script is set to pause for half an hour at every 100 pages.

```python
# need to pause every 100 friends
friend_graph = {}
GRAPH_FILENAME = 'friend_graph.pickle'

if os.path.isfile(GRAPH_FILENAME):
    with open(GRAPH_FILENAME, 'rb') as f:
        friend_graph = pickle.load(f)
    print('Loaded existing graph, found {} keys'.format(len(friend_graph.keys())))

count = 0
for url in tqdm(uniq_urls):

    count += 1
    if count % 100 == 0:
        print ("Too many queries, pause for a while...")
        time.sleep(1800)

    friend_username = find_friend_from_url(url)
    if (friend_username in friend_graph.keys()) and (len(friend_graph[friend_username]) >1):
        continue

    friend_graph[friend_username] = [username]
    mutual_url = 'https://www.facebook.com/{}/friends_mutual'.format(friend_username)
    mutual_page = get_fb_page(mutual_url)

    parser = MyHTMLParser()
    parser.urls = []
    parser.feed(mutual_page)
    mutual_friends_urls = set(parser.urls)
    print('Found {} urls'.format(len(mutual_friends_urls)))

    for mutual_url in mutual_friends_urls:
        mutual_friend = find_friend_from_url(mutual_url)
        friend_graph[friend_username].append(mutual_friend)

    with open(GRAPH_FILENAME, 'wb') as f:
        pickle.dump(friend_graph, f)

    time.sleep(5)
```

It will take a while to scrap all the friends but when it is done, friend names and their connections should be stored in the pickle file `friend_graph.pickle`.

## Step 2: Community detection and plotting the friend graph

The following libraries beed to be installed:

`python-louvain==0.10`
`networkx==2.1`
`colorlover==0.2.1`
`numpy==1.14.0`
`plotly==2.5.1`

Note that `python-louvain` is imported as `community` as the code below.

```python
import plotly.offline as py
from plotly.graph_objs import *

import community
import networkx as nx
import colorlover as cl
import numpy as np
import pickle

py.init_notebook_mode(connected=True)

# This is your Facebook id. It can also be a number
CENTRAL_ID = 'Facebook_ID'

# This is the pickle file containing the raw graph data
GRAPH_FILENAME = 'friend_graph.pickle'

# To generate the friend graph, see:
# friend_graph a dict of lists in the form {'friend1': ['friend2, 'friend3']}
with open(GRAPH_FILENAME, 'rb') as f:
    friend_graph = pickle.load(f)
```

First, we'll clean the edges of the graph and only keep friends with a least 2 mutual friends.

```python
edges = []
nodes = [CENTRAL_ID]

central_friends = {}

for k, v in friend_graph.items():
    intersection_size = len(np.intersect1d(list(friend_graph.keys()), v))
    if intersection_size > 2:
        central_friends[k] = v

print('Firtered out {} items'.format(len(friend_graph.keys()) - len(central_friends.keys())))

# Extract edges from graph
for k, v in central_friends.items():
    for item in v:
        if item in central_friends.keys():
            edges.append((k, item))

```
Now add in edges and nodes and use the Louvain method described in
>"Fast unfolding of communities in large networks, Vincent D Blondel, Jean-Loup Guillaume, Renaud Lambiotte, Renaud Lefebvre, Journal of Statistical Mechanics: Theory and Experiment 2008(10), P10008 (12pp)."

Details can be found in [this GitHub repo](https://github.com/taynaud/python-louvain)

```python
# Create the graph.
# Small reminder: friends are the nodes and friendships are the edges here
G = nx.Graph()
G.add_nodes_from([CENTRAL_ID])
G.add_nodes_from(central_friends.keys())

G.add_edges_from(edges)
print('Added {} edges'.format(len(edges) ))

pos = nx.spring_layout(G)

part = community.best_partition(G)

# Get a list of all node ids
nodeID = G.node.keys()
```

```python
# The louvain community library returns cluster ids, we have turn them into colors using color lovers

colors = cl.scales['12']['qual']['Paired']

def scatter_nodes(pos, labels=None, color='rgb(152, 0, 0)', size=8, opacity=1):
    # pos is the dict of node positions
    # labels is a list  of labels of len(pos), to be displayed when hovering the mouse over the nodes
    # color is the color for nodes. When it is set as None the Plotly default color is used
    # size is the size of the dots representing the nodes
    # opacity is a value between [0,1] defining the node color opacity

    trace = Scatter(x=[],
                    y=[],  
                    mode='markers',
                    marker=Marker(
        showscale=False,
        colorscale='Greens',
        reversescale=True,
        color=[],
        size=10,
    line=dict(width=0)))
    for nd in nodeID:
        trace['x'].append(pos[nd][0])
        trace['y'].append(pos[nd][1])
        color = colors[part[nd] % len(colors)]
        trace['marker']['color'].append(color)
    attrib=dict(name='', text=labels , hoverinfo='text', opacity=opacity) # a dict of Plotly node attributes
    trace=dict(trace, **attrib)# concatenate the dict trace and attrib
    trace['marker']['size']=size

    return trace

def scatter_edges(G, pos, line_color='#a3a3c2', line_width=1, opacity=.2):
    trace = Scatter(x=[],
                    y=[],
                    mode='lines'
                   )
    for edge in G.edges():
        trace['x'] += [pos[edge[0]][0],pos[edge[1]][0], None]
        trace['y'] += [pos[edge[0]][1],pos[edge[1]][1], None]  
        trace['hoverinfo']='none'
        trace['line']['width']=line_width
        if line_color is not None: # when it is None a default Plotly color is used
            trace['line']['color']=line_color
    return trace

# Node label information available on hover. Note that some html tags such as line break <br> are recognized within a string.
labels = []

for nd in nodeID:
      labels.append('{} ({})'.format(nd, part[nd],))

trace1 = scatter_edges(G, pos, line_width=0.25)
trace2 = scatter_nodes(pos, labels=labels)

# Configure the plot and call Plotly
width=600
height=600
axis=dict(showline=False, # hide axis line, grid, ticklabels and  title
          zeroline=False,
          showgrid=False,
          showticklabels=False,
          title=''
          )
layout=Layout(title= '',

    font= Font(),
    showlegend=False,
    autosize=False,
    width=width,
    height=height,
    xaxis=dict(
        title='Facebook friend graph',
        titlefont=dict(
        size=14,
        color='#fff'),
        showgrid=False,

        showline=False,
        showticklabels=False,
        zeroline=False
    ),
    yaxis=YAxis(axis),
    margin=Margin(
        l=40,
        r=40,
        b=85,
        t=100,
        pad=0,

    ),
    hovermode='closest',
    paper_bgcolor='rgba(0,0,0,1)',
    plot_bgcolor='rgba(0,0,0,1)'
    )


data=Data([trace1, trace2])

fig = Figure(data=data, layout=layout)
py.iplot(fig, image='png')
```
