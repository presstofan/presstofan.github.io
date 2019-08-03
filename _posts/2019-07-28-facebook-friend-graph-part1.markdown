---
title:  "Creating Facebook friend network graph using Python - Part 1"
date:   2019-07-28 12:30:00 +0100
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

I have recently been asked to generate a social network graph to show how friends on Facebook are connected. This type of network graph can be used to identify communities as well as the gatekeepers of each community in the network, the value of which has been explained in the iconic paper [The Strength of Weak Ties](https://www.jstor.org/stable/2776392?seq=1#page_scan_tab_contents) by Mark Granovetter. [Here](https://www.forbes.com/sites/jacobmorgan/2014/03/11/every-employee-weak-ties-work/) is a more accessible article about this idea.

It used to be much easier to generate graphs like this (through some 3rd part apps or the Facebook API) but Facebook is getting increasingly strict about what data can be accessed through the API. The `all_mutual_friends`, `mutual_friends`, and `three_degree_mutual_friends` context edges of the Facebook Social Context API were deprecated on April 4, 2018, and immediately started returning empty data sets. They have now been fully removed. This post presents an alternative way to retrieve Facebook friend network with the help of Python and Selenium Webdriver.

## Step 1: Set up Selenium ChromeDriver

In order to make the script work, we will need to set up Selenium ChromeDriver first. ChromeDriver is a web automation framework that allows you to control the behaviours of the Chrome browser using a programming language (Python in this case). I found [this guide from Chris Kenst](https://www.kenst.com/2015/03/installing-chromedriver-on-mac-osx) particularly useful for making the ChromeDriver work on macOS.

## Step 2: Set up the helper functions

Once making sure the ChromeDriver, the following script by [Lucas Allen](https://github.com/lgallen/twitter-graph) and modified by [Eliot Andres](https://github.com/EliotAndres/facebook-friend-graph) can be used to scrap the friend network. First, we will need to load a couple of useful packages. Note that I have used Python 3 to build the script below. You might need to set up a virtual environment to make it works. I would recommend [Pipenv] (https://docs.pipenv.org/en/latest/) for future-proofing your virtual environment management workflow. 

Below are a couple of key modules that we will use in the script:

* `html.parser`: for parsing HTML
* `webdriver`: for controlling the ChromeDriver
* `tqdm`: for showing a progress bar while scrapping
* `pickle`: for backing up the checkpoint and storing the scraped friend network
* `getpass`: for the login prompt

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
Now, we will need to code up several functions that help to navigate through Facebook pages. Facebook pages are not static and many of them are using AJAX (Asynchronous JavaScript And XML), which allows web pages to be updated asynchronously by exchanging data with a web server behind the scenes. This means that it is possible to update parts of a web page, without reloading the whole page. For example, when visiting the friend page, only the first few friends are listed. But more friends will be loaded once the user scrolls down to the bottom of the page. The `get_fb_page` function does exactly that. Given a page, the function will determine the height of the body to calculate the scrolling distance, scroll down to the bottom, wait for a while for the AJAX to load new content, calculate new scrolling distance and scroll down again. It will stop when no new content can be loaded.

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

## Step 3: Execute the scrapping plan

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

The ChromeDriver will open a new Chrome instance and log in Facebook. Once successfully log in. Execute the codes below to get the URLs to your friends' homepage. It will also store the friend list into a Pickle object `uniq_urls.pickle`, so we don't need to replace the same process when a network failure occurs. As we are not using a well-supported API, this can happen quite often so be prepared.

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

Next, we will use the URLs collected above to access the mutual friend page of each friend and retrieve the list of mutual friends. This is a slightly tricky part as Facebook are not happy with repetitive activities even you check your own friend list. It could temporarily ban you from accessing friend pages if you request more than 100 pages in one go. As such, the script is set to pause for half an hour at every 100 pages. If you are still uncomfortable with this rate, I would suggest further reduce the rate and leave the script running overnight. It will get the job done eventually.

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
It will take a while to retrieve all the friends but when it is done, friend names and their connections should be stored in the pickle file `friend_graph.pickle`.  In the next post, we will have some fun analysing the friend network.