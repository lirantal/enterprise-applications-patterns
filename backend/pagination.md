# Pagination

Pagination is a pattern applied when handling lists of data items that need to be inspected beyond a limited view or session. Possibly its most popular manifestation is through frontend engineers whom are building table list views and enable a paged navigation for the list items with the ability to go through pages.

Pagination can either be implemented on the client-side, or on the server-side:

- On the client-side - by receiving the entire data set and implementing an in-browser pagination through the data. This methodology may also be split to two implementation details - a virtualized pagination which optimizes the work on the DOM by only rendering the visible part, or a more naive approach which renders the entire data set onto the DOM and show/hides the data as the user navigates through it.
- On the server-side - by receiving input from the client on which part of the data set is requested, a server-side implementation can return only a small subset of the data instead of returning all of it which is usually undesired.
  Pagination however, is not a sole requirement of frontend consumers. If an API can be queried by any consuming service, frontend or backend, to return a list of data items which is potentially large, a pagination is used to limit the request and its load on the server.

Exploring a use-case: return a list of all users from the database through an API call to `GET /users`.

- The users data set can be potentially large, spanning millions or billions of users.
- The users data set items may be a large object and not just an array of ids, which will incur a large response payload.
- Transferring all the data at once will take time, and possibly incur a heavy resources hit on the server such as maintaining memory objects to hold the data even if the data is streamed from a database backend using result-set cursors.
- Transferring all the data at once means that if a connection error occurs or any other error impacting either the consuming client or the backend service, the request is largely unusable.

A different and more conservative approach is to set a top limit on the backend service for the amount of items that can be requested in a single call and the consuming client can paginate through the results. The client may need to send bulk requests in order to gather all the data, and depending on how pagination is implemented, it may be able to send them in parallel.

## Indexing

The importance of indexing as layer out by Yahoo engineering presentation titled _Efficient Pagination Using MySQL_ at Percona conf 2009:

Assuming index is set: KEY a_b_c (a, b, c)

- ORDER may get resolved using Index
  – ORDER BY a
  – ORDER BY a,b
  – ORDER BY a, b, c
  – ORDER BY a DESC, b DESC, c DESC

- WHERE and ORDER both resolved using index:
  – WHERE a = const ORDER BY b, c
  – WHERE a = const AND b = const ORDER BY c
  – WHERE a = const ORDER BY b, c
  – WHERE a = const AND b > const ORDER BY b, c

- ORDER will not get resolved uisng index (file sort)
  – ORDER BY a ASC, b DESC, c DESC /_ mixed sort direction _/
  – WHERE g = const ORDER BY b, c /_ a prefix is missing _/
  – WHERE a = const ORDER BY c /_ b is missing _/
  – WHERE a = const ORDER BY a, d /_ d is not part of index _/

reference: https://www.percona.com/files/presentations/ppc2009/PPC2009_mysql_pagination.pdf and video for it [Efficient Pagination Using MySQL - YouTube](https://www.youtube.com/watch?v=B6iNuEuD-gc)

Same technique is also referenced here: [We need tool support for keyset pagination](https://use-the-index-luke.com/no-offset)

Also referred here: [What is the best way to paginate results in SQL Server - Stack Overflow](https://stackoverflow.com/questions/109232/what-is-the-best-way-to-paginate-results-in-sql-server) and the seek method or keyset pagination: [Faster SQL Pagination with jOOQ Using the Seek Method – Java, SQL and jOOQ.](https://blog.jooq.org/2013/10/26/faster-sql-paging-with-jooq-using-the-seek-method/)

## Offset Pagination

An offset pagination is the simplest and most naive approach to implement on the server-side and has been made popular due to databases like MySQL with a LIMIT and OFFSET constructs built-in to the database engine.

In offset pagination a client will call an API that will return the data set and details about the pagination such as the total amount of items, the last page number, etc. The client can then make a subsequent call, specifying the page number and amount of items it is expecting to be returned.

Server-side offset pagination is suited for small data sets encompassing tens of thousands (<=~100k) due to the fact that the database server has to actually read through all the rows up until the offset, discard it, and fetch the rows up until the expected limit.

A practical example that illustrates the problem when using the EXPLAIN directive to explore the query execution plan:

```
mysql> EXPLAIN SELECT * FROM message
 ORDER BY id DESC
 LIMIT 10000, 20\G
***************** 1. row **************
 id: 1
 select_type: SIMPLE
 table: message
 type: index
 possible_keys: NULL
 key: PRIMARY
 key_len: 4
 ref: NULL
 rows: 10020
 Extra:
1 row in set (0.00 sec)
```

As can be seen, the database is scanning through the entire table (based on an index) in order to reach the expected amount of rows, far from an efficient way to paginate through data on a large scale.

Page drifting is another issue that happens with offset pagination. Consider the following example - there is a total of 20 items in the data set and the client requests the first set of items limited to 10 results (`GET /users?offset=0&limit=10`). Before a second client request, item number 10 is removed. The second client request to get the next 10 items (`GET /users?offset=10&limit=10`) will effectively skip the original 10th item that now drifted back to the 9th place.

### Drawbacks

- **Performance** - Working on datasets beyond the 100k roughly will not perform well. If the offset is very large and the data that needs to be fetched to reach it may not fit into memory then this may cause the database server to hit the disk and incur even more I/O costs.
- **Data Integrity** - While browsing through pages, items in the data set may be added, removed or updated, which will cause them to be skipped or viewed twice, often referred to as page drifting.

### When to use?

- When it is absolutely required to provide a full page navigation, allowing to skip through pages and loading a specific page of the dataset on-demand.
- When the dataset is relatively small

## Continuation Token Pagination

An evolution to the naive offset pagination is the **Keyset Pagination** (often referred to as the Seek Method Pagination). It addresses requirements of a scalable pagination solution on the server-side that is performant to the level of constant time, and addresses the issues of page drifting to return an up to date stream of data regardless of changes being made on items in the data set.

> In practice, there is actually no such type of pagination called Continuation Token, but rather the idea is to emphasize the use of opaque continuation tokens which encapsulate an implementation detail on the server-side, be it keyset pagination, seek pagination or other types.

With keyset pagination, we rely on filtering results entirely based on the _where_ clause which will always receive a new item to filter by, that represents the records to fetch after or before the previous data set (depending on whether the sort order is ascending or descending).

A pre-requisite for being able to filter reliably on the data set is having a uniquely comparable index (such as an incremental integer-based primary key on a table).

### Keyset Pagination Flow

A review of a basic flow in practice for a keyset pagination, with an assumption that the data set should be sorted by the timestamp of the last updated record in an ascending order:

1. A client makes a request to get the list of users (`GET /users`)
2. As no filters have been specified, the server queries for the data based on defaults which is sorting by the timestamp in ascending order and returning a data set limited to 10 items.
3. Before the server returns the response, it checks what is the last item's values timestamp and id (referred to later as `timestampColumn`, and `idColumn`) and creates a token out of them that will be used by the client to request the next page. That token may simply be `1393027200_9` where the values are plainly concatenated using an underscore delimiter. It is however desired to not expose this internal implementation detail to the client and so it is recommended to encode the token with base64 so that it is generally referred to as an opaque token - effectively this is our continuation token. A practical response by the server may then look like:

```json
{
  "data": [
    {
      "id": 1,
      "username": "emmanuelgoldstein"
    }
  ],
  "metadata": {
    "nextToken": "1393027200_9"
  }
}
```

4. When the client needs to fetch the next page of the dataset it simply sends the token it received: `GET /users?nextToken=1393027200_9`
5. The server implements the database lookup to retrieve the data set using the following query, which first compares that the timestamp column is after the filtered timestamp to match against (being 1393027200). In cases, where two items or more may have the same timestamp value we further filter and seek the items with their id column that are after id 9. In practice, the query will look as follows:

```
WHERE
  timestampColumn >= T
  AND (
	(timestampColumn > T)
	  OR
	((timestampColumn = T) AND (timestampColumn > idColumn))
  )
  AND (timestampColumn < now())
```

After retrieving the results the server then responds with another continuation token for the next set, and so on.

### Drawbacks

- **Implementation difficulty** - Not as straight-forward to implement as offset pagination which is offloaded to the database to handle, but even as moderate implementation difficulty it may will require more custom logic handling to build the paginated queries unless an ORM or other database library provides support.
- **Data Integrity** - Items may be delivered multiple times, as they incur changes and will be fetched by next pages. This may be a desired outcome, but nonetheless should be taken into account.

### When to use?

- When performance is important and the data set is relatively large, spanning about or more than hundreds of thousands of records (>=100k items in the data set)
- When data integrity is important and it's required to return a reliable data set without skipping any record or returning it twice.

## FAQ

- **What about showing the total numbers of items?**
  Maintain a cachable and in-accurate count of the total number of items that may be updated every so often. Question the requirement and ask _does the user care if there are 6371 records 6102?_. Consider the following as a message to show the user: _showing 150 items out of thousands_.

- **What level of navigation is possible with continuation tokens?**
  It is possible to provide a _next page_ and _previous page_ navigation links based on items that were seen last.

## Resources

This guide is largely based on examples and insights presented in Markus and Philip's posts mentioned in the below resources and it is highly encouraged to review them for a more elaborate explanation and details of the keyset implementation and its impact:

1. [Markus Winand's post on no offsets](https://use-the-index-luke.com/no-offset)
2. [Philip Phauer's blog on continuation tokens](https://blog.philipphauer.de/web-api-pagination-timestamp-id-continuation-token)
3. [Yahoo's Efficient Pagination Using MySQL presentation at Percona conf 2009](https://www.percona.com/files/presentations/ppc2009/PPC2009_mysql_pagination.pdf)
