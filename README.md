# Instagram System Design (feed + comments + likes)

## Use cases

### We'll scope the problem to handle only the following use cases

* **User** creates a post
    * **Service** pushes posts to followers
* **User** views the home timeline (activity from people the user is following)
    * **User** sees the post's metadata alongside the post: author's name, profile pic number of likes and comments
* **User** puts a like to the post
* **User** puts a comment to the post
* **Service** has high availability


### Constraints and assumptions

* Every user has a predefined list of followed/following users.

#### State assumptions

General

* Traffic is not evenly distributed
* Publishing posts should be fast
    * Fanning out a post to all of your followers should be fast unless you have millions of followers
* 1 billion active users
* **Every post receives 50 likes**
* **Every post receives 5 comments**
* 500 million posts per day or 15 billion posts per month
    * Each post averages a fanout of 50 deliveries
    * 5 billion total posts delivered on fanout per day
    * 150 billion posts delivered on fanout per month
* 250 billion read requests per month
* 10 billion searches per month

Timeline

* Viewing the timeline should be fast
* Putting a like or a comment should be fast
* Instagram is more read heavy than write heavy
    * Optimize for fast reads of posts
* Ingesting posts is write heavy
* The timeline is chronological


------------------#### Calculate usage------------------

* Size per post:
    * `tweet_id` - 8 bytes
    * `user_id` - 32 bytes
    * `text` - 140 bytes
    * `media` - 10 KB average
    * Total: ~10 KB
* Size per comment:
* Size per user:
* Monthly size of content:

* 
* 150 TB of new tweet content per month
    * 10 KB per tweet * 500 million tweets per day * 30 days per month
    * 5.4 PB of new tweet content in 3 years
* 100 thousand read requests per second
    * 250 billion read requests per month * (400 requests per second / 1 billion requests per month)
* 6,000 tweets per second
    * 15 billion tweets per month * (400 requests per second / 1 billion requests per month)
* 60 thousand tweets delivered on fanout per second
    * 150 billion tweets delivered on fanout per month * (400 requests per second / 1 billion requests per month)
* 4,000 search requests per second
    * 10 billion searches per month * (400 requests per second / 1 billion requests per month)

Handy conversion guide:

* 2.5 million seconds per month
* 1 request per second = 2.5 million requests per month
* 40 requests per second = 100 million requests per month
* 400 requests per second = 1 billion requests per month
-------------------------------------------------------------------------------------

### Storage

Overall, there might be a need to see all likes and comments of a certain user, 
and in the meantime be able to see likes and comments on a certain post, which 
brings the logic identical to the following/followed mechanics. Therefore, the suggestion 
is to store Users, Posts, and Comments as *nodes* in GraphDB, while using such relations as "Created" and "Liked" as edges.
All metadata, such as post content, or user pic, can be stored in NoSQL DB, since having 15b monthly posts + 5 comments 
on each post might be problematic for SQL Dbs.
https://graphaware.com/neo4j/2014/07/23/node-degrees-neo4j.html


## Step 2: A high-level design

![High Level Architecture](https://i.imgur.com/soAucVL.png)

![NoSQL Data Layout](https://i.imgur.com/5WOfXTE.png)

![Graph DB Data Layout](https://i.imgur.com/W0FaPdi.png)

### Use case: User creates a post

**Request:**

```
$ curl -X POST https://instagram.com/api/v1/post?user_id=123 -F "image=@/home/user1/Desktop/test.jpg"
```

**Response:** 

```
201 Created
```

The data flow:
* The **Web Server** forwards the request to the **Write API** server
* The **Write API** contacts the **Post Service** which does the following:
    * Links **User** and the newly created post in Graph DB
    * Stores **Comment** metadata in Document NoSQL DB.
    * Stores media in the **Object Store**
    * Ensures consistency between all storage
* The **Write API** contacts the **Fan Out Service**, which does the following:
    * Queries the **Graph Cache Service** to find the user's followers stored in the **In-Memory Graph Cache**
    * Stores the post id in the *home timeline of the user's followers* in a **Memory Cache**

### Use case: User views the home timeline

**! It's important to mention, that the approach below can be split into several steps, such as returning only post metadata first 
and later counting likes/comments alongside retrieving the author's metadata. This should be decided after some more thorough tests**

Request:

```
curl https://instagram.com/api/v1/feed?user_id=123"
```

Response:

```
{
    feed: [
      {
        "id": "post_id_1",
        "timestamp": "timestamp",
        "author": {
          "id": "id",
          "name": "name",
          "prifilePiCUrl": "url"
        },
        "likesNumber": "10",
        "commentsNumber": "10",
        "text": "text" 
      },
      {"id": "post_id_2"},
        ...
      {"id": "post_id_3"},
    ],
    "nextPage": "URL",
    "prevPage": "URL",
}
```

* The **Web Server** forwards the request to the **Read API** server
* The **Read API** server contacts the **Timeline Service**, which does the following:
    * Gets the timeline data stored in the **Memory Cache**, containing post ids
    * Queries the **Post Service** to obtain author's ids and number of likes/comments
        * Number of **Likes** and **Comments** are retrieved from **Graph Cache Service**
        * **Post** metadata is retrieved from Document NoSQL DB
    * Queries the **User Service** to obtain additional info about the author (NoSQL), such as name and profile pic URL


### Use case: User puts a like on a post
**Request:**

```
$ curl -X POST https://instagram.com/api/v1/post/{id}/likes"
$ curl -X POST https://instagram.com/api/v1/post/{id}/unlikes"
```

**Response:** 

```
201 Created
```

The data flow:
* The **Web Server** forwards the request to the **Write API** server
* The **Write API** contacts **Post Service** which does the following:
    * Stores the relation **Liked** between **User** and a **Post** in the **Graph DB**.


### Use case: User comments on a post

**Request:**

```
$ curl -X POST https://instagram.com/api/v1/post/{id}/comment -d "<comment>"
```

**Response:** 

```
201 Created
```

The data flow:
* The **Web Server** forwards the request to the **Write API** server
* The **Write API** contacts **Comment Service** which does the following:
    * Stores the relation **Created**  between **User** and **Comment** in the persistant **Graph DB**.
    * Stores the relation **Received** between **Post** and **Comment** in the persistant **Graph DB**.
    * Stores the comment metadata in Document NoSQL DB.
* **Comment Service** ensures the consistency between NoSQL DB and Graph DB.

### Use case: User reads comments of a post

**Request:**

```
$ curl -X GET https://instagram.com/api/v1/post/{id}/comments
```

**Response:** 

```
{
    comments: [
      {
        "id": "comment_id_1",
        "timestamp": "timestamp",
        "author": {
          "id": "id",
          "name": "name",
          "prifilePiCUrl": "url"
        },
        "text": "text" 
      },
      {"id": "comment_id_2"},
        ...
      {"id": "commment_id_3"},
    ],
    "nextPage": "URL",
    "prevPage": "URL",
}
```

The data flow:
* The **Web Server** forwards the request to the **Read API** server
* The **Read API** contacts **Comment Service** which does the following:
    * Retrieves **Comment** Metadata from own NoSQL db.
    * Retrieves **User** Metadata via **User Service**
