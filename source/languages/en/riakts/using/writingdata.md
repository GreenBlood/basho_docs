---
title: Writing Data to Riak TS
project: riakts
version: 1.0.0+
document: guide
toc: true
index: true
audience: beginner
---

[activating]: https://www.docs.basho.com/riakts/1.0.0/using/activating
[planning]: http://docs.basho.com/riakts/1.0.0/using/planning

Now that you've [planned][planning] and [activated][activating] your Riak TS table, you are ready to write data to it.


##Writing Data

Riak TS allows you to write multiple rows of data at a time. To demonstrate, we'll use the example table from earlier:

```sql
CREATE TABLE GeoCheckin
(
   myfamily    varchar   not null,
   myseries    varchar   not null,
   time        timestamp not null,
   weather     varchar   not null,
   temperature double,
   PRIMARY KEY (
     (myfamily, myseries, quantum(time, 15, 'm')),
     myfamily, myseries, time
   )
)
```

To write data to your table, put the data in a list:

```ruby
client = Riak::Client.new 'myriakdb.host', pb_port: 10017
submission = Riak::TimeSeries::Submission.new client, "GeoCheckin"
submission.measurements = [["family1", "series1", 1234567, "hot", 23.5], ["family2", "series99", 1234567, "windy", 19.8]]
submission.write!
```

```erlang
{ok, Pid} = riakc_pb_socket:start_link("myriakdb.host", 10017).
riakc_ts:put(Pid, "GeoCheckin", [[<<"family1">>, <<"series1">>, 1234567, <<"hot">>, 23.5], [<<"family2">>, <<"series99">>, 1234567, <<"windy">>, 19.8]]).
```

```java
RiakClient client = RiakClient.newClient(10017, "myriakdb.host");
List<Row> rows = Arrays.asList(
    new Row(new Cell("family1"), new Cell("series1"), 
            Cell.newTimestamp(1234567), new Cell("hot"), new Cell(23.5)),
    
    new Row(new Cell("family2"), new Cell("series99"),
            Cell.newTimestamp(1234567), new Cell("windy"), new Cell(19.8)));

Store storeCmd = new Store.Builder("GeoCheckin").withRows(rows).build();
client.execute(storeCmd);
```

>**Note on validation**:
>
>Riak TS 1.0.0 does not have client-side insertion validation. Please take caution when creating data to insert by ensuring that each row’s cells match the order/types of the table, and that you do not create a null-value cell for a non-nullable column.

If all of the data are correctly written the response is: `ok` in Erlang, and will not raise an error in Ruby.

If some of the data failed to write, an `RpbErrorResp` error occurs with a number that failed. In the event that your write fails, you should:

1. Do a binary search with half the data, then the other half, and etc. to pinpoint the problem; or
2. Check the data one at a time until one fails.
 

###Guidelines

* In the absence of information about which columns are provided, a write will assume that columns are in the same order they've been declared in the table.
* Timestamps should be in UTC milliseconds.
* Only ASCII is supported.


##Error conditions

There are two error conditions:

* Writing data to a TS table that doesn’t exist, or
* Writing data which doesn’t match the specification of the TS table.