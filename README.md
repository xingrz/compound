Installation
============

It is fine to pull from github (less bugs, I hope)

    $ git clone git://github.com/1602/express-on-railway.git
    $ cd express-on-railway
    $ npm install
    $ cd -
    $ rm -rf express-on-railway

Or install from npm registry:

    $ npm install express-on-railway

This package depends on express, ejs and node-redis-mapper

Usage
=====

    $ mkdir blog && cd blog
    $ railway init

or, slightly simplier, but with the same result

    $ railway init blog

Short functionality review
==========================

Directory structure
-------------------

On initialization rails-like directories tree generated, like that:

    .
    |-- app
    |   |-- controllers
    |   |   |-- admin
    |   |   |   |-- categories_controller.js
    |   |   |   |-- posts_controller.js
    |   |   |   `-- tags_controller.js
    |   |   |-- comments_controller.js
    |   |   `-- posts_controller.js
    |   |-- models
    |   |   |-- category.js
    |   |   |-- post.js
    |   |   `-- tag.js
    |   |-- views
    |   |   |-- admin
    |   |   |   `-- posts
    |   |   |       |-- edit.ejs
    |   |   |       |-- index.ejs
    |   |   |       |-- new.ejs
    |   |   |-- layouts
    |   |   |   `-- application_layout.ejs
    |   |   |-- partials
    |   |   `-- posts
    |   |       |-- index.ejs
    |   |       `-- show.ejs
    |   `-- helpers
    |       |-- admin
    |       |   |-- posts_helper.js
    |       |   `-- tags_helper.js
    |       `-- posts_helper.js
    `-- config
        |-- database.json
        `-- routes.js

Routing
-------

Now we do not have to tediously describe REST rotes for each resource, enough to write in `config / routes.js` code like this:

    exports.routes = function (map) {
        map.resources('posts', function (post) {
            post.resources('comments');
        });
    };

instead of:

    var ctl = require('./lib/posts_controller.js');
    app.get('/posts/new.:format?', ctl.new);
    app.get('/posts.:format?', ctl.index);
    app.post('/posts.:format?', ctl.create);
    app.get('/posts/:id.:format?', ctl.show);
    app.put('/posts/:id.:format?', ctl.update);
    app.delete('/posts/:id.:format?', ctl.destroy);
    app.get('/posts/:id/edit.:format?', ctl.edit);

    var com_ctl = require('./lib/comments_controller.js');
    app.get('/posts/:post_id/comments/new.:format?', com_ctl.new);
    app.get('/posts/:post_id/comments.:format?', com_ctl.index);
    app.post('/posts/:post_id/comments.:format?', com_ctl.create);
    app.get('/posts/:post_id/comments/:id.:format?', com_ctl.show);
    app.put('/posts/:post_id/comments/:id.:format?', com_ctl.update);
    app.delete('/posts/:post_id/comments/:id.:format?', com_ctl.destroy);
    app.get('/posts/:post_id/comments/:id/edit.:format?', com_ctl.edit);

and you can more finely tune the resources to specify certain actions, middleware, and other. Here example routes of [my blog][1]:

    exports.routes = function (map) {
        map.get('/', 'posts#index');
        map.get(':id', 'posts#show');
        map.get('sitemap.txt', 'posts#map');
    
        map.namespace('admin', function (admin) {
            admin.resources('posts', {middleware: basic_auth, except: ['show']}, function (post) {
                post.resources('comments');
                post.get('likes', 'posts#likes')
            });
        });
    };

for debugging routes described in `config/routes.js` I have written jake-task that generates the following output:

    $ jake routes
                     GET    /                               posts#index
                     GET    /:id                            posts#show
         sitemap.txt GET    /sitemap.txt                    posts#map
         admin_posts GET    /admin/posts.:format?           admin/posts#index
         admin_posts POST   /admin/posts.:format?           admin/posts#create
      new_admin_post GET    /admin/posts/new.:format?       admin/posts#new
     edit_admin_post GET    /admin/posts/:id/edit.:format?  admin/posts#edit
          admin_post DELETE /admin/posts/:id.:format?       admin/posts#destroy
          admin_post PUT    /admin/posts/:id.:format?       admin/posts#update
    likes_admin_post PUT    /admin/posts/:id/likes.:format? admin/posts#likes


Helpers
-------

In addition to regular rails helpers `link_to`, `form_for`, `javascript_include_tag`, `form_for`, etc. there are also helpers for routing: each route generates a helper method that can be invoked in a view:

    <%- link_to("New post", new_admin_post) %>
    <%- link_to("New post", edit_admin_post(post)) %>

generates output:

    <a href="/admin/posts/new">New post</a>
    <a href="/admin/posts/10/edit">New post</a>

Controllers
-----------

The controller is a module containing the declaration of actions such as this:

    beforeFilter(loadPost, {only: ['edit', 'update', 'destroy']});

    action('index', function () {
        Post.allInstances({order: 'created_at'}, function (collection) {
            render({ posts: collection });
        });
    });

    action('create', function () {
        Post.create(req.body, function () {
            redirect(path_to.admin_posts);
        });
    });

    action('new', function () {
        render({ post: new Post });
    });

    action('edit', function () {
        render({ post: request.post });
    });

    action('update', function () {
        request.post.save(req.locale, req.body, function () {
            redirect(path_to.admin_posts);
        });
    });

    function loadPost () {
        Post.find(req.params.id, function () {
            request.post = this;
            next();
        });
    }

## Generators ##

Railway offers several built-in generators: for a model, controller and for 
initialization. Can be invoked as follows:

    railway generate [what] [params]

`what` can be `model`, `controller` or `scaffold`. Example of controller generation:

    $ railway generate controller admin/posts index new edit update
    exists  app/
    exists  app/controllers/
    create  app/controllers/admin/
    create  app/controllers/admin/posts_controller.js
    create  app/helpers/
    create  app/helpers/admin/
    create  app/helpers/admin/posts_helper.js
    exists  app/views/
    create  app/views/admin/
    create  app/views/admin/posts/
    create  app/views/admin/posts/index.ejs
    create  app/views/admin/posts/new.ejs
    create  app/views/admin/posts/edit.ejs
    create  app/views/admin/posts/update.ejs

Currently it generates only *.ejs views

Models
------

At the moment I store objects in redis data store. For that purpose I have
written simple driver, that adds persistence-related methods to models described
in app/models/*.js. I can work with models the following way:

File `app/models/post.js`:

    var Post = describe('Post', function () {
        property('title',   String);
        property('preview', String);
        property('content', String);
        property('tags',    Array);
    });

In controller:

    // create new object
    Post.create(params, function () {
        console.log(post.id);
        console.log(post.created_at);
    });

    // find by primary key
    Post.find(params.id, function (err) {
        if (!err) {
            this.update_attributes({
                title: 'Hello world',
                preview: 'asda',
                tags: 'world,hello,example,redis-mapper,find'.split(',')
            });
        }
    });

    // collection
    Post.allInstances(function (posts) {
        posts.forEach(function (post) {
            console.log(post.title);
        });
    });

REPL console
------------

To run REPL console use command

    railway console

or it's shortcut

    railway c

It just simple node-js console with some Railway bingings, e.g. models. Just one note
about working with console. Node.js is asunchronous by his nature, and it's great
but it made console debugging much more complicated, because you should use callback
to fetch result from database, for example. I have added one useful method to
simplify async debugging using railway console. It's name `c`, you can pass it
as parameter to any function requires callback, and it will store parameters passed
to callback to variables `_0, _1, ..., _N` where N is index in `arguments`.

Example:

    railway c
    railway> User.find(53, c)
    Callback called with 2 arguments:
    _0 = null
    _1 = [object Object]
    railway> _1
    { email: [Getter/Setter],
      password: [Getter/Setter],
      activationCode: [Getter/Setter],
      activated: [Getter/Setter],
      forcePassChange: [Getter/Setter],
      routesCount: [Getter/Setter],
      isAdmin: [Getter/Setter],
      id: [Getter/Setter] }


MIT License
===========

    Copyright (C) 2011 by Anatoliy Chakkaev
    
    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:
    
    The above copyright notice and this permission notice shall be included in
    all copies or substantial portions of the Software.
    
    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
    THE SOFTWARE.


  [1]: http://node-js.ru
