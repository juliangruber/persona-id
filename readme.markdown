# persona-id

Easily use [mozilla persona](https://login.persona.org/) for authentication.

# example

First we can make a simple core http server with some simple in-memory sessions.
When the user is signed in, we'll update the `#whoami` element on the page to
show their persona id.

``` js
var http = require('http');
var fs = require('fs');
var hyperstream = require('hyperstream');
var ecstatic = require('ecstatic')(__dirname + '/static');

var persona = require('persona-id')('http://localhost:5000');

var sessions = {};
persona.on('create', function (sid, id) {
    sessions[sid] = id.email;
});

persona.on('destroy', function (sid) {
    delete sessions[sid];
});

var server = http.createServer(function (req, res) {
    if (persona.test(req)) {
        persona.handle(req, res);
    }
    else if (req.url === '/') {
        var sid = persona.getId(req);
        fs.createReadStream(__dirname + '/static/index.html')
            .pipe(hyperstream({ '#whoami': sessions[sid] || '' }))
            .pipe(res)
        ;
    }
    else ecstatic(req, res)
});
server.listen(5000);
```

Now for the browser code:

``` js
var http = require('http');
var persona = require('persona-id')();

persona.on('login', function (id) {
    button.value = 'unidentify';
    whoami.textContent = id;
});

persona.on('logout', function () {
    button.value = 'identify';
    whoami.textContent = '';
});

var button = document.getElementById('identify');
var whoami = document.getElementById('whoami');

var who = whoami.textContent;
persona.set(who);

button.addEventListener('click', function () {
    if (!persona.id) {
        persona.identify();
    }
    else persona.unidentify();
});
```

Here we make an identify button and read the `#whoami` element that the server
set to see if the server already knows who we are. You can use whatever
mechanism you want to `.set()` the identity when the server knows it.

Then just compile the browser code with browserify:

```
$ browserify browser.js > static/bundle.js
```

and run the server:

```
$ node server.js
```

You now have a login button that persists across page loads!

It should be easy to extend the in-memory session to persist sessions to disk on
your own.

# methods

``` js
var persona = require('persona-id')
```

## server methods

### var p = persona(opts)

Create a new persona instance `p` from some options `opts`:

* opts.audience - the host:port of your server (mandatory)
* opts.prefix - the prefix to use for internal routes, default: `'_persona'`
* opts.verify - the verification endpoint to use, default:
https://verifier.login.persona.org/verify
* opts.sessionName - the session cookie name to use, default:
`'_persona_session_id'`.

If `opts` is a string, its value will be used as the `opts.audience`.

### p.test(req)

Return whether `p` knows how to handle the `req.method` and `req.url`.

### p.handle(req, res)

Handle the request, reading request data and sending responses for login and
logout requests.

### p.getId(req)

Return the session id for `req.headers.cookie` given the `sessionName` set by
`persona(opts)`.

## server events

### p.on('create', function (sid, id) {})

When a new session is created, this event fires with the session id string `sid`
and the `id` object, which has these properties: id.email, id.audience,
id.expires, id.issuer.

### p.on('destroy', function (sid) {})

When a user logs out, this event fires with the session id.

## browser methods

### var p = persona()

Create a new persona instance `p`.

### p.identify(opts)

Sign in. [Options](https://developer.mozilla.org/en-US/docs/Web/API/navigator.id.request#Parameters) can be used to customize the Persona dialog.

### p.unidentify()

Sign out.

### p.set(email)

Set the persona id `email` string for when the server already knows from session
data that the user is authenticated.

## browser events

### p.on('login', function (id) {})

When the user is known to be authenticated, this event fires with the persona id
string `id`.

### p.on('logout', function () {})

When the user is known to be unauthenticated, this event fires.

Note that this event fires when `p.set(undefined)` is called too.

# install

With [npm](https://npmjs.org) do:

```
npm install persona-id
```

# license

MIT
