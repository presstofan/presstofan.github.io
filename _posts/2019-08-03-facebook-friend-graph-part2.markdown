---
title:  "Creating Facebook friend network graph using Python - Part 2"
date:   2019-08-03 22:22:00 +0100
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

In the last post, we have successfully retrieved our friend network on Facebook. Now, the fun part begins. We will use the data retrieved to generate a network graph and run some network analyses.

## Step 1: Load packages and data

Below are a couple of key modules we will use in the script:

* `community`: implements community detection by using the louvain method
* `networkx`: for the creation, manipulation, and study of the structure, dynamics, and functions of complex networks
* `pickle`: for loading the pickle object that stores the friend network

```python
# Setting for plotting graph in the Jupyter Notebook
%matplotlib inline
import matplotlib.pyplot as plt
# Set the size of the graph
plt.rcParams['figure.figsize'] = [10, 10] 

import community
import networkx as nx
import colorlover as cl
import numpy as np
import pickle

py.init_notebook_mode(connected=True)

# This is your Facebook id. It can also be a number
CENTRAL_ID = 'FACEBOOK_ID'

# This is the pickle file containing the raw graph data
GRAPH_FILENAME = 'friend_graph.pickle'

# Load the friend_graph picklefile
with open(GRAPH_FILENAME, 'rb') as f:
    friend_graph = pickle.load(f)
```

`friend_graph` is a dictionary of lists in the form of {'friend1': ['friend2, 'friend3']}, where the keys are each of your friends and the value lists have all your mutual friends.

## Step 2: Clean the data and reshape it to a suitable network data structure

Before plotting and analysing the network, we need to do some pre-work. First, we may want to get rid of the friends which whom we do not have any mutual friend. After all, we are interested in groups and communities so friends without mutual friends do not add much value. Of course, you can skip this step if you want to keep them.

```python
# Only keep friends with at least 2 common friends
central_friends = {}

for k, v in friend_graph.items():
    # This contains the list number of mutual friends.
    # Doing len(v) does not work because ometimes instead of returning mutual
    # friends, Facebook returns all the person's friends
    intersection_size = len(np.intersect1d(list(friend_graph.keys()), v))
    if intersection_size > 2:
        central_friends[k] = v
        
print('Firtered out {} items'.format(len(friend_graph.keys()) - len(central_friends.keys())))
```

When storing network data, two types of data structure are commonly used: adjacency matrix and edge list. An adjacency matrix is a square matrix used to represent a finite graph. The elements of the matrix indicate whether pairs of vertices are adjacent or not in the graph. In our case, the vertices are friends and the elements of the matrix represent friendship ties. Edge list, as its name suggests, contains a list of all the edges in the network. Edge List can be more efficient for storing data when the network is large and sparse. We will convert the `friend_graph` object to an edge list.

```python
# Extract edges from graph

edges = []
nodes = [CENTRAL_ID]

for k, v in central_friends.items():
    for item in v:
        if item in central_friends.keys() or item == CENTRAL_ID:
            edges.append((k, item))
```

## Step 3: Create a network object and visualise the network

Now, we will use the `networkx` module to create a network object out of the edge list.

```python
G = nx.Graph()
G.add_nodes_from([CENTRAL_ID])
G.add_nodes_from(central_friends.keys())

G.add_edges_from(edges)
print('Added {} edges'.format(len(edges) ))
```

The `nx.draw_networkx` function can then be called to draw the graph. It use `matplotlib` under the hood. The default plot has axis so don't forget to remove them.

```python
nx.draw_networkx(G, with_labels=False, node_size=15, width=0.3, node_color='blue', edge_color='grey')
limits=plt.axis('off') # remove axis
```
Below is the network graph generated. Blue dots (call "nodes") are friends and the lines (called "edges") are friendship ties. Since this is my Facebook friend network, everyone is connected to me (the central node). This visualisation uses force-directed positioning function as default. Imagine there are two types force: 1) spring-like attractive forces (based on Hooke's law) which attract pairs of endpoints of the graph's edges towards each other and 2) simultaneously repulsive forces (like those of electrically charged particles based on Coulomb's law) which separate all pairs of nodes. In equilibrium states for this system of forces, the edges tend to have uniform length (because of the spring forces), and nodes that are not connected by an edge tend to be drawn further apart (because of the electrical repulsion).

![Facebook Friends Network Graph](/assets/images/posts_images/190803/facebook_graph.png)

The visualisation is already quite illuminating as it shows the different communities in my friend network. I can roughly tell where their boundaries are. However, let's be more precise and use a mathematical algorithm to do the job.

## Step 4: Detect communities

In this step, we'll colour the different friend communities. We'll use the louvain method described in Fast unfolding of communities in large networks, Vincent D Blondel, Jean-Loup Guillaume, Renaud Lambiotte, Renaud Lefebvre, Journal of Statistical Mechanics: Theory and Experiment 2008(10), P10008 (12pp). Codes are [here](https://github.com/taynaud/python-louvain) and the details of the method can be found in [this Wikipedia page](https://en.wikipedia.org/wiki/Louvain_modularity).

![Facebook Friends Network Graph](/assets/images/posts_images/190803/facebook_graph_comm.png)

6 communities have emerged and they correspond to my friends from school, undergraduate, postgraduate and work. I have removed all the labels for privacy but you could add them in yourselves. You might find a couple of interesting links between groups of friends that you were not aware of before! 

That's all for today. In the next post, we will continue exploring the network by a couple of useful measures of centrality.