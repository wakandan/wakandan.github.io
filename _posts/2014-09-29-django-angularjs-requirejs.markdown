---
layout: post
title:  "Use Angularjs + RequireJs with Django"
date:   2014-09-29
categories: programming django
---
For the last few days I have been trying to add [AngularJS][AngularJS] into [Django][Django] template. From my past experience with angularjs projects, usually as the projects grow, the number of js files needed to be included into the base html template explodes and it's a pain to manage. I know [RequireJS][RequireJS] would solve that problem, but never had the chance/time to come around and try it out. Finally today I had something working.

Before we move on, if you are reading this post you must already know the fact that using angularjs with django is _NOT_ easy. Some even uses "barely worths the effort". The main problem was that both django and angular uses {%raw%} {{var}} {%endraw%} to bind a variable in a template. Some suggested to modify angular template tag into something like `{[{}]}` so as to not collide with django, but obviously this approach won't scale. Your app will definitely use 3rd party plugins; changing the default template tags will render plugins' templates useless.

Below is some finding/experiments I did with django + requirejs + angularjs to figure out how to use angularjs value binding in django template & the use of requirejs to organize angularjs modules.

Use Angularjs with django template
==================================

So why is the collision between django template and angularjs template? It probably happens because one is trying to put angularjs template code _INSIDE_ django template. Do we really have to do that? Why can't we use `ng-include` to load the template with angular code instead of writting it inside django template directly? Since `ng-include` is valid html, there would be no collision. Of course, it would be nice to _NOT_ do it, but this approach really removes a lot of headaches.

Here is my sample code:

{% highlight html %}
<!--inside your django template-->
...
<div ng-include="'/static/scripts/blog/templates/TestCtrl.html'">
    <!--this div should be empty-->
</div>
...


<!--/static/scripts/blog/templates/TestCtrl.html-->
<div ng-controller='myApp.blog.controller.TestCtrl'>
     <p>This is TestCtrlTpl template. Below is $scope.helloWorld</p>
     <p>
     {{helloWorld}}
     </p>
     <input type="text" ng-model='value1'/>
     <input type="text" ng-model='value2'/>
     <input type="text" ng-value='value1+value2'/>
 </div>
...

<!--TestCtrl-->
$scope.helloWorld = 'Hello World';

{% endhighlight %}

Note that I use _absolute path_ to the template. Using relative paths doesn't really work, since django manage the static root for us. Oh, and don't forget the enclosing `' '`. Failing to do so will result in `ng-include` code does't get compiled correctly, which turns the element into a comment. I had no idea why. Regardless of how long I have worked with angularjs, this bit always caught me off guard.

Content of `TestCtrl.html` and `TestCtrl` is basic stuff - trying to test angularjs binding is working. Also notice the long controller name, it will bring us to the second goal of this post

Organize angularjs modules using requirejs:
===========================================

In my preliminary research, I came across [this post][angularjs requirejs] which really gets me started on using requirejs. Requirejs was a bit hard to understand, but I kind of get what it can do on the surface after reading the abovementioned article.

When I first got started with angularjs, forking from [angular-seed][angular-seed] or similar seed projects usually assumes one kind of structures: angularjs code is organized by _module types_. There will be a top level `scripts` folder, under that will be `controllers`, `directives`, `services` and so on. Been there before, I must say this approach doesn't work really well with a large project which is composed of a lot of features. May be I'm not fast on updating, but organizing by _features_ is actually the trend now. I think MEAN stack in its early days used module-type-based approach, but last time I checked it seems to be using feature-based approach now.

Ok, what do I mean by feature-based approach. Let's take the example from my play app, I have 2 simple features: `common` and `blog`. Using this approach, all controllers, services, etc... are organized into those features. `common` is as its name suggests, for common code that can be used system wide resides. `blog` is separate feature of the app. It might follow this structure:

{% highlight bash %}
├── app.js          #entry point for angularjs
├── blog
│   ├── routes.js 
│   ├── controllers
│   ├── directives
│   ├── services
│   ├── templates 
│   └── .. and so on
├── common 
│   ├── routes.js 
│   ├── controllers
│   ├── directives
│   ├── services
│   ├── templates 
│   └── .. and so on
└── .. some other files
{% endhighlight %}

Onto Requirejs: it organizes files into modules. Modules generally have this structure

{% highlight javascript %}
define([
    'bootstrap',            //predefined path, more below 
    './module',             //load file module.js in the same folder
    '../parent_module',     //load file parent_module.js in the parent folder
    ...
], function(
    bs      //bootstrap loaded defined and aliased
){
    //do something interesting here
}
{% endhighlight %}

Following the structure in the article, here's the file organization experiment I had and what I understood from it

{% highlight bash %}
#content of the django's static folder
➜  static git:(master) ✗ ls
bower_components    #store bower dependencies
main.js             #entry point for requirejs 
scripts             #all java script files 
styles              #style files 
➜  static git:(master) ✗ cd scripts 
➜  scripts git:(master) ✗ tree
.
├── app.js          #entry point for angularjs
├── blog
│   ├── controllers
│   │   └── testCtrl.js
│   ├── module.js
│   ├── module.require.js
│   ├── namespace.js
│   └── templates
│       └── TestCtrl.html
├── common
│   ├── module.js
│   ├── module.require.js
│   ├── namespace.js
│   └── templates
└── namespace.js

{% endhighlight %}

Primary content of javascripts files in this structure:

* `namespace.js`: Define the module name which might be derived from parent module name and will be used by children modules when defining these modules.
* `module.js`: Module declaration lives here (`angular.module()...`)
* `module.require.js`: Logical structure to be required by requirejs, importing (and subsequently loading) needed modules

I will explain the files by their functions. I will only go into details about the `blog` module, the `common` module is using the same approach. First is the namespace files:

{% highlight javascript %}
//namespace.js: define app name
define([], function(){
    'use strict';
    return 'myApp';
})

//blog/namespace.js: avoid hard coding module names
define(['../namespace'], function(namespace){
    'use strict';
    return namespace+'.blog';   //define blog's module name = 'myApp.blog'
})

//blog/namespace.js is used in blog/module.js like this:
//blog/module.js
define([
    'angular',
    './namespace',
    ], function(angular, namespace){
        'use strict';
        return angular.module(namespace, []); <-- important
    })
{% endhighlight %}

Good usage example of the namespace file is the `blog/namespace.js` file. It depends on the top app namespace file - return `myApp`, and is used by `blog/module.js`. The blog module name therefore will be `myApp.blog`.

Second is the module files. Most interesting module file was `blog/module.js` which was already shown above. Essentially it just loads angularjs & the module name, then defines an angular module module with it. Nothing fancy there.

Third is the module.require files. They are the entry point for requirejs to load modules in a systematic manner. I had a hard time following the require chain, so I hope to remove some of that for you when you use it.

{% highlight javascript %}
//blog/module.require.js
define([
    './module',
    './controllers/testCtrl'
], function(module){
    'use strict';
    //test controller should be loaded by now
})

//blog/controller/testCtrl.js
define([
    '../module',
    '../namespace'
], function(module, namespace){
    'use strict';
    module.controller(namespace + '.controller.TestCtrl', ['$scope', function($scope){
        console.log('controller initialized', namespace)
        $scope.helloWorld = 'Hello World!';
    }])    
})

{% endhighlight %}


`blog/module.require.js` 1) loads blog module declaration in `module.js` and 2) loads `TestCtrl` defined in `./controllers/testCtrl`. Whatever file depends on this `blog/module.require.js` file will load these two on the process.

`testCtrl` will need the 1) namespace to make its own controller name and 2) the `myApp.blog` module itself for angularjs declaration. After that it's just normal angularjs code. The author of mentioned article seems to suggest to go further and define blog controller's module, namespace and module.require but in my opinion it's not necessary since under the feature/module itself, the file functions are already separated by their (well-known) names (e.g: add .controllers, .directives,.. into module names) anyway.

And lastly, the `app.js` file and `main.js` file:

{% highlight javascript %}
//app.js
define([
    'angular',
    './namespace',
    './common/namespace',
    './blog/namespace',
    './common/module.require',
    './blog/module.require'
], function(
    angular, 
    namespace, 
    namespaceCommon, 
    namespaceBlog
) {
    var app = angular.module(namespace, [
            namespaceCommon, 
            namespaceBlog
        ]);
    'use strict';
    app.run(function(){

    })
    return app;
})

//main.js
require.config({
    paths: {
        jquery: 'bower_components/jquery/dist/jquery.min',
        underscore: 'bower_components/underscore/underscore-min',
        bootstrap: 'bower_components/bootstrap/dist/js/bootstrap.min',
        angular: 'bower_components/angular/angular.min',
        less: 'bower_components/less/dist/less-1.7.5.min'
    },
    shim: { 
        angular: {
            exports: 'angular',
            deps: ['jquery']
        }, 
        bootstrap: {
            deps: ['jquery']
        }
    }
})

require([
    'angular',
    './scripts/namespace',
    './scripts/app',
    'jquery',
    'bootstrap', 
    'underscore',
    'less'
], function(angular, namespace, app) {
    angular.element(document).ready(function(){
        //bootstrap angular
        angular.bootstrap(document, [namespace]) 
    }) 
})
{% endhighlight %}

`app.js` defines the top angularjs app. It uses the namespace file for getting its name. It loads common and blog module names for the module declarations. It declares dependency on common module & blog module to make sure those modules are loaded before the app is declared 

`main.js` is the most important file, since it's the entry point for the whole app. In the `.config` part, it defines paths for downloaded bower modules. `Shim` is for defining the order of loading - we tells it to load jquery before loading angularjs and bootstrap. 

In the `require` part, it: 

* requires `angular` for bootstrapping the app. Remember that manually bootstrapping the app this way, you don't have to use `ng-app` anywhere in the template 
* loads `namespace` for the bootstrap declaration
* loads `app` to make sure the angularjs app is loaded before bootstrapping
* and loads other 3rd party modules (jquery, bootstrap, underscore...) as well

Summary:
========

To summary this process, breath-wise:

* Define `namespace.js` files to handle module names without hardcoding
* Define `module.js` files that use these `namespace.js` file to _declare_ the modules. It will also be used by children modules as well.
* Then lastly, defines `module.require.js` to make sure `module.js` and all children modules get loaded.
* In `app.js`, declare the main angularjs app and loads its depedencies which are all the `module.require.js` files
* In `main.js`, loads the main app and all 3rd party modules 

Depth-wise:

* Define `namespace.js` files first
* Define `module.js` files for the main modules (blog, common) using the namespace files 
* Define `module.require.js` to load written `module.js` file
* Then define `app.js` and load it in `main.js`

Conclusion: 
===========

I couldn't get a hang of requirejs before. But through this experiment, I kind of get what it does, and also set up a nice approach for using django + requirejs + angularjs + less for future projects 

[angularjs requirejs]: http://thaiat.github.io/blog/2014/02/26/angularjs-and-requirejs-for-very-large-applications/
[AngularJS]: https://angularjs.org/
[Django]: https://www.djangoproject.com/
[RequireJS]: http://requirejs.org/
[angular-seed]: https://github.com/angular/angular-seed
