# Interview Overview

## Step 1: Requirements Clarification

* Will users of our service be able to post tweets and follow other people?
* Should we also design to create and display the user’s timeline?
* Will tweets contain photos and videos?
* Are we focusing on the backend only, or are we developing the front-end too?
* Will users be able to search tweets?
* Do we need to display hot trending topics?
* Will there be any push notification for new (or important) tweets?

## Step 2: Back of the Envelope Calculations

* What scale is expected from the system (e.g., number of new tweets, number of tweet views, number of timeline generations per sec., etc.)?
* How much storage will we need? We will have different storage requirements if users can have photos and videos in their tweets.
* What network bandwidth usage are we expecting? This will be crucial in deciding how we will manage traffic and balance load between servers.

## Step 3: System Interface Definition

Examples:
```
postTweet(user_id, tweet_data, tweet_location, user_location, timestamp, …)  
generateTimeline(user_id, current_time, user_location, …)  
markTweetFavorite(user_id, tweet_id, timestamp, …)  
```

## Step 4: Define Data Model

Example Entites:
* User: UserID, Name, Email, DoB, CreationDate, LastLogin, etc.
* Tweet: TweetID, Content, TweetLocation, NumberOfLikes, TimeStamp, etc.
* UserFollow: UserID1, UserID2
* FavoriteTweets: UserID, TweetID, TimeStamp

Which Database systems to use?

## Step 5: High-level Design

Draw a block diagram with 5-6 boxes representing the core components of our system. We should identify enough components that are needed to solve the actual problem from end to end.

## Step 6: Detailed Design

Dig deeper into 2-3 major components, depending on what interviewer wants.  We should present different approaches, their pros and cons, and explain why we will prefer one approach over the other.

* Since we will be storing a massive amount of data, how should we partition our data to distribute it to multiple databases? Should we try to store all the data of a user on the same database? What issue could it cause?
* How will we handle hot users who tweet a lot or follow lots of people?
* Since users’ timeline will contain the most recent (and relevant) tweets, should we try to store our data so that it is optimized for scanning the latest tweets?
* How much and at which layer should we introduce cache to speed things up?
* What components need better load balancing?

## Step 7: Identifying and Resolving Bottlenecks

* Is there any single point of failure in our system? What are we doing to mitigate it?
* Do we have enough replicas of the data so that we can still serve our users if we lose a few servers?
* Similarly, do we have enough copies of different services running such that a few failures will not cause a total system shutdown?
* How are we monitoring the performance of our service? Do we get alerts whenever critical components fail or their performance degrades?

# Designing a URL Shortening service Like TinyURL

Let's design a URL shortening service like TinyURL. This service will provide short aliases redirecting to long URLs.
* Simiilar:  bit.ly, ow.ly, short.io

Benefits:
* save a lot of space when displayed, printed, messaged, or tweeted
* Less likely to be mistyped
* used to optimize links across devices
* track individual links to analyze audience
* mesure ad performance
* hide affiliated original URL

## Requirements and Goals of the System

Functional Requirements
1. Given a URL, our service should generate a shorter and unique alias of it. This is called a short link. This link should be short enough to be easily copied and pasted into applications.
2. When users access a short link, our service should redirect them to the original link.
3. Users should optionally be able to pick a custom short link for their URL.
4. Links will expire after a standard default timespan. Users should be able to specify the expiration time.

Non-Functional Requirements:
1. The system should be highly available. This is required because, if our service is down, all the URL redirections will start failing.
2. URL redirection should happen in real-time with minimal latency.
3. Shortened links should not be guessable (not predictable).

Extended Requirements:
1. Analytics; e.g., how many times a redirection happened?
2. Our service should also be accessible through REST APIs by other services.

## Capacity Estimations
Our system will be read-heavy. There will be lots of redirection requests compared to new URL shortenings. Let’s assume a 100:1 ratio between read and write.

**Traffic Estimates:**
* 100:1 Read to write radio
* 500M new URLs per month
* 500M * 100 = 50B redirections per month
* QPS (write) = 500M / (30 * 24 * 3600) = ~200 URLs / sec
* QPS (read) = 100 * 200 = 20K / sec

**Storage Estimates**: 
* Assume storing request and links for 5 years
* 500M * (12 * 5) = 30 billion objects
* Assuming 500 Bytes per object: 30 billion * 500 = 15TB

**Bandwidth Estimates**: For write requests, since we expect 200 new URLs every second, total incoming data for our service will be 100KB per second:
* Write Requests: 200 Urls / s * 500 Bytes = 100KB / Sec
* Read Requests: 100KB * 100 = ~10MB / Sec

**Memory Estimates**: If we want to cache some of the hot URLs that are frequently accessed, how much memory will we need to store them? If we follow the 80-20 rule, meaning 20% of URLs generate 80% of traffic, we would like to cache these 20% hot URLs.
* 20K read requests / sec * 3600 * 24 = ~1.7 Billion read requests / day
* To cache 20%: ~2.0 bill * 0.2 * 500 bytes = 2 bill * 100 bytes = 200 GB of memory

**Summary of Estimates:** Assuming 500 mill new URLs per month and 100:1 read/write ratio:

* New URLs: 200/s
* URL redirections: 20K/s
* Incoming data: 100KB/s
* Outgoing data: 10 MB/s
* Storage for 5 years: 15 TB
* Memory for cache: 170 GB

## System APIs
We can have SOAP or REST APIs to expose the functionality of our service. Following could be the definitions

**Create URL**:
`createURL(api_dev_key, original_url, custom_alias=None, user_name=None, expire_date=None)`

Parameters:
* api_dev_key (string): The API developer key of a registered account. This will be used to, among other things, throttle users based on their allocated quota.
* original_url (string): Original URL to be shortened.
* custom_alias (string): Optional custom key for the URL.
* user_name (string): Optional user name to be used in the encoding.
* expire_date (string): Optional expiration date for the shortened URL.

Returns: (string) with shortened URL. Else error

**Delete URL** 
`deleteURL(api_dev_key, url_key)`

## Database Design
NOTE: Defining the DB schema in the early stages of the interview would help to understand the data flow among various components and later would guide towards data partitioning.

A few observations about the nature of the data we will store:
* We need to store billions of records.
* Each object we store is small (less than 1K).
* There are no relationships between records—other than storing which user created a URL.
* Our service is read-heavy.

We would need two tables: one for storing information about the URL mappings and one for the user’s data who created the short link.

![image](https://user-images.githubusercontent.com/13190696/158417230-4d476125-2a6f-4b69-94fc-0c81da814569.png)

Since we anticipate storing billions of rows, and we don’t need to use relationships between objects – a NoSQL store like DynamoDB, Cassandra or Riak is a better choice.
* DynamoDB: Key-value or document data structures
* Cassandra: wide-column database
* Riak: key-value store

## Basic System Design and Algorithm

Multiple approaches:

### a. Encoding Actual URL
This encoding could be base36 ([a-z ,0-9]) or base62 ([A-Z, a-z, 0-9]) and if we add ‘+’ and ‘/’ we can use Base64 encoding. A reasonable question would be, what should be the length of the short key? 6, 8, or 10 characters?

Using base64 encoding, a 6 letters long key would result in 64^6 = ~68.7 billion possible strings.
Using base64 encoding, an 8 letters long key would result in 64^8 = ~281 trillion possible strings.

Problems:
* Might get duplicate shortened urls for same long URL.

Workaround:
* Could append user ID. But if user isn't logged in, then we would need to request a uniqueness key
* Can append sequence number to url.

![image](https://user-images.githubusercontent.com/13190696/158420862-e6d3a12a-089f-4f5d-b8bb-3097e22857ce.png)

### b. Generating Keys Offline

We can have a standalone Key Generation Service (KGS) that generates random six-letter strings beforehand and stores them in a database (let’s call it key-DB).

**Concurrency Issues?**
* KGS can use two tables to store keys: one for keys that are not used yet, and one for all the used keys. 
* As soon as KGS gives keys to one of the servers, it can move them to the used keys table. KGS can always keep some keys in memory to quickly provide them whenever a server needs them.
* For simplicity, as soon as KGS loads some keys in memory, it can move them to the used keys table.
* KGS also has to make sure not to give the same key to multiple servers. For that, it must synchronize (or get a lock on) the data structure holding the keys before removing keys from it and giving them to a server.

**key-DB size?**: With base64 encoding, we can generate 68.7B unique six letters keys. If we need one byte to store one alpha-numeric character, we can store all these keys in:
6 (characters per key) * 68.7B (unique keys) = 412 GB.

**Isn’t KGS a single point of failure?** Yes, it is. To solve this, we can have a standby replica of KGS. Whenever the primary server dies, the standby server can take over to generate and provide keys.

**How would we perform a key lookup?** We can look up the key in our database to get the full URL. If it’s present in the DB, issue an “HTTP 302 Redirect” status back to the browser, passing the stored URL in the “Location” field of the request. If that key is not present in our system, issue an “HTTP 404 Not Found” status or redirect the user back to the homepage.

**Should we impose size limits on custom aliases?** Our service supports custom aliases. Users can pick any ‘key’ they like, but providing a custom alias is not mandatory. However, it is reasonable (and often desirable) to impose a size limit on a custom alias to ensure we have a consistent URL database. Let’s assume users can specify a maximum of 16 characters per customer key (as reflected in the above database schema).





