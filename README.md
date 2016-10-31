
# node-sftp-server

A simple interface to be able to implement an SFTP Server using Node.js. Based 
on excellent work by [@mscdex](https://github.com/mscdex) - [ssh2](https://github.com/mscdex/ssh2) and [ssh2-streams](https://github.com/mscdex/ssh2-streams). Without
which none of this would be possible.

In all cases, this library will only ever perform a subset of what can be 
accomplished with [ssh2](https://github.com/mscdex/ssh2). If there's something more advanced you need
to do and this library won't support it, that one is probably the one to look 
at. And certainly pull requests would be welcome, too!

The easiest way to get the hang of this library is probably to look at the 
`server_example.js` to start with, until this documentation gets more fully
fleshed-out.

# Usage

```js
var SFTPServer=require('node-sftp-server');
```

## SFTPServer Object

### constructor

```js
var myserver=new SFTPServer("path_to_private_key_file");
```

This returns a new `SFTPServer()` object, which is an EventEmitter. If the private
key is not specified, the constructor will try to use `ssh_host_rsa_key` in the current directory.

### methods 
```js
.listen(portnumber)
```
Listens for an SFTP client to connect to the server on this port.

### events
`connect` - passes along a simple context object which has - 

- username: 
- password:
- method: 

You can call `.reject()` to reject the connection, or call `.accept(callback)`
to work with the new connection. The callback will be passed a Session object
as its parameter.

`end` - emitted when the user disconnects from the server.

## Session Object

This object is passed to you when you call `.accept(callback)` - your callback
should expect to be passed a session object as a parameter. The session object
is an EventEmitter as well.

### events

`.on("realpath",function (path,callback) { })` - the server wants to determine the 'real' path
for some user. For instance, if a user, when they log in, is immediately deposited
into `/home/<username>/` - you could implement that here. Invoke the callback 
with the calculated path - e.g. `callback("/home/"+username)`.  *TODO* - Error 
management here!

`.on("stat",function (path,statkind,statresponder) { })` - on any of STAT, LSTAT, or FSTAT
requests (the type will be passed in "statkind"). The statresponder object is a `Statter`
object from the source code. Communicate status back by calling methods and setting properties
on the object like this:

#### Directory
```js
session.on('stat', function(path, statkind, statresponder) {
    statresponder.is_directory();    // Tells statresponder that we're describing a directory.
    statresponder.permissions = 755; // Octal permissions, like what you'd send to a chmod command
    statresponder.uid = 1;           // User ID that owns the file.
    statresponder.gid = 1;           // Group ID that owns the file.
    statresponder.size = 0;          // File size in bytes.
    statresponder.atime = 123456;    // Created at (unix style timestamp in seconds-from-epoch).
    statresponder.mtime = 123456;    // Modified at (unix style timestamp in seconds-from-epoch).

    stat.file();   // Tells the statter to actually send the values above down the wire.
});
```

#### File
```js
session.on('stat', function(path, statkind, statresponder) {
    statresponder.is_file();         // Tells statresponder that we're describing a file.
    statresponder.permissions = 644; // Octal permissions, like what you'd send to a chmod command
    statresponder.uid = 1;           // User ID that owns the file.
    statresponder.gid = 1;           // Group ID that owns the file.
    statresponder.size = 1234;       // File size in bytes.
    statresponder.atime = 123456;    // Created at (unix style timestamp in seconds-from-epoch).
    statresponder.mtime = 123456;    // Modified at (unix style timestamp in seconds-from-epoch).

    stat.file();   // Tells the statter to actually send the values above down the wire.
});
```

#### Errors

You can also respond with file not found messages like this:

```js
session.on('stat', function(path, statkind, statresponder) {
    statresponder.noFile(); // Tells the statter to send a file not found stat down the wire.
});
```

`.on("readdir",function (path,directory_emitter) { })` - on a directory listing attempt, the
directory_emitter will keep emitting `dir` messages with a `responder` as a
parameter, allowing you to respond with `responder.file(filename)` to return
a file entry in the directory, or `responder.end()` if the directory listing
is complete.

`.on("readfile",function (path,writable_stream) { })` - the client is attempting to read a file
from the server - place or pipe the contents of the file into the `writable_stream`.

`.on("writefile",function (path,readable_stream) { })` - the client is attempting to write a 
file to the server - the `readable_stream` corresponds to the actual file. You
may `.pipe()` that into a writable stream of your own, or use it directly.

`.on("delete",function (path,callback) { })` - the client wishes to delete a file. Respond with
`callback.ok()` or `callback.fail()` or any of the other error types

## Error Callbacks

Many of the session events pass some kind of 'responder' or 'callback' object
as a parameter. Those typically will have several error conditions that you can
use to refuse the request - 

- `responder.fail()` - general failure?
- `responder.nofile()` - no such file or directory
- `responder.denied()` - access denied
- `responder.bad_message()` - protocol error; bad message (unusual)
- `responder.unsupported()` - operation not supported
- `responder.ok()` - success
