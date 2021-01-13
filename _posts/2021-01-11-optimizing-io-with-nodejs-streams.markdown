---
layout: post
title:  "optimizing I/O with Node.JS streams"
date:   2020-06-23 19:09:00
categories: NodeJS Streams MongoDB
---

We always heard about how Node.JS is very good to build I/O intensive applications. A couple weeks ago, I found myself on a situation where I got the chance to improve an I/O Intensive application performance. But first things first, what does I/O intensive actually means?

<h3>I/O intensive</h3>

I/O stands for input and output. An input/output process could be a database reading/writing, a file reading/writing or receiving/sending network requests. <u>An I/O intensive application pass most it's the time sending or waiting for data</u> and this is what web servers mostly do, example: A web server receives a HTTP request, queries a database, logs info into a file and then responds the request. The pipeline described is a series of I/O processes. Now the question is, why is Node.js suited for I/O intensive applications.

<h3>How does Node.JS handles I/O?</h3>

I once had a boss who wasn't fammiliar with Node.JS, he had been a Java developer for years now. Everytime we talked about Node.js he asked how can Node.JS use a single thread model and still be a good tool for writing web applications. The foundation of my old boss' question is the same as ours question now. How can Node.JS handles I/O with it's single thread model?

<h4>The classic way</h4>

What my old boss was used to were web servers spawning threads in order to process a user request. Althroug it works, it doesn't scale very well for I/O intensive applications. The scalation problem is ultimately related to the nature of I/O described before "An I/O intensive application pass most it's the time sending or waiting for data". It's computational expensive spawn threads that blocks waiting for data.

<h4>What does Node.JS do different?</h4>

<img src="{{'/assets/images/posts/2021-01-11-optimizing-io-with-nodejs-streams/nodejs-event-loop.jpeg' | relative_url}}" alt="nodejs-event-loop.jpeg">

When my old boss talked about Node.JS being single thread he was actually talking about the event loop. Node.js is a runtime environment for Javascript, and the event loop is part of the Javascript model. The event loop is responsible for processing items from the call stack, those items are everything happening synchronously on our code, like for loops. One of the things the Node.js adds to Javascript is the <a href="https://github.com/libuv/libuv">libuv</a>, libuv is a C library for asynchronous I/O. <u>Libuv receives tasks from Node.JS applications and processes them in threads apart from event loop (main thread)</u>. After completition, the results are returned via their callbacks. These callbacks are enqueued in a task queue and processed by the event loop.

You might wonder what exactly is better about this approach, the Node.JS applications don't keep a thread per connection or per request. Threads only process I/O tasks for the application, it doesn't matter if all threads come from one user or from 100 users. If you have to start 3 I/O processes simultaneously, Node.JS parallelize this for you automatically when you keep the code asynchronous.

<h3>A Node.JS I/O intensive application</h3>

A month ago, my colleague showed his side project. It was chat conversations extractor build with Node.JS, this extractor was capable of connecting to our MongoDB and extract all conversations from a period and write the data into a CSV file. This was a quick project with no intention of being performative at all. This project saved me a lot of time when I was asked to extract conversation data for a short period of time.

A couple of weeks ago, an analysis department asked me for the equivalent of two weeks of conversation data. We were in a bit of a hurry so I was trying to deliver their request as quick as possible. Turns out, our application crashed every time I submitted the extraction of 2 weeks period. I had to extract per day and was even crashing for some days. Although I had finished what I was asked for, I thought of this as an opportunity to test how much I could improve this application performance since intentionally none was made before my changes.

<h4>What was the current state</h4>

The application expected 2 dates, and then queried MongoDB docs based on the parameters. The MongoDB collection stored docs that represents a contact and inside the documents there were a documents array with conversations between the contact and our chat bot. Every contact who had a conversation with our bot in the period given, was returned from query.

<h4>What changed (in chronological order)</h4>

The first problem to address was to avoid crashes. The question was, what was crashing our application? It turns out MongoDB was returning more data than our application could handle. This was indeed an expected behavior, Node.JS limits the RAM memory usage. The short and also wrong answer was to increase this limit right away. The first problem we found and is something that will appear again in other forms, memory footprint.

Basically, we were trying to fit all MongoDB data on our limited RAM memory resource. We didn't need all MongoDB data at once because we couldn't even write all this data at once, so why get it all at once? The solution was to change the way we fetch data from MongoDB to a stream based approach.

<h5>Node.JS streams</h5>

<p>According to the Node.js v15.5.1 Documentation,
    <blockquote cite="https://nodejs.org/api/stream.html#stream_stream">
    A stream is an abstract interface for working with streaming data in Node.js.
    </blockquote>
</p>

This application is using the <a href="https://github.com/libuv/libuv">mongoose library<a>. This library exposes a method called "cursor()". Let's see the reference below.
We can imagine streaming data like little pieces of data being consumed by our application in sequential order. 

<p>According to the Mongoose 5.11.11 Documentation,
    <blockquote cite="https://mongoosejs.com/docs/api/querycursor.html#querycursor_QueryCursor">
    A QueryCursor is a concurrency primitive for processing query results one document at a time. A QueryCursor fulfills the Node.js streams3 API, in addition to several other mechanisms for loading documents from MongoDB one at a time.
    </blockquote>
</p>

These "little pieces of data" could be the MongoDB docs we were querying before. In oppose to return all docs at once, <u>now we divide the mongoDB docs returned as chunks</u> and process every one of them individually. The benefits of the streaming data is the ability to process parts of the whole as soon as they arrive and a lower memory footprint. <u>By the time the last piece of information arrives, our application doesn't need to still hold the first part in memory.</u>

After a refactor on the way our aplication was querying data, it never ran out of memory again. It's important to notice that the impacts of the use of mongoDB cursor it's not only on memory but in backpressure handle too. We can define backpressure as difficult to process our data. When we process stream data, our application can adapt to the current backpressure.

<img src="{{'/assets/images/posts/2021-01-11-optimizing-io-with-nodejs-streams/original-vs-custom-cursor.png' | relative_url}}" alt="original-vs-custom-cursor.png">

The image above shows the result of our little implementation comparing response time for extractions of the equivalent of one-day data, just by changing the way we're querying our data, we have now made our application resilient to big results from mongoDB. It's important to note that we have a difference in response times when we are writing results but not when we are only querying them. This difference is the result of another improvement, this time on writing data to file process.

<h5>Writing file with Node.JS WriteStream</h5>

Our original application was writing data to files synchronously. We already know that this process slows down our application a lot, because now we're keeping I/O processes on our event loop. There is more, even if we were already writing lines to our CSV file asynchronously, we could still improve the process with Node.JS <a href="https://nodejs.org/api/fs.html#fs_class_fs_writestream">WriteStream</a>. The Node.JS WriteStream creates a stream that writes data to a file, using this approach we are able to enqueue data to the stream, and this data is then writed to the file asynchronously. That way we again doesn't need to keep holding all file's lines in memory, we could send them to be writed as soon as we generate them.

It's important to remember the backpressure concept when dealing with WriteStream. We can't send data to a WriteStream forever, the WriteStream has an internal buffer. If the application sent the data to the WriteStream via the "write()" method, the data will be buffered, if the buffer size exceeds the highWaterMark option set in the WriteStream constructor, the method will buffer the data but return false as a result. The highWaterMark works like a threshold not a limitation to the buffer size, it's possible to keep buffering data until the application runs out of memory.

<h5>MongoDB query Optimizations</h5>

<img src="{{'/assets/images/posts/2021-01-11-optimizing-io-with-nodejs-streams/mongodb-response-time-comparisson.png' | relative_url}}" alt="mongodb-response-time-comparisson.png">

The first thing modified on the project was the cursor batch size. The cursor batch size determines how many documents will the cursor pull from database at first, this is only limited by the total size of the results (16MB). What really improved overall performance was to filter only important fields on the documents, it improved our process in almost 1 minute. The main reason was that the original version was bringing all contacts in the collection with conversation that happened in a determined time window. The problem was that we we're querying for a lot of data we actually doesn't use.

When dealing with MongoDB, it's important to always create indexes on the fields you are using to query data.

<h4>Binary Classification</h4>

In Binary Classification we have two classes, a positive and a negative class. We can return to our Wi-Fi Fingerprint problem, with the Binary Classification we can say if a fingerprint belongs to a point.

<h4>Multiclass Classification</h4>

<img src="{{'/assets/images/posts/2021-01-11-optimizing-io-with-nodejs-streams/multiclass-classification.png' | relative_url}}" alt="multiclass-classification.png" style="display: block; margin-left: auto; margin-right: auto; width: 70%;">

Binary Classification is excellent to determine if a Wi-Fi Fingerprint belongs to a point (class). But our real problem is that we have lots of different classrooms (also known as points and also known as classes) at the college. There are lots of ways to implement a Multiclass Classification but one famous way is using the "One vs. All" technique. In this method, a set of binary classifiers are created (one for each class) and the input data is executed through all those classifiers, the classification that responds with the best confidence is chosen as the right one.

A Multiclass Classification was exactly what I needed. Given input data (also known as Wi-Fi Fingerprint) my model would respond to me what is the most likely class (or point/classroom at college I was on).

<h3>The hardware prototype</h3>

<h4>Schematics</h4>

<img src="{{'/assets/images/posts/2021-01-11-optimizing-io-with-nodejs-streams/hardware-schematics.png' | relative_url}}" alt="hardware-schematics.png" style="margin-left: auto; margin-right: auto; width: 40%;">

From the schematics drawings above a 3D version was made an then I used a 3D printer.

<h4>The prototype</h4>

<img src="{{'/assets/images/posts/2021-01-11-optimizing-io-with-nodejs-streams/origina-vs-custom-cursor.png' | relative_url}}" alt="origina-vs-custom-cursor.png">

The main idea was built something very easy to use, you just need to plug your headphones to start using it.

<h3>How we define a route between the current position and the destination</h3>

<img src="{{'/assets/images/posts/2021-01-11-optimizing-io-with-nodejs-streams/college-map.png' | relative_url}}" alt="college-map.png">

The image above is a photo of the college map. This photo shows all classrooms (in codes like "A120") and how they are connected. At this point, I had the almost all info I needed to build my map, the questions were, how to structure this data to be used by software?

<h4>Graphs</h4>

<img src="{{'/assets/images/posts/2021-01-11-optimizing-io-with-nodejs-streams/graphs.png' | relative_url}}" alt="graphs.png">

Graphs are a way to structure our data, here the important part is the relationship between our data. The image above is a graph representation of the classrooms I considered for my proof of concept. If we think about going from point A to point B, there is no way to reach the destination without pass by all points in between (unless you teleport yourself). This was kind of what I needed to inform users about their next step towards their destination. The good part is that Graph theory it's something very established, there are a lot of algorithms developed to perform a search inside a graph no matter the size.

<h3>How the navigation works</h3>

The idea was to build a portable device using the Raspberry Pi board. This device could be used by someone who is blind via an audio interface.

The steps are described bellow:

<ul>
  <li>User chooses a classroom as destination</li>
  <li>The device captures the Wi-FI Fingerprint for the current position</li>
  <li>The device calculates a route between the current classroom and the destination classroom</li>
  <li>The device gives an intruction to the user via audio</li>
  <li>The device starts a process of verifying the current Wi-Fi Fingerprint in order to give to the user the next instruction</li>
</ul>

<h3>The project itself</h3>

<img src="{{'/assets/images/posts/2021-01-11-optimizing-io-with-nodejs-streams/wi-fi fingerprints schematic.png' | relative_url}}" alt="wi-fi fingerprints schematic.png">

I had never used Machine Learning for anything before and had a pretty short schedule to finish my project. So I started with Python and it's library Sklearn. At some point, I encountered problems in executing a Sklearn model trained in a powerful machine in a Raspberry. And it's also important to remember that even if I were able to execute it, a Raspberry Pi has very limited resources. So I went for a model where the devices were constantly consulting an external REST API via HTTP. The image above it's a schematic representing the devices and the REST API.

<h3>Final considerations</h3>

This was by far the most ambitious project I ever did by myself. The amount of search I did in areas that I had never seen before was huge. But looking back was worthy because it was the first time I experienced some problems like the need for a data pipeline. I was working with 3 Raspberry boards saving data collected as JSON. I needed to merge all this data and convert it into a tabular format to train my machine learning models. Only the site-survey process took a very long time, but <u>I was able to collect 258.369 Wi-Fi Fingerprints in total</u> (not all were used in the dataset because I limited the classrooms).

The project was an academic success although it had flaws like I wasn't able to separate the college access points apart from other wireless devices from people at college passing by (I realized after that there were ways of doing that). The algorithm responsible for guiding the user could be improved a lot.

The idea was to show some of the work I did, and I hope it could be a starting point for others. You can find my GitHub repository for this project <a href="https://github.com/HenriqueLBorges/WI-FI-Fingerprints-with-Machine-Learning">here</a>, there you can find even Amazon S3 links where I uploaded all my data. You can also find my thesis <a href="{{'/assets/files/pdfs/Produto v7.2 - final.pdf' | relative_url}}">here</a> (it's in Brazilian Portuguese) where I describe in detail what was done in each part and also list every article or book used for research.

Hope you enjoy ;)