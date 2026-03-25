# Hands-on Labs for Lenses 6.1

## Lab 1 - Lenses UI Orientation

### Login

Using your student lab sheet point your favorite web browser to the URL listed.

`https://studentXX-hq.lenses.training` (replace the XX with your student number)

User name: admin

Password: $tudentXX (replace the XX with your student number)

Now that you're all logged in, the Lenses help screen will pop up. It has helpful links to support and our questions forum. It also has a link to our Community Slack, you should join! But for now just click on Let's Start!

### Navigating Connected Kafka Environments

When you first log in to the UI you are at the HQ level of Lenses. At this level you can work with multiple Kafka clusters at the same time from a single unified interface. You can search for clusters by name or search across all clusters' topics and schemas by keywords. You can also use the Global SQL Studio at this level. You can access these views via the icons down the side of the UI.

Take a moment to explore the sidebar. Notice how all your connected Kafka environments are visible from a single pane of glass. The HQ federates calls to Agents sitting close to each cluster, but the data stays local.

### Browsing and Searching Topics

Go ahead and click on the Topics link. Here we can see all the topics in all the clusters. These topics are fully searchable.

Scroll over to the right to see all the different fields in the topics you can see.

You can browse and drill down into individual topics here as well as search topics and schemas by keyword.

**Scenario:** You're helping a coworker build a status dashboard for company executives. He told you that executives like dashboards with maps so he's looking for data streams with location data in them to use for his dashboard.

Let's do a few searches to figure out which topics have location data for him to use. In the search bar for topics type in the keyword "longitude". You'll notice that topics come up that don't have "longitude" in their title. That's because Lenses searches inside the schemas as well. Click on Search in Schema to bring up the actual key names that Lenses surfaced.

Now you can point your coworker quickly in the right direction for his executive dashboards.

### Using SQL Studio for Ad-Hoc Topic Querying

Lenses comes with a SQL Studio designed to allow you to explore and manipulate your topics all from one central location. Hover over the `nyc_yellow_taxi_trip_data` topic and click on the SQL button. This will open up that topic in the SQL Studio.

Lenses will automatically run a simple search that surfaces the last 10 events from each partition in the topic. You can view the events in Grid form, but you can also toggle to List to see them more in a JSON style format. Take a look at the different events in the list below.

Now we're going to craft a search that focuses on just the vendors, their fares, and what type of payment was used.

Delete the search that was initially run and type in the following.

```sql
SELECT VendorID, fare_amount, payment_type
FROM nyc_yellow_taxi_trip_data
```

Once you have it typed in, click on the time range picker on the right hand side and click on the "Last 5 Minutes" selection. This will automatically add a time range limiter to your search and run the search.

Take note of the autocomplete feature as you type. SQL Studio will suggest topic names, field names, and SQL keywords. Use it aggressively, it saves a lot of time.

### Viewing Schemas, Configurations, and Connected Consumers

Before we move on, let's explore what else you can see from the topic view. Navigate back to Topics and drill down into the `nyc_yellow_taxi_trip_data` topic.

From the topic detail view you can see:

- **Schema**: The Avro/JSON/Protobuf schema attached to the topic (if any). This is searchable from the HQ level, which is how our "longitude" search worked earlier.
- **Configuration**: The topic-level Kafka configs like retention, partitions, replication factor, etc.
- **Consumers**: Any consumer groups currently connected to this topic and their lag.

Spend a few minutes clicking through these tabs to get a feel for the information available.

---

## Lab 2 - SQL Snapshot Deep Dive

In this lab we're going to dig into SQL Snapshot, the point-in-time query engine in Lenses. Remember from the lecture: SQL Snapshot behaves like a Kafka consumer. By default it starts reading from the oldest event in the topic and works its way forward. This matters when you're working with large topics.

### Craft a Needle-in-a-Haystack Search

A needle-in-a-haystack search is when you know exactly what you're looking for. You have a specific record, ID, or value in mind and you want to find it.

**Scenario:** You are building an application that looks for taxi fare fraud in the system. Let's use SQL Studio to search for events that could be fraudulent. For example a trip with a distance of 0 but a fare amount greater than 0.

Open up `nyc_yellow_taxi_trip_data` in SQL Studio and craft the following search:

```sql
SELECT *
FROM nyc_yellow_taxi_trip_data
WHERE trip_distance = 0 AND fare_amount > 0
LIMIT 100;
```

Hit run and take a look at the results. Looks like there's a lot of possible fraud for our new application to surface!

Notice a few things about this query:

1. We used a specific WHERE clause to target exactly the records we wanted. This is the needle-in-a-haystack pattern.
2. We used `LIMIT 100` to control the result set size. Remember, the default LIMIT is 10,000 but we don't need that many to prove our point.

### Craft a Broad-Based Statistical Search

A broad-based search is when you want to understand the shape of your data. You're not looking for a specific record, you're looking at aggregates and patterns.

Let's run a broader query to understand the distribution of payment types across our taxi data:

```sql
SELECT payment_type, count(*) as total
FROM nyc_yellow_taxi_trip_data
GROUP BY payment_type
```

This gives us a statistical view of the data, which payment types are most common.

### Working with SQL Snapshot Limits

Now let's see those built-in limits in action. Remember, SQL Snapshot is a Kafka consumer and it has limits that prevent runaway queries.

Try running a query without a LIMIT clause on a larger scan:

```sql
SELECT *
FROM nyc_yellow_taxi_trip_data
WHERE fare_amount > 50
```

Watch what happens. The query will eventually stop, either because it hit the default `max.size` (20MB), `max.query.time` (1 hour), `max.idle.time` (5 seconds of no new records), or the default `LIMIT` of 10,000 records.

You can override these limits with SET statements. Try increasing the max size:

```sql
SET max.size = '100m';

SELECT *
FROM nyc_yellow_taxi_trip_data
WHERE fare_amount > 50
LIMIT 500;
```

Here's the full list of limits you can adjust:

| Limit | Default | Example |
|-------|---------|---------|
| `max.size` | 20MB | `SET max.size = '1g';` |
| `max.query.time` | 1 hour | `SET max.query.time = '60000ms';` |
| `max.idle.time` | 5 seconds | `SET max.idle.time = '5s';` |
| `LIMIT N` | 10,000 | `SELECT * FROM payments LIMIT 100;` |

We have a comprehensive guide to the Lenses SQL Studio [here in the docs](https://docs.lenses.io/latest/user-guide/sql-studio). It's quite powerful and can even be used to create new topics based on SQL output.
