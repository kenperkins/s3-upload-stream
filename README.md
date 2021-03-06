## s3-upload-stream [![Build Status](https://travis-ci.org/nathanpeck/s3-upload-stream.svg)](https://travis-ci.org/nathanpeck/s3-upload-stream)

A pipeable write stream which uploads to Amazon S3 using the multipart file upload API.

[![NPM](https://nodei.co/npm/s3-upload-stream.png?downloads=true)](https://www.npmjs.org/package/s3-upload-stream)

### Changelog

#### 1.0.0 (2014-09-15)

Major overhaul of the functional interface. Breaks compatability with older versions of the module in favor of a cleaner, more streamlined approach. A migration guide for users of older versions of the module has been included in the documentation.

#### 0.6.2 (2014-08-31)

Upgrading the AWS SDK dependency to the latest version. Fixes issue #11

[Historical Changelogs](CHANGELOG.md)

### Why use this stream?

* This upload stream does not require you to know the length of your content prior to beginning uploading. Many other popular S3 wrappers such as [Knox](https://github.com/LearnBoost/knox) also allow you to upload streams to S3, but they require you to specify the content length. This is not always feasible.
* By piping content to S3 via the multipart file upload API you can keep memory usage low even when operating on a stream that is GB in size. Many other libraries actually store the entire stream in memory and then upload it in one piece. This stream avoids high memory usage by flushing the stream to S3 in 5 MB parts such that it should only ever store 5 MB of the stream data at a time.
* This package is designed to use the official Amazon SDK for Node.js, helping keep it small and efficient. For maximum flexibility you pass in the aws-sdk client yourself, allowing you to use a uniform version of AWS SDK throughout your code base.
* You can provide options for the upload call directly to do things like set server side encryption, reduced redundancy storage, or access level on the object, which some other similar streams are lacking.
* Emits "part" events which expose the amount of incoming data received by the writable stream versus the amount of data that has been uploaded via the multipart API so far, allowing you to create a progress bar if that is a requirement.

### Limits

* The multipart upload API does not accept parts less than 5 MB in size. So although this stream emits "part" events which can be used to show progress, the progress is not very granular, as the events are only per part. By default this means that you will receive an event each 5 MB.
* The Amazon SDK has a limit of 10,000 parts when doing a mulitpart upload. Since the part size is currently set to 5 MB this means that your stream will fail to upload if it contains more than 50 GB of data. This can be solved by using the 'stream.maxPartSize()' method of the writable stream to set the max size of an upload part, as documented below. By increasing this value you should be able to save streams that are many TB in size.

## Example

```js
var s3Stream = require('s3-upload-stream'),
    AWS      = require('aws-sdk'),
    zlib     = require('zlib'),
    fs       = require('fs');

// Set the client to be used for the upload.
AWS.config.loadFromPath('./config.json');
s3Stream.client(new AWS.S3());

// Create the streams
var read = fs.createReadStream('/path/to/a/file');
var compress = zlib.createGzip();
var upload = s3Stream.upload({
  "Bucket": "bucket-name",
  "Key": "key-name"
});

// Optional configuration
upload.maxPartSize(20971520); // 20 MB
upload.concurrentParts(5);

// Handle errors.
upload.on('error', function (error) {
  console.log(error);
});

/* Handle progress. Example details object:
   { ETag: '"f9ef956c83756a80ad62f54ae5e7d34b"',
     PartNumber: 5,
     receivedSize: 29671068,
     uploadedSize: 29671068 }
*/
upload.on('part', function (details) {
  console.log(details);
});

/* Handle upload completion. Example details object:
   { Location: 'https://bucketName.s3.amazonaws.com/filename.ext',
     Bucket: 'bucketName',
     Key: 'filename.ext',
     ETag: '"bf2acbedf84207d696c8da7dbb205b9f-5"' }
*/
upload.on('uploaded', function (details) {
  console.log(details);
});

// Pipe the incoming filestream through compression, and up to S3.
read.pipe(compress).pipe(upload);
```

## Usage

### package.client(s3);

Configures the S3 client for s3-upload-stream to use. Please note that this module has only been tested with AWS SDK 2.0 and greater.

This module does not include the AWS SDK itself. Rather you must require the AWS SDK in your own application code, instantiate an S3 client and then supply it to s3-upload-stream.

The main advantage of this is that rather than being stuck with a set version of the AWS SDK that ships with s3-upload-stream you can ensure that s3-upload-stream is using whichever verison of the SDK you want.

When setting up the S3 client the recommended approach for credential management is to [set your AWS API keys using environment variables](http://docs.aws.amazon.com/AWSJavaScriptSDK/guide/node-configuring.html) or [AMI roles](http://docs.aws.amazon.com/IAM/latest/UserGuide/WorkingWithRoles.html).

If you are following this approach then you can configure the S3 client very simply:

```js
var s3Stream = require('s3-upload-stream'),
    AWS      = require('aws-sdk');

s3Stream.client(new AWS.S3());
```

However, some environments may require you to keep your credentials in a file, or hardcoded. In that case you can use the following form:

```js
var s3Stream = require('s3-upload-stream'),
    AWS      = require('aws-sdk');

AWS.config.update({accessKeyId: 'akid', secretAccessKey: 'secret'});
s3Stream.client(new AWS.S3());
```

### package.upload(destination)

Create an upload stream that will upload to the specified destination. The upload stream is returned immeadiately.

The destination details is an object in which you can specify many different [destination properties enumerated in the AWS S3 documentation](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#createMultipartUpload-property).

__Example:__

```js
var s3Stream = require('s3-upload-stream'),
    AWS      = require('aws-sdk');

s3Stream.client(new AWS.S3());

var read = fs.createReadStream('/path/to/a/file');
var upload = s3Client.upload({
  "Bucket": "bucket-name",
  "Key": "key-name",
  "ACL": "public-read",
  "StorageClass": "REDUCED_REDUNDANCY",
  "ContentType": "binary/octet-stream"
});

read.pipe(upload);
```

## Optional Configuration

### stream.maxPartSize(sizeInBytes)

Used to adjust the maximum amount of stream data that will be buffered in memory prior to flushing. The lowest possible value, and default value, is 5 MB. It is not possible to set this value any lower than 5 MB due to Amazon S3 restrictions, but there is no hard upper limit. The higher the value you choose the more stream data will be buffered in memory before flushing to S3.

The main reason for setting this to a higher value instead of using the default is if you have a stream with more than 50 GB of data, and therefore need larger part sizes in order to flush the entire stream while also staying within Amazon's upper limit of 10,000 parts for the multipart upload API.

```js
var s3Stream = require('s3-upload-stream'),
    AWS      = require('aws-sdk');

s3Stream.client(new AWS.S3());

var read = fs.createReadStream('/path/to/a/file');
var upload = s3Client.upload({
  "Bucket": "bucket-name",
  "Key": "key-name"
});

upload.maxPartSize(20971520); // 20 MB

read.pipe(upload);
```

### stream.concurrentParts(numberOfParts)

Used to adjust the number of parts that are concurrently uploaded to S3. By default this is just one at a time, to keep memory usage low and allow the upstream to deal with backpressure. However, in some cases you may wish to drain the stream that you are piping in quickly, and then issue concurrent upload requests to upload multiple parts.

Keep in mind that total memory usage will be at least `maxPartSize` * `concurrentParts` as each concurrent part will be `maxPartSize` large, so it is not recommended that you set both `maxPartSize` and `concurrentParts` to high values, or your process will be buffering large amounts of data in its memory.

```js
var s3Stream = require('s3-upload-stream'),
    AWS      = require('aws-sdk');

s3Stream.client(new AWS.S3());

var read = fs.createReadStream('/path/to/a/file');
var upload = s3Client.upload({
  "Bucket": "bucket-name",
  "Key": "key-name"
});

upload.concurrentParts(5);

read.pipe(upload);
```

### Migrating from pre-1.0 s3-upload-stream

The methods and interface for s3-upload-stream has changed since 1.0 and is no longer compatible with the older versions.

The differences are:

* This package no longer includes Amazon SDK, and now you must include it in your own app code and pass an instantiated Amazon S3 client in.
* The upload stream is now returned immeadiately, instead of in a callback.
* The "chunk" event emitted is now called "part" instead.
* The .maxPartSize() and .concurrentParts() methods are now methods of the writable stream itself, instead of being methods of an object returned from the upload stream constructor method.

If you have questions about how to migrate from the older version of the package after reviewing these docs feel free to open an issue with your code example.

### Tuning configuration of the AWS SDK

The following configuration tuning can help prevent errors when using less reliable internet connections (such as 3G data if you are using Node.js on the Tessel) by causing the AWS SDK to detect upload timeouts and retry.

```js
var AWS = require('aws-sdk');
AWS.config.httpOptions = {timeout: 5000};
```

### Installation

```
npm install s3-upload-stream
```

### Running Tests

```
npm test
```

### License

(The MIT License)

Copyright (c) 2014 Nathan Peck <nathan@storydesk.com>

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
'Software'), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
