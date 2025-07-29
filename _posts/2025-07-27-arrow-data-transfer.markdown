---
layout: post
title:  "Exploring Arrow for HTTP Services in Data-Intensive Applications"
date:   2025-07-27 11:29:21 +0100
categories: intro
---

Maybe this title is a bit ambiguous. It is a bit hard to explain in a few words something that, for me at least, felt quite complex some weeks ago. This project started with a simple idea: I wanted to build an HTTP service on top of DuckDB. In my work in Data Engineering and Data Architecture, I feel like building REST APIs has never been the meat of what I did. I know how a REST API works and, obviously, I've used quite a lot of them. 

When I started this project, I had my mind open to explore whatever interested me the most. At first, I thought I could build a query service on top of a remote DuckDB instance, i.e., you run DuckDB on a server and expose a REST endpoint that allows you to do queries. I did this in Julia (using the Genie framework) to explore a different language. Unfortunately, I found the Julia community and docs a bit underwhelming compared to Python... maybe something to come back to in the future.

While re-building the service with FastAPI in Python, I was confronted with "the" question: how should I transfer the results to the client? Of course, there is already a standard in databases, ODBC and JDBC. However, I was building my own endpoints and tackling this as a REST kind of service. The standard for data transferring in REST services has for a while been JSON (or XML if that's your vibe). JSON is very nice because it is normally very readable (except for recursive JSON), being a text format is also very easy to debug, and it has great support across languages. On the flip side, JSON has an overhead of serialization and deserialization, it is not very compact (even though you can use compression before sending over the network), and is generally not thought through for transferring very large amounts of data.

## Welcome Arrow IPC
Before jumping into what Arrow IPC is, I should introduce Apache Arrow for those who don't know it. Apache Arrow is an open-source project that specifies a column-oriented memory format that is language agnostic ([more here](https://arrow.apache.org/docs/index.html)). You may or may not know Arrow, but there is a big chance that if you are using some modern data tooling, Arrow is there somewhere. Some examples are:
- [Polars](https://pola.rs/) a modern DataFrame framework written in Rust that can consume and produce Arrow data often with zero-copy operations.
- [PySpark](https://spark.apache.org/docs/latest/api/python/tutorial/sql/arrow_pandas.html) uses Arrow to exchange data between the JVM and other Python processes (and vice versa) as fast and efficiently as possible.
- [DuckDB](https://duckdb.org/2021/12/03/duck-arrow.html) also integrates with Arrow, allowing you to read larger-than-memory datasets in batches and use a new and modern way to connect to databases, the Arrow Database Connectivity protocol ([ADBC](https://duckdb.org/2023/08/04/adbc.html)).

Being that good at interoperability, it is not surprising that Arrow moved into the client-server data transfer with [Arrow IPC streams](https://arrow.apache.org/docs/python/ipc.html). Arrow IPC is a protocol for serializing Arrow record batches into a binary format that can be sent over the wire. Any client that can read an Arrow IPC stream can handle the deserialization of the payload. Another great thing about Arrow is that it can send data in streaming format, breaking the payload up into different `RecordBatch` sizes. Because Arrow record batches have a columnar disposition, any data processing engine on the client side that integrates with Arrow will have a much easier time processing the received data. It is also worth noting that the IPC format also attaches a schema in the payload with Arrow types, solving the common issue of interpreting the data wrongly on the client side!

## Comparing JSON with Arrow (pears vs apples)
Comparing JSON with Arrow is a bit unfair, since they are clearly designed with different purposes in mind. However, this comparison table may help you decide which format you may want to go for in your future projects:

| Feature                          | JSON                          | Arrow IPC                     |
|----------------------------------|-------------------------------|--------------------------------|
| **Readability**                  | Human-readable text format    | Binary format, not human-readable |
| **Serialization/Deserialization**| Slower due to text processing | Faster due to binary format   |
| **Compactness**                  | Less compact, larger payloads | Highly compact, smaller payloads |
| **Cross-language support**       | Excellent, widely supported   | Excellent, widely supported   |
| **Handling large datasets**      | Inefficient for large data    | Optimized for large datasets  |
| **Streaming support**            | Possible with `jsonl` but not efficient for large data                       | Supports streaming with `RecordBatch` |
| **Schema inclusion**             | Not included, prone to errors | Includes schema, ensures data consistency |
| **Data types**        | Limited (no distinction between int/float) | Rich types: int, float, timestamps, nested, etc.  |
| **Debugging**                    | Easy due to text format       | Harder due to binary format   |
| **Random access**     | Not supported (need to read the entire file)                              | Supported (in Arrow file format)                  |
| **Use case**                     | General-purpose, small to medium data | Data-intensive applications, large datasets |

## A small benchmark (just to prove my point)
If you have very little time on your hands &mdash;or just a very small attention window&mdash; this is the section where I show you the numbers. I ran a very simple benchmark where I would call an endpoint that would query the TPCH `lineitem` table for a limit number of rows, then serialize the result into Arrow or JSON, send them over the network, and wait till they are deserialized on the other side. I ran this benchmark for different row limits. For those interested, the code looks like this:
```python
def benchmark():
    benchmark_data = []
    for nrows in [1_000, 10_000, 100_000, 1_000_000, 10_000_000]:
        t1 = time.time()
        read_arrow_stream_from_url_batches(
            f"http://localhost:8000/rows/arrow/lineitem?nrows={nrows}"
        )
        t2 = time.time()
        time_arrow = t2 - t1
        print(f"Time taken to read {nrows} from Arrow stream: {t2 - t1:.2f} seconds")

        t1 = time.time()
        json_request_handler(f"http://localhost:8000/rows/json/lineitem?nrows={nrows}")
        t2 = time.time()
        time_json = t2 - t1
        print(f"Time taken to read {nrows} with JSON: {t2 - t1:.2f} seconds")

        benchmark_data.append(
            {"nrows": nrows, "time_arrow": time_arrow, "time_json": time_json}
        )
    return benchmark_data
```

After running the benchmark, I used the collected data to make a simple plot:
![benchmark](../content/arrow_data_transfer/benchmark.png)

In the image, you can clearly see that after 1M rows &mdash;where Arrow already performs 10x better than JSON&mdash; the thing goes bananas. At 10M rows, JSON goes up to more than 100 seconds, while Arrow is still under 10 seconds. I stopped the benchmark here, but just out of curiosity, I decided to put up the limit for the Arrow request to 100M rows, which it still managed in 30 seconds. Very impressive!

For the people out there that can't stand not knowing the little details, you may be wondering where the time is actually lost. Well, first let's look at a couple of diagrams that break down the different possible points where things go so wrong for JSON and why Arrow may be beating the big J up so badly:

![ser_deser_process](../content/arrow_data_transfer/arrow_json_process.png)

Looking at this image, there are some points where the application could be profiled:
- In the serialization, I am already doing something a bit funky: presenting the DuckDB query results as a Numpy object, which I then serialize using a custom encoder. The Arrow DuckDB interface, on the other hand, is zero-copy and works like a blast. This is unfair to JSON since it is just not a good format for the task.
- On the transfer side, I am also being a bit unfair, since with Arrow I am yielding record batches over the wire, which allows the client to use a streaming request. With JSON, on the other hand, the client needs to wait for all of the data to arrive in order to deserialize. Using `jsonl` could change this, although the JSON serialization would then need to be row-oriented rather than column-oriented. 
- Deserialization, on the other hand, could be easy to measure and a leveled playing field. Even though with Arrow I am using a streaming request, I am waiting for all the chunks to arrive before deserializing the results.

The first point we know is a win for Arrow because of the great integration with DuckDB. The second one is an unfair playing field, but we can still use something like cURL or the Chrome DevTools to profile some metrics. Regarding the third one, we can gather some metrics and see what we can learn from deserializing large amounts of data in these two different formats.

### Finding the bottleneck: using Chrome DevTools
Well, Chrome DevTools are definitely NOT made for data-intensive applications. However, they do have a beautiful Timing tab under the Network tab that gives you some very insightful information about your requests. The following screenshots come from one request to the JSON endpoint and another one to the Arrow endpoint for 1M rows:

![JSON request chrome tools](../content/arrow_data_transfer/chrome_tools_json_1M.png)
![Arrow request chrome tools](../content/arrow_data_transfer/chrome_tools_arrow_1M.png)

We are only going to focus on the `Waiting for server response` and `Content Download` metrics since the others are negligible and also don't have anything to do with serialization performance or time of transfer. In the top screenshot, we have the JSON request. It seems that indeed the serialization process takes around 70% of the total request time and ~30% would be the actual time over the wire. In Arrow's case (bottom screenshot), almost all of the time is spent in serialization and writing the data to the sink (server-side response), while the time over the wire (content download) is incredibly small.

**Note** that because in Arrow we are "streaming" the data, the response of the server is also quicker since it only needs the first batch to be ready in order to send it over to the client.

### Finding the bottleneck: deserialization
Now this was interesting. While clearly deserializing is not the only bottleneck that makes JSON way more inefficient for this type of application, it is clearly a factor. I added some logs in my client at the deserialization stage to time these steps. I was shocked to see that **Arrow has negligible deserialization overhead** to the millisecond level, running the benchmark up to 10M rows. JSON, on the other hand... well, you can see in this graph:
![Arrow vs JSON deserialization benchmark](../content/arrow_data_transfer/arrow_json_deser_benchmark.png)

This graph is pretty similar to the one we saw above. The bigger the payload, the bigger the struggle for JSON. The main :exploding_head: difference is that Arrow doesn't show any overhead while deserializing data. Why? Not sure, but maybe Arrow just maps the binary data layout directly into Arrow structures... and pointing to data is way faster than copying.

## Why is Arrow not all over HTTP services then?
Well, without repeating myself too much, JSON is a readable format. If there is some miscommunication between client and service, it will be way easier to debug a JSON payload than a binary one like Arrow IPC. Also, JSON is the incumbent. It has been there for a long time, and people are familiar with it. And, of course, they may target different use cases in the first place. Arrow is columnar-oriented and targeted for data-intensive applications; JSON is object/row-oriented and a standard for API development in the web application context.

There is, however, another concern that slowly rose up while delving deep into this project: **there is just not that much documentation on how to create an HTTP service using Arrow**. There is quite a bit on [Arrow Flight](https://arrow.apache.org/docs/format/Flight.html), a gRPC beast made for high-volume data transfer. And while very interesting, it also forces you into a certain structure that you may not want to comply with.

The only thing you are left with then is this official [arrow-experiments](https://github.com/apache/arrow-experiments/tree/main/http) repository. This definitely helped a lot, but it is far from a good guide for new developers trying to build an HTTP service using Arrow IPC (at least for Python devs). Moreover, Arrow just feels a bit more difficult. You can see it in the backend code for the service that I built ([here](https://github.com/guillesd/duckdb-arrow-service/blob/main/python/backend.py)). The JSON serialization is much more straightforward. Maybe this is also because Arrow requires you to understand some things about memory buffers and byte sinks, which is inherently more complex in a high-level language than it is to deal with a string or a dictionary.

## Conclusions
Conclusions are subjective. Here is a heavy dose of subjectivity:
- I love the Arrow + DuckDB pairing. It is fast, efficient (no copies), and just works wonderfully. It is also extremely easy to plug in a DuckDB instance on the client side to query the results of the request since DuckDB can query Arrow tables and also Arrow files over HTTP via the `httpfs` extension.
- I would love for Apache Arrow to add more documentation on how to build HTTP services using Arrow IPC. The more documentation, the bigger the chance that the community adopts Arrow IPC for HTTP services. Arrow Flight is cool, but maybe not everyone wants to use gRPC.
- JSON will still be there. At the end of the day, there is a whole ecosystem around it. But it is a highly inefficient format for big payloads.

If you want to dig deeper or run your own benchmarks, here is the repo with all of the resources I used: [duckdb-arrow-service](https://github.com/guillesd/duckdb-arrow-service).