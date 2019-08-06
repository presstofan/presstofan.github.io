---
title:  "Creating Facebook friend network graph using Python - Part 3"
date:   2019-08-06 21:54:00 +0100
categories:
  - Python
tags:
  - python
  - facebook
  - social-network
featured_image:
header:
  teaser: assets/images/facebook_graph_teaser_500x300.png
---

In the last post, I have demonstrated how to extract communities from a network. Now, we are going to look at a group of metrics that measure the 'importance' of nodes - Centrality. How do we determine who is 'important' in a network? That will, of course, depend on how we define importance. But broadly speaking, important nodes could be the ones that are 1) well connected, 2) close to everyone, or 3) gatekeepers who control the flow of information. These three definitions which correspond to three types of Centrality, namely Degree Centrality, Closeness Centrality and Betweenness Centrality. I will go through them one by one.

## Degree Centrality

Degree Centrality is simply defined as the total number of direct contact. In directed networks (i.e. the connections between nodes have direction), Degree Centrality can be calculated as In-degree or Out-degree. But for my Facebook network, the connections work both ways hence we will only deal with one Degree Centrality. In addition to a measure of popularity, it can be seen as an index of exposure to what is flowing through the network. For example, in a gossip network, a central actor more likely to hear a given bit of gossip. It can also be interpreted as the opportunity to influence and be influenced directly. Let's calculate the Degree Centrality for each node in my Facebook network and visualise it. Note that I will remove myself from the network for the centrality calculation below since by definition I am connected to everyone in my friend network, which is kind of meaningless. 

```python
# remove myself from the graph
G_f = copy.deepcopy(G) 
G_f.remove_node(CENTRAL_ID)

# keep the position
pos_f = copy.deepcopy(pos) 
pos_f.pop(CENTRAL_ID, None)

# Degree centrality
degree = nx.degree_centrality(G_f)
values = [degree.get(node)*500 for node in G_f.nodes()]

plt.rcParams['figure.figsize'] = [10, 10]
nx.draw_networkx(G_f, pos =pos_f,
                 cmap = plt.get_cmap('Reds'), 
                 node_color = values, node_size=values, 
                 width=0.2, edge_color='grey', with_labels=False)
limits=plt.axis('off') # turn of axisb
```
![Facebook Friends Network Graph Degree](/assets/images/posts_images/190806/facebook_graph_degree.png)

The result is illuminating as it highlights my friends who have the most mutual friends with me. Note that they are not necessarily the people who have the largest number of friends on Facebook as the network is biased towards me. Also, note that the Degree Centrality calculated by `networkx` has been normalised by the total number of possible connections in the network.

## Closeness Centrality

Closeness Centrality is defined as the sum of geodesic distances (shortest path) to all other nodes. It is an inverse measure of centrality as the higher the metric, the further apart the node from other nodes. It can be seen as the index of the expected time of arrival for whatever is flowing through the network for a given node. For example in a gossip network, the central player tends to hear things first.

```python
#  Closeness centrality
close = nx.closeness_centrality(G_f)
values = [close.get(node)*100 for node in G_f.nodes()]

nx.draw_networkx(G_f, pos = pos_f, 
                 cmap = plt.get_cmap('Blues'), 
                 node_color = values, node_size=values,
                 width=0.2, edge_color='grey', with_labels=False)

limits=plt.axis('off') # turn of axisb
```

![Facebook Friends Network Graph Close](/assets/images/posts_images/190806/facebook_graph_close.png)

The graph above shows my friend network with the size of the nodes reflects the Closeness Centrality. Since the sum of distances depends on the number of nodes in the graph, closeness is normalized by the sum of the minimum possible distances n-1. Nothing too interesting here and we would possibly need a bigger more complete network to see the difference.

## Betweenness Centrality

Betweenness Centrality is a measure of how often a node lies along the shortest path between two other nodes. It can be seen as an index of potential for gatekeeping, brokering, controlling the flow, and also of liaising otherwise separate parts of the network. it often indicates power and access to the diversity of whatever flows in the network and the potential for synthesizing. Nodes with high Betweenness Centrality are usually bridges between groups or k-local bridges. 

```python
#  Betweeness centrality
between = nx.betweenness_centrality(G_f)
values = [between.get(node)*500 for node in G_f.nodes()]

nx.draw_networkx(G_f, pos = pos_f, 
                 cmap = plt.get_cmap('Greens'), 
                 node_color = values, node_size=values, 
                 width=0.2, edge_color='grey', with_labels=False)

limits=plt.axis('off') # turn of axisb
```

![Facebook Friends Network Graph Between](/assets/images/posts_images/190806/facebook_graph_between.png)

`networkx` normalises Betweenness Centrality by for a node v by calculating the sum of the fraction of all-pairs shortest paths that pass through v. The few nodes that have been highlighted by this metrics are my friends who belong to more than one communities. For example, they could be my high school friends who also went to the same uni with me. 

There are also other types of Centrality. The [networkx](https://networkx.github.io/documentation/networkx-1.10/reference/algorithms.centrality.html) documentation list a few more. Eigenvector Centrality is a useful one. The idea behind it is that a node must have a high score if connected to nodes that are themselves well connected. Google's PageRank algorithm for their earlier search engine is a variant of Eigenvector Centrality. Feel free to check it out.

That is all for now on social network analysis. Many interesting things haven't been covered here. I will revisit this topic in the future if time permits.