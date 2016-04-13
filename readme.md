# mississippi2

[![NPM](https://nodei.co/npm/mississippi2.png?downloads=true)](https://nodei.co/npm/mississippi2/)

This module was inspired from [mississippi](https://github.com/maxogden/mississippi).

## usage

```bash
npm install mississippi2 --save
```

```js
var miss = require('mississippi2')
```

## methods

- [pipe](#pipe)
- [merge](#merge) *
- [condition](#condition) *
- [each](#each)
- [map](#map) *
- [filter](#filter) *
- [reduce](#reduce) *
- [split](#split) *
- [spy](#spy) *
- [pipeline](#pipeline)
- [duplex](#duplex)
- [through](#through)
- [from](#from)
- [fromValue](#fromvalue) *
- [fromPromise](#frompromise) *
- [fromObservable](#fromobservable) *
- [to](#to)
- [toString](#tostring) *
- [toArray](#toarray) *
- [toPromise](#topromise) *
- [toObservable](#toobservable) *
- [concat](#concat)
- [unique](#unique) *
- [toJSON](#tojson) *
- [stringify](#stringify) *
- [child](#child) *
- [finished](#finished)
- [throttle](#throttle) *
- [Parser](#parser) *

### pipe

##### `miss.pipe(stream1, stream2, stream3, ..., cb)`

Pipes streams together and destroys all of them if one of them closes. Calls `cb` with `(error)` if there was an error in any of the streams.

When using standard `source.pipe(destination)` the source will _not_ be destroyed if the destination emits close or error. You are also not able to provide a callback to tell when the pipe has finished.

`miss.pipe` does these two things for you, ensuring you handle stream errors 100% of the time (unhandled errors are probably the most common bug in most node streams code)

#### original module

`miss.pipe` is provided by [`require('pump')`](https://npmjs.org/pump)

#### example

```js
// lets do a simple file copy
var fs = require('fs')

var read = fs.createReadStream('./original.zip')
var write = fs.createWriteStream('./copy.zip')

// use miss.pipe instead of read.pipe(write)
miss.pipe(read, write, function (err) {
  if (err) return console.error('Copy error!', err)
  console.log('Copied successfully')
})
```

### merge

##### `miss.merge(streams, [options])`

Return a streams that merged alll streams together and emmit parallely events. When merge readable streams, return a readable stream that reads from multiple readable streams at the same time. If you want to emits multiple other streams one after another, use [merge2](https://github.com/teambition/merge2). When merge writable streams, retrun a writable stream that writes to multiple other writeable streams.

#### original module

`miss.merge` is provided by [`require('multi-duplex-stream')`](https://github.com/emilbayes/multi-duplex-stream)

#### example

```js
// merge readable stream
var multiRead = miss.merge([
  miss.fromValue("hello"),
  miss.fromValue("world")
]);

multiRead.on("data", function(data){
  console.log(data); // "hello" "world" or "world" "hello"
});

multiRead.on("end", function(){
  console.log("no more data");
});

// merge writable stream
var read = miss.fromValue('hello, world');
var write1 = fs.createWriteStream('./file1.txt');
var write2 = fs.createWriteStream('./file2.txt');

var multiWrite = miss.merge([write1,write2]);

read.pipe(multiWrite).on("end",function(){
  // both file1 and file2 now contains "hello, world"
});
```

### condition

##### `miss.condition(condition, stream, [elseStream])`

Condition stream can conditionally control the flow of stream data.

Condition stream will pipe data to `stream` whenever `condition` is truthy.

If `condition` is falsey and `elseStream` is passed, data will pipe to `elseStream`.

After data is piped to `stream` or `elseStream` or neither, data is piped down-stream.

#### original module

`miss.condition` is provided by [`require('ternary-stream')`](https://github.com/robrich/ternary-stream)

#### example

```js
// if the condition returns truthy, data is piped to the child stream
var condition = function (data) {
  return true;
};

process.stdin
  .pipe(miss.condition(condition, process.stdout))
  .pipe(fs.createWriteStream('./out.txt'));

// Data will conditionally go to stdout, and always go to the file
var through2 = require('through2');

var count = 0;
var condition = function (data) {
  count++;
  return count % 2;
};

process.stdin
  .pipe(miss.condition(condition, fs.createWriteStream('./truthy.txt'), fs.createWriteStream('./falsey.txt')))
  .pipe(process.stdout);
```

### each

##### `miss.each(stream, each, [done])`

Iterate the data in `stream` one chunk at a time. Your `each` function will be called with `(data, next)` where data is a data chunk and next is a callback. Call `next` when you are ready to consume the next chunk.

Optionally you can call `next` with an error to destroy the stream. You can also pass the optional third argument, `done`, which is a function that will be called with `(err)` when the stream ends. The `err` argument will be populated with an error if the stream emitted an error.

#### original module

`miss.each` is provided by [`require('stream-each')`](https://npmjs.org/stream-each)

#### example

```js
var fs = require('fs')
var split = require('split2')

var newLineSeparatedNumbers = fs.createReadStream('numbers.txt')

var pipeline = miss.pipeline(newLineSeparatedNumbers, split())
var each = miss.each(pipeline, eachLine, done)
var sum = 0

function eachLine (line, next) {
  sum += parseInt(line.toString())
  next()
}

function done (err) {
  if (err) throw err
  console.log('sum is', sum)
}
```

### map

##### `miss.map([options,] fn)`

Return a `stream.Transfrom` instance that will call `fn(chunk, index)` on each stream segment.

Note you will __NOT__ be able to skip chunks. This is intended for modification only. If you want filter the stream content, use `miss.filter`. This transform also does not have a flush function.

#### original module

`miss.map` is provided by [`require('through2-map')`](https://github.com/brycebaril/through2-map)

#### example

```js
var truncate = miss.map(function (chunk) {
  return chunk.slice(0, 10)
});

// Then use your map:
source.pipe(truncate).pipe(sink)

```

### filter

##### `miss.filter([options], fn)`

Create a  `through2-filter`  instance that will call `fn(chunk)`. If `fn(chunk)` returns "true" the chunk will be passed downstream. Otherwise it will be dropped.

Note you will __NOT__ be able to alter the content of the chunks. This is intended for filtering only. If you want to modify the stream content, use either  `miss.through`  or `miss.map`.

#### original module

`miss.filter` is provided by [`require('through2-filter')`](https://github.com/brycebaril/through2-filter)

#### example

```js
var skip = miss.filter(function (chunk) {
  // skip buffers longer than 100
  return chunk.length < 100
})

// Then use your filter:
source.pipe(skip).pipe(sink)
```

### reduce

##### `miss.reduce([options,] fn [,initial])`

Create a Reduce *instance*. Works like `Array.prototype.reduce` meaning you can specify a `fn` function that takes up to *three* arguments: fn(previous, current, index) and you can specify an `initial` value.

This stream will only ever emit a *single* chunk. For more traditional `stream.Transform` filters or transforms, consider `miss.through` `miss.filter` or `miss.map`.

#### original module

`miss.reduce` is provided by [`require('through2-reduce')`](https://github.com/brycebaril/through2-reduce)

#### example

```js
var sum = miss.reduce({objectMode: true}, function (previous, current) { return previous + current })

// Then use your reduce: (e.g. source is an objectMode stream of numbers)
source.pipe(sum).pipe(sink)
```

### split

##### `miss.split([matcher, mapper, options])`

Break up a stream and reassemble it so that each line is a chunk.

`matcher` may be a `String`, or a `RegExp`.

#### original module

`miss.split` is provided by [`require('split2')`](https://github.com/mcollina/split2)

#### example

```js
 fs.createReadStream(file)
    .pipe(miss.split())
    .on('data', function (line) {
      //each chunk now is a seperate line!
    })
```

### spy 

##### `miss.spy([options], fn)`

Create a `miss.spy` instance that will call `fn(chunk)` and then silently pass through data downstream.

Note you will **NOT** be able to do anything but spy and abort the stream pipeline. To do any filtering or transformations you should consider `miss.through` `miss.filter` or `miss.map`.

#### original module

`miss.spy` is provided by [`require('through2-spy')`](https://github.com/brycebaril/through2-spy)

#### example

```js
var count = 0;
var countChunks = miss.spy(function (chunk) {
  count++
});

// Then use your spy:
source.pipe(countChunks).pipe(sink);
```

### pipeline

##### `var pipeline = miss.pipeline(stream1, stream2, stream3, ...)`

Builds a pipeline from all the transform streams passed in as arguments by piping them together and returning a single stream object that lets you write to the first stream and read from the last stream.

If any of the streams in the pipeline emits an error or gets destroyed, or you destroy the stream it returns, all of the streams will be destroyed and cleaned up for you.

#### original module

`miss.pipeline` is provided by [`require('pumpify')`](https://npmjs.org/pumpify)

#### example

```js
// first create some transform streams (note: these two modules are fictional)
var imageResize = require('image-resizer-stream')({width: 400})
var pngOptimizer = require('png-optimizer-stream')({quality: 60})

// instead of doing a.pipe(b), use pipelin
var resizeAndOptimize = miss.pipeline(imageResize, pngOptimizer)
// `resizeAndOptimize` is a transform stream. when you write to it, it writes
// to `imageResize`. when you read from it, it reads from `pngOptimizer`.
// it handles piping all the streams together for you

// use it like any other transform stream
var fs = require('fs')

var read = fs.createReadStream('./image.png')
var write = fs.createWriteStream('./resized-and-optimized.png')

miss.pipe(read, resizeAndOptimize, write, function (err) {
  if (err) return console.error('Image processing error!', err)
  console.log('Image processed successfully')
})
```

### duplex

##### `var duplex = miss.duplex([writable, readable, opts])`

Take two separate streams, a writable and a readable, and turn them into a single [duplex (readable and writable) stream](https://nodejs.org/api/stream.html#stream_class_stream_duplex).

The returned stream will emit data from the readable. When you write to it it writes to the writable.

You can either choose to supply the writable and the readable at the time you create the stream, or you can do it later using the `.setWritable` and `.setReadable` methods and data written to the stream in the meantime will be buffered for you.

#### original module

`miss.duplex` is provided by [`require('duplexify')`](https://npmjs.org/duplexify)

#### example

```js
// lets spawn a process and take its stdout and stdin and combine them into 1 stream
var child = require('child_process')

// @- tells it to read from stdin, --data-binary sets 'raw' binary mode
var curl = child.spawn('curl -X POST --data-binary @- http://foo.com')

// duplexCurl will write to stdin and read from stdout
var duplexCurl = miss.duplex(curl.stdin, curl.stdout)
```

### through

#####`var transformer = miss.through([options, transformFunction, flushFunction])`

Make a custom [transform stream](https://nodejs.org/docs/latest/api/stream.html#stream_class_stream_transform).

The `options` object is passed to the internal transform stream and can be used to create an `objectMode` stream (or use the shortcut `miss.through.obj([...])`)

The `transformFunction` is called when data is available for the writable side and has the signature `(chunk, encoding, cb)`. Within the function, add data to the readable side any number of times with `this.push(data)`. Call `cb()` to indicate processing of the `chunk` is complete. Or to easily emit a single error or chunk, call `cb(err, chunk)`

The `flushFunction`, with signature `(cb)`, is called just before the stream is complete and should be used to wrap up stream processing.

#### original module

`miss.through` is provided by [`require('through2')`](https://npmjs.org/through2)

#### example

```js
var fs = require('fs')

var read = fs.createReadStream('./boring_lowercase.txt')
var write = fs.createWriteStream('./AWESOMECASE.TXT')

// Leaving out the options object
var uppercaser = miss.through(
  function (chunk, enc, cb) {
    cb(null, chunk.toString().toUpperCase())
  },
  function (cb) {
    cb(null, 'ONE LAST BIT OF UPPERCASE')
  }
)

miss.pipe(read, uppercaser, write, function (err) {
  if (err) return console.error('Trouble uppercasing!')
  console.log('Splendid uppercasing!')
})
```

### from

#####`miss.from([opts], read)`

Make a custom [readable stream](https://nodejs.org/docs/latest/api/stream.html#stream_class_stream_readable).

`opts` contains the options to pass on to the ReadableStream constructor e.g. for creating a readable object stream (or use the shortcut `miss.from.obj([...])`).

Returns a readable stream that calls `read(size, next)` when data is requested from the stream.

- `size` is the recommended amount of data (in bytes) to retrieve.
- `next(err, chunk)` should be called when you're ready to emit more data.

#### original module

`miss.from` is provided by [`require('from2')`](https://npmjs.org/from2)

#### example

```js
function fromString(string) {
  return miss.from(function(size, next) {
    // if there's no more content
    // left in the string, close the stream.
    if (string.length <= 0) return next(null, null)

    // Pull in a new chunk of text,
    // removing it from the string.
    var chunk = string.slice(0, size)
    string = string.slice(size)

    // Emit "chunk" from the stream.
    next(null, chunk)
  })
}

// pipe "hello world" out
// to stdout.
fromString('hello world').pipe(process.stdout)
```

### fromValue

#####`miss.fromValue(value)`

Create streams from (single) arbitrary Javascript values like `strings`, `functions`, `arrays`, etc. 

Please __Note__ miss.fromValue are [Readable](http://nodejs.org/api/stream.html#stream_class_stream_readable_1) streams.

#### original module

`miss.fromValue` is provided by [`require('stream-from-value')`](https://github.com/schnittstabil/stream-from-value)

#### example

```js
miss.fromValue('some string')
  .pipe(process.stdout); // output: some string

miss.fromValue(new Buffer('some string'))
  .pipe(process.stdout); // output: some string

// Stream of (arbitrary) Javascript Value
miss.fromValue.obj(['some', 'mixed', 'array', 42]).on('data', function(data){
    console.log(data); // output: [ 'some', 'mixed', 'array', 42 ]
  });
```

### fromPromise

#####`miss.fromPromise(promise)`

Make a [Readable](http://nodejs.org/api/stream.html#stream_class_stream_readable_1) streams from `ECMAScript 2015 Promises` that return Javascript values like numbers, strings, objects, functions.

#### original module

`miss.fromPromise` is provided by [`require('stream-from-promise')`](https://github.com/schnittstabil/stream-from-promise)

#### example

```js
// `String promises
var stringPromise = new Promise(function(resolve, reject){
  setTimeout(function(){
    resolve('strrrring!');
  }, 500);
});

miss.fromPromise(stringPromise).pipe(process.stdout); // => output: strrrring!

// Buffer promises
var bufferPromise = new Promise(function(resolve, reject){
  setTimeout(function(){
    resolve(new Buffer('buff!'));
  }, 500);
});

StreamFromPromise(bufferPromise)
  .pipe(process.stdout); // output: buff!

```

### fromObservable

#####`miss.fromObservable(observable, stream, [encoding])`

Writes an observable sequence to a writable stream.

__Arguments__

- `observable` *(Observable)*: Observable sequence to write to a stream.
- `stream` *(Stream)*: The stream to write to.
- `[encoding]` *(String)*: The encoding of the item to write.

__Returns__
- *(Disposable)*: The subscription handle.

#### original module

`miss.fromObsesrvable` is provided by [`require("rx-node").writeToStream`](https://github.com/Reactive-Extensions/rx-node)

#### example

```js
var Rx = require('rx');
var source = Rx.Observable.range(0, 5);

var subscription = miss.fromObsesrvable(source, process.stdout, 'utf8');
// => 01234
```

### to

#####`miss.to([options], write, [flush])`

Make a custom [writable stream](https://nodejs.org/docs/latest/api/stream.html#stream_class_stream_writable).

`opts` contains the options to pass on to the WritableStream constructor e.g. for creating a readable object stream (or use the shortcut `miss.to.obj([...])`).

Returns a writable stream that calls `write(data, enc, cb)` when data is written to the stream.

- `data` is the received data to write the destination.
- `enc` encoding of the piece of data received.
- `next(err, chunk)` should be called when you're ready to write more data, or encountered an error.

`flush(cb)` is called before `finish` is emitted and allows for cleanup steps to occur.

#### original module

`miss.to` is provided by [`require('flush-write-stream')`](https://npmjs.org/flush-write-stream)

#### example

```js
var ws = miss.to(write, flush)

ws.on('finish', function () {
  console.log('finished')
})

ws.write('hello')
ws.write('world')
ws.end()

function write (data, enc, cb) {
  // i am your normal ._write method
  console.log('writing', data.toString())
  cb()
}

function flush (cb) {
  // i am called before finish is emitted
  setTimeout(cb, 1000) // wait 1 sec
}
```

If you run the above it will produce the following output

```
writing hello
writing world
(nothing happens for 1 sec)
finished
```

### toString

#####`miss.toString(stream [, callback])`

Pipe a stream into a string, collect value with callback or promise.

Collects stream data into a string. Executes optional `callback(err, string)`. Returns a promise.

#### original module

`miss.toString` is provided by [`require('stream-to-string')`](https://github.com/jasonpincin/stream-to-string)

#### example

```js
// with callback
var stream = miss.through();

miss.toString(stream, function (err, msg) {
    console.log(msg)
})

// or with promises
miss.toString(stream).then(function (msg) {
    console.log(msg)
})

stream.write('this is a')
stream.write(' test')
stream.end()
```

### toArray

#####`miss.toArray([stream], [callback(err, arr)])`

Concatenate a readable stream's data into a single array.

Returns all the data objects in an array. This is useful for streams in object mode if you want to just use an array.

#### original module

`miss.toArray` is provided by [`require('stream-to-array')`](https://github.com/stream-utils/stream-to-array)

#### example

```js
// with callback
var stream = miss.through();

miss.toArray(stream, function (err, msg) {
    console.log(msg)
})

// or with promises
miss.toArray(stream).then(function (msg) {
    console.log(msg)
})

stream.write('this is a')
stream.write(' test')
stream.end()
```

### toPromise

#####`miss.toPromise(stream)`

Convert streams (readable or writable) to promises.

#### original module

`miss.toPromise` is provided by [`require('stream-to-promise2')`](https://github.com/htoooth/stream-to-promise2)

#### example

```js
miss.toPromise(readableStream).then(function (buffer) {
  // buffer.length === 3
});
readableStream.emit('data', new Buffer());
readableStream.emit('data', new Buffer());
readableStream.emit('data', new Buffer());
readableStream.emit('end'); // promise is resolved here
```

### toObservable

#####`miss.toObservable(stream, finishEventName, dataEventName)`

Converts a flowing readable to an Observable sequence.

__Arguments__

- `stream` *(Stream)*: A stream to convert to a observable sequence.
- `[dataEventName]` *(String)*: Event that notifies about incoming data. ("data" by default)

__Returns__

- *(Observable)*: An observable sequence which fires on each 'data' event as well as handling 'error' and 'end' events.

#### original module

`miss.toObservable` is provided by [`require("rx-node").writeToStream`](https://github.com/Reactive-Extensions/rx-node)

#### example

```js
var subscription = miss.toObservable(process.stdin)
    .subscribe(function (x) { console.log(x); });

// => r<Buffer 72>
// => x<Buffer 78>
```

### concat

#####`var concat = miss.concat(cb)`

Returns a writable stream that concatenates all data written to the stream and calls a callback with the single result.

Calling `miss.concat(cb)` returns a writable stream. `cb` is called when the writable stream is finished, e.g. when all data is done being written to it. `cb` is called with a single argument, `(data)`, which will containe the result of concatenating all the data written to the stream.

Note that `miss.concat` will not handle stream errors for you. To handle errors, use `miss.pipe` or handle the `error` event manually.

#### original module

`miss.concat` is provided by [`require('concat-stream')`](https://npmjs.org/concat-stream)

#### example

```js
var fs = require('fs')
var concat = require('concat-stream')

var readStream = fs.createReadStream('cat.png')
var concatStream = concat(gotPicture)

readStream.on('error', handleError)
readStream.pipe(concatStream)

function gotPicture(imageBuffer) {
  // imageBuffer is all of `cat.png` as a node.js Buffer
}

function handleError(err) {
  // handle your error appropriately here, e.g.:
  console.error(err) // print the error to STDERR
  process.exit(1) // exit program with non-zero exit code
}
```

### unique

#####`miss.unique([fn])`

Filter duplicates from a stream based on a hashing `fn(chunk)`. By default, this hashing function is:

```js
sha256sum(JSON.stringify(doc))
```

#### original module

`miss.unique` is provided by [`require('unique-hash-stream')`](https://github.com/stream-utils/unique-hash-stream)

#### example

```js
var stream = miss.through();

var hashFn = function(chunk){
  return chunk;
};

stream.pipe(miss.unique(hashFn)).pipe(process.stdout);

stream.write("a");
stream.write("a");
stream.write("b");
stream.end();

```

### toJSON

#####`miss.toJSON(stream, callback)`

Read all from `stream`, then `JSON.parse` and call `callback(err, json)` with the result. If there's an Error in the stream itself, or parsing the JSON, an error will be passed.

#### original module

`miss.toJSON` is provided by [`require('stream-to-json')`](https://www.npmjs.com/package/stream-to-json)

#### example

```js
var request = require('request');
 
miss.toJSON(request('/some/url.json'), function(err, json) {
  if (err) throw err;
  console.log(json);
});
```


### stringify

#####`miss.stringify([options])`

Similar to JSONStream.stringify() except it is, by default, a binary stream, and it is a streams2 implementation.

Please __NOTE__ : The main use case for this is to stream a database query to a web client. This is meant to be used only with `arrays`, not `objects`.

__Separators__

- The stream always starts with '[\n'.
- Documents are separated by '\n,\n'.
- The stream is terminated with '\n]\n'.

#### original module

`miss.stringify` is provided by [`require('streaming-json-stringify')`](https://github.com/stream-utils/streaming-json-stringify)

#### example

```js
app.get('/things', function (req, res, next) {
  res.setHeader('Content-Type', 'application/json; charset=utf-8')

  db.things.find()
  .stream()
  .pipe(miss.stringify())
  .pipe(res)
})
```

will yield something like

```
[
{"_id":"123412341234123412341234"}
,
{"_id":"123412341234123412341234"}
]
```


### child

#####`miss.child(command, [args], [options])`

Spawn a child process as a duplex stream.

Convenience wrapper for:

```js
new Child_Process().spawn(command, [args], [options])
```

#### original module

`miss.child` is provided by [`require('duplex-child-process')`](https://github.com/stream-utils/duplex-child-process)

#### example

```js
var toJPEG = miss.child.spawn('convert', ['-', 'JPEG:-'])
var getFormat = miss.child.spawn('identify', ['-format', '%m', '-'])

fs.createReadStream('img.png')
.pipe(toJPEG)
.pipe(getFormat)
.once('readable', function () {
  var format = this.read().toString('utf8')
  assert.equal(format, 'JPEG')
})
```

### finished

#####`miss.finished(stream, cb)`

Waits for `stream` to finish or error and then calls `cb` with `(err)`. `cb` will only be called once. `err` will be null if the stream finished without error, or else it will be populated with the error from the streams `error` event.

This function is useful for simplifying stream handling code as it lets you handle success or error conditions in a single code path. It's used internally `miss.pipe`.

#### original module

`miss.finished` is provided by [`require('end-of-stream')`](https://npmjs.org/end-of-stream)

#### example

```js
var copySource = fs.createReadStream('./movie.mp4')
var copyDest = fs.createWriteStream('./movie-copy.mp4')

copySource.pipe(copyDest)

miss.finished(copyDest, function(err) {
  if (err) return console.log('write failed', err)
  console.log('write success')
})
```

### throttle

#####`miss.throttle(n)`

This stream offers a `Throttle` passthrough stream class, which allows you to write data to it and it will be passed through in `n` bytes per second. It can be useful for throttling HTTP uploads or to simulate reading from a file in real-time, etc.

#### original module

`miss.throttle` is provided by [`require("throttle")`](https://github.com/TooTallNate/node-throttle)

#### example

```js
// throttling stdin at 1 byte per second and outputting the data to stdout:
process.stdin.pipe(miss.throttle(1)).pipe(process.stdout);
```

### Parser

#####`miss.Parser`

This tool offers the `stream-parser` mixin, which provides an easy-to-use API for parsing bytes from `Writable` and/or `Transform` stream instances. This module is great for implementing streaming parsers for standardized file formats.

For `Writable` streams, the parser takes control over the `_write` callback function. For Transform streams, the parser controls the `_transform` callback function.

##### api

  The `Parser` stream mixin works with either `Writable` or `Transform` stream
  instances/subclasses. Provides a convenient generic "parsing" API:

```js
_bytes(n, cb) - buffers "n" bytes and then calls "cb" with the "chunk"
_skipBytes(n, cb) - skips "n" bytes and then calls "cb" when done
```

  If you extend a `Transform` stream, then the `_passthrough()` function is also
  added:

```js
_passthrough(n, cb) - passes through "n" bytes untouched and then calls "cb"
```

###### ._bytes(n, cb)

  Buffers `n` bytes and then invokes `cb` once that amount has been collected.

###### ._skipBytes(n, cb)

  Skips over the next `n` bytes and then invokes `cb` once that amount has been
  discarded.

###### ._passthrough(n, cb)

  Passes through `n` bytes to the readable side of this stream untouched,
  then invokes `cb` once that amount has been passed through. This function is only defined
  when stream-parser is extending a `Transform` stream.

##### original module

`miss.Parser` is provided by [`require('stream-parser')`](https://github.com/TooTallNate/node-stream-parser)

#### example

Let's create a quick `Transform` stream subclass that utilizes the parser's
`_bytes()` and `_passthrough()` functions to parse a theoretical file format that
has an 8-byte header we want to parse, and then pass through the rest of the data.

```js
var inherits = require('util').inherits;
var Transform = require('stream').Transform;

// create a Transform stream subclass
function MyParser () {
  Transform.call(this);

  // buffer the first 8 bytes written
  this._bytes(8, this.onheader);
}
inherits(MyParser, Transform);

// mixin stream-parser into MyParser's `prototype`
miss.Parser(MyParser.prototype);

// invoked when the first 8 bytes have been received
MyParser.prototype.onheader = function (buffer, output) {
  // parse the "buffer" into a useful "header" object
  var header = {};
  header.type = buffer.readUInt32LE(0);
  header.name = buffer.toString('utf8', 4);
  this.emit('header', header);

  // it's usually a good idea to queue the next "piece" within the callback
  this._passthrough(Infinity);
};


// now we can *use* it!
var parser = new MyParser();
parser.on('header', function (header) {
  console.error('got "header"', header);
});
process.stdin.pipe(parser).pipe(process.stdout);
```