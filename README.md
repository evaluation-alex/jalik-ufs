# UploadFS

UploadFS is a package for the Meteor framework that aims to make file uploading easy, fast and configurable.
Some important features are supported like the ability to **start, stop or even abort a transfer**, securing file access, transforming files on writing or reading...

If you want to support this package and feel graceful for all the work, please share this package with the community or feel free to send me pull requests if you want to contribute.

Also I'll be glad to receive donations, whatever you give it will be much appreciated.

[![Donate](https://img.shields.io/badge/Donate-PayPal-green.svg)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=SS78MUMW8AH4N)

## Version 0.6.1

This version brings a huge improvement of transfer speed for large files, the upload channel has been rewritten using standard POST HTTP method instead of Meteor methods. Also a lot of code has been simplified for future versions.

### Breaking changes

#### UploadFS.readAsArrayBuffer() is DEPRECATED

The method `UploadFS.readAsArrayBuffer()` is not available anymore, as uploads are using POST binary data, we don't need `ArrayBuffer`.

```js
UploadFS.selectFiles(function(ev){
    UploadFS.readAsArrayBuffer(ev, function (data, file) {
        let photo = {
            name: file.name,
            size: file.size,
            type: file.type
        };
        let worker = new UploadFS.Uploader({
            store: photosStore,
            data: data,
            file: photo
        });
        worker.start();
    });
});
```

The new code is smaller and easier to read :

```js
UploadFS.selectFiles(function(file){
    let photo = {
        name: file.name,
        size: file.size,
        type: file.type
    };
    let worker = new UploadFS.Uploader({
        store: photosStore,
        data: file,
        file: photo
    });
    worker.start();
});
```

#### Permissions are defined differently

Before `v0.6.1` you would do like this :

```js
Photos.allow({
    insert: function (userId, doc) {
        return userId;
    },
    update: function (userId, doc) {
        return userId === doc.userId;
    },
    remove: function (userId, doc) {
        return userId === doc.userId;
    }
});
```

Now you can set the permissions when you create the store :

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    permissions: new UploadFS.Permissions({
        insert: function (userId, doc) {
            return userId;
        },
        update: function (userId, doc) {
            return userId === doc.userId;
        },
        remove: function (userId, doc) {
            return userId === doc.userId;
        }
    }
});
```

## Testing

You can test the package by downloading and running [UFS-Example](https://github.com/jalik/ufs-example) which is simple demo of UploadFS.

## Installation

To install the package, execute this command in the root of your project :
```
meteor add jalik:ufs
```

If later you want to remove the package :
```
meteor remove jalik:ufs
```

## Plugins

As the package is modular, you can add support for custom stores and even create one, it's easy.

* [UploadFS.store.Local](https://github.com/jalik/jalik-ufs-local)
* [UploadFS.store.GridFS](https://github.com/jalik/jalik-ufs-gridfs)
* [UploadFS.store.WABS](https://github.com/sebakerckhof/ufs-wabs)
* [UploadFS.store.S3](https://github.com/sebakerckhof/ufs-s3)

## Introduction

In file uploading, you basically have a client and a server, I haven't change those things.
So on the client side, you create an uploader for each file transfer needed, while on the server side you only configure a store where the file will be saved.

In this documentation, I am using the `UploadFS.store.Local` store which saves files on the filesystem.

## Configuration

You can access and modify settings via `UploadFS.config`.

```js
// Activate simulation for slowing file reading
UploadFS.config.simulateReadDelay = 1000; // 1 sec

// Activate simulation for slowing file uploading
UploadFS.config.simulateUploadSpeed = 128000; // 128kb/s

// Activate simulation for slowing file writing
UploadFS.config.simulateWriteDelay = 2000; // 2 sec

// This path will be appended to the site URL, be sure to not put a "/" as first character
// for example, a PNG file with the _id 12345 in the "photos" store will be available via this URL :
// http://www.yourdomain.com/uploads/photos/12345.png
UploadFS.config.storesPath = 'uploads';

// Set the temporary directory where uploading files will be saved
// before sent to the store.
UploadFS.config.tmpDir = '/tmp/uploads';
```

## Creating a Store

**All stores must be available on the client and the server.**

A store is the place where your files are saved, it could be your local hard drive or a distant cloud hosting solution.
Let say you have a **photos** collection which is used to save the files info.

```js
Photos = new Mongo.Collection('photos');
```

What you need is to create the store that will will contains the data of the **Photos** collection.
The **name** of the store must be unique because it will be used by the uploader to send the file on the store.

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos'
});
```

## Filtering uploads

You can pass an `UploadFS.Filter` to the store to define restrictions on file uploads.
Filter is tested before inserting a file in the collection and uses `Meteor.deny()`.
If the file does not match the filter, it won't be inserted and will not be uploaded.

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    // Apply a filter to restrict file upload
    filter: new UploadFS.Filter({
        minSize: 1,
        maxSize: 1024 * 1000 // 1MB,
        contentTypes: ['image/*'],
        extensions: ['jpg', 'png']
    })
});
```

If you need a more advanced filter, you can pass a method using the `onCheck` option.

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    // Apply a filter to restrict file upload
    filter: new UploadFS.Filter({
        onCheck: function(file) {
            if (file.extension !== 'png') {
                return false;
            }
            return true;
        }
    })
});
```

## Transforming files

If you need to modify the file before it is saved to the store, you have to use the `transformRead` or `transformWrite` parameter.
A common use is to resize/compress images to get lighter versions of the uploaded files.

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    // Transform file when reading
    transformRead: function (readStream, writeStream, fileId, file, request) {
        readStream.pipe(writeStream); // this returns the raw data
    }
    // Transform file when writing
    transformWrite: function (readStream, writeStream, fileId, file) {
        var im = Npm.require('imagemagick-stream');
        var resize = im().resize('800x600').quality(90);
        readStream.pipe(resize).pipe(writeStream);
    }
});
```

## Copying files (since v0.3.6)

You can copy files to other stores on the fly, it could be for backup or just to have alternative versions of the same file (eg: thumbnails).

```js
Thumbnails = new Mongo.Collection('thumbnails');

PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    copyTo: [
        new UploadFS.store.Local({
            collection: Thumbnails,
            name: 'thumbnails',
            transformWrite: function(readStream, writeStream, fileId, file) {
                var im = Npm.require('imagemagick-stream');
                var resize = im().resize('128x128').quality(80);
                readStream.pipe(resize).pipe(writeStream);
            }
        })
    ]
});
```

You can also manually copy a file to another store by using the `copy()` method.

```js
PhotosStore.copy(fileId, BackupStore, function(err, copyId, copyFile) {
    !err && console.log(fileId + ' has been copied as ' + copyId);
});
```

Copies contains 2 fields that links to the original file, `originalId` and `originalStore`.
So if you want to display a thumbnail instead of the original file you would do like this :

```html
<template name="photos">
    {{#each photos}}
        <img src="{{thumb.url}}">
    {{/each}}
</template>
```

```js
Template.photos.helpers({
    photos: function() {
        return Photos.find();
    },
    thumb: function() {
        return Thumbnails.findOne({originalId: this._id});
    }
});
```

## Setting permissions

If you don't want anyone to do anything, you must define permission rules.
By default, there is no restriction (except the filter) on insert, remove and update actions.

**The permission system has changed since `v0.6.1`, you must define permissions like this :**

```js
PhotosStore.setPermissions(new UploadFS.Permissions({
    insert: function (userId, doc) {
        return userId;
    },
    update: function (userId, doc) {
        return userId === doc.userId;
    },
    remove: function (userId, doc) {
        return userId === doc.userId;
    }
});
```

or when you create the store :

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    permissions: new UploadFS.Permissions({
        insert: function (userId, doc) {
            return userId;
        },
        update: function (userId, doc) {
            return userId === doc.userId;
        },
        remove: function (userId, doc) {
            return userId === doc.userId;
        }
    }
});
```

## Securing file access

When returning the file for a HTTP request, you can do some checks to decide whether or not the file should be sent to the client.
This is done by defining the `onRead()` method on the store.

**Note:** Since v0.3.5, every file has a token attribute when its transfer is complete, this token can be used as a password to access/display the file. Just be sure to not publish it if not needed. You can also change this token whenever you want making older links to be staled.

```html
{{#with image}}
<a href="{{url}}?token={{token}}">
    <img src="{{url}}?token={{token}}">
</a>
{{/with}}
```

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    onRead: function (fileId, file, request, response) {
        // Allow file access if not private or if token is correct
        if (file.isPublic || request.query.token === file.token) {
            return true;
        } else {
            response.writeHead(403);
            return false;
        }
    }
});
```

## Server events

Some events are triggered to allow you to do something at the right moment on server side.

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    // Called when file has been uploaded
    onFinishUpload: function (file) {
        console.log(file.name + 'has been uploaded');
    }
});
```

## Handling errors

On server side, you can do something when there is a store IO error.

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    // Called when a copy error happened
    onCopyError: function (err, fileId, file) {
        console.error('Cannot create copy ' + file.name);
    }
    // Called when a read error happened
    onReadError: function (err, fileId, file) {
        console.error('Cannot read ' + file.name);
    }
    // Called when a write error happened
    onWriteError: function (err, fileId, file) {
        console.error('Cannot write ' + file.name);
    }
});
```

## Reading a file from a store

If you need to get a file directly from a store, do like below :

```js
// Get the file from database
var file = Photos.findOne(fileId);

// Get the file stream from the store
var readStream = PhotosStore.getReadStream(fileId, file);

readStream.on('error', Meteor.bindEnvironment(function (error) {
    console.error(err);
}));
readStream.on('data', Meteor.bindEnvironment(function (data) {
    // handle the data
}));
```

## Writing a file to a store

If you need to save a file directly to a store, do like below :

```js
// Insert the file in database
var fileId = store.create(file);

// Save the file to the store
store.write(stream, fileId, function(err, file) {
    if (err) {
        console.error(err);
    }else {
        console.log('file saved to store');
    }
});
```

## Uploading files

### Uploading from a file

When the store on the server is configured, you can upload files to it.

Here is the template to upload one or more files :

```html
<template name="upload">
    <button type="button" name="upload">Select files</button>
</template>
```

And there the code to upload the selected files :

```js
Template.upload.events({
    'click button[name=upload]': function (ev) {
        var self = this;

        UploadFS.selectFiles(function (file) {
            // Prepare the file to insert in database, note that we don't provide an URL,
            // it will be set automatically by the uploader when file transfer is complete.
            var photo = {
                name: file.name,
                size: file.size,
                type: file.type,
                customField1: 1337,
                customField2: {
                    a: 1,
                    b: 2
                }
            };

            // Create a new Uploader for this file
            var uploader = new UploadFS.Uploader({
                // This is where the uploader will save the file
                store: PhotosStore,
                // Optimize speed transfer by increasing/decreasing chunk size automatically
                adaptive: true,
                // Define the upload capacity (if upload speed is 1MB/s, then it will try to maintain upload at 80%, so 800KB/s)
                // (used only if adaptive = true)
                capacity: 0.8, // 80%
                // The size of each chunk sent to the server
                chunkSize: 8 * 1024, // 8k
                // The max chunk size (used only if adaptive = true)
                maxChunkSize: 128 * 1024, // 128k
                // This tells how many tries to do if an error occurs during upload
                maxTries: 5,
                // The File/Blob object containing the data
                data: file,
                // The document to save in the collection
                file: photo,
                // The error callback
                onError: function (err) {
                    console.error(err);
                },
                onAbort: function (file) {
                    console.log(file.name + ' upload has been aborted');
                },
                onComplete: function (file) {
                    console.log(file.name + ' has been uploaded');
                },
                onCreate: function (file) {
                    console.log(file.name + ' has been created with ID ' + file._id);
                },
                onProgress: function (file, progress) {
                    console.log(file.name + ' ' + (progress*100) + '% uploaded');
                }
                onStart: function (file) {
                    console.log(file.name + ' started');
                },
                onStop: function (file) {
                    console.log(file.name + ' stopped');
                },
            });

            // Starts the upload
            uploader.start();

            // Stops the upload
            uploader.stop();

            // Abort the upload
            uploader.abort();
        });
    }
});
```

Notice : You can use `UploadFS.selectFile(callback)` or `UploadFS.selectFiles(callback)` to select one or multiple files,
the callback is called with one argument that represents the File/Blob object for each selected file.

During uploading you can get some kind of useful information like the following :
 - `uploader.getAverageSpeed()` returns the average speed in bytes per second
 - `uploader.getElapsedTime()` returns the elapsed time in milliseconds
 - `uploader.getRemainingTime()` returns the remaining time in milliseconds
 - `uploader.getSpeed()` returns the speed in bytes per second

### Uploading from an URL

If you want to upload a file directly from a URL, use the `importFromURL(url, fileAttr, storeName, callback)` method.
This method is available both on the client and the server.

```js
var url = 'https://www.google.com/images/branding/googlelogo/1x/googlelogo_color_272x92dp.png';
var attr = { name: 'Google Logo', description: 'Logo from www.google.com' };

UploadFS.importFromURL(url, attr, PhotosStore, function (err, file) {
    if (err) {
        displayError(err);
    } else {
        console.log('Photo saved :', file);
    }
});

```

## Displaying images

After that, if everything went good, you have you file saved to the store and in database.
You can get the file as usual and display it using the url attribute of the document.

Here is the template to display a list of photos :

```html
<template name="photos">
    <div>
        {{#each photos}}
            {{#if uploading}}
                <img src="/images/spinner.gif" title="{{name}}">
                <span>{{completed}}%</span>
            {{else}}
                <img src="{{url}}" title="{{name}}">
            {{/if}}
        {{/each}}
    </div>
</template>
```

And there the code to load the file :

```js
Template.photos.helpers({
    completed: function() {
        return Math.round(this.progress * 100);
    },
    photos: function() {
        return Photos.find();
    }
});
```

## Template helpers

Some helpers are available by default to help you work with files inside templates.

```html
{{#if UFS.isApplication}}
    <a href="{{url}}">Download</a>
{{/if}}
{{#if UFS.isAudio}}
    <audio src="{{url}}" controls></audio>
{{/if}}
{{#if UFS.isImage}}
    <img src="{{url}}">
{{/if}}
{{#if UFS.isText}}
    <iframe src={{url}}></iframe>
{{/if}}
{{#if UFS.isVideo}}
    <video src="{{url}}" controls></video>
{{/if}}
```
