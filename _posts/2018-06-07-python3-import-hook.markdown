---
layout: post
title: "Using python3 import hooks to fix import module code on-the-fly"
---

My current company was genenrating scala code from thrift files. The problem was, since the main application code was going to be written in scala, they never bother to make sure to name their thrift attributes to avoid collisions with keywords from other programming languagues other than scala. That prevents me and others from writing client for our services in Python for example. Here's an example:

{% highlight thrift %}
enum ServiceErrorCode {
    None = 0,
    Exception = 1,
    InvalidInput = 3,
    NotFound = 4,
    Duplicate = 5,
    AuthenticationFailed = 10
}
{% endhighlight %}

And here's another example:

{% highlight thrift %}
enum TrueFalse {
    False = 0,
    True = 1,
    Unknown = 2147483647
}
{% endhighlight %}

Having worked on another project where we were using generated code before, I knew that changing code manually and checked in manually-edited code would never work. It just takes too much effort to maintain the modified repo. This time, I was lucky because the target language is python. Python3 supports import hooks where one can add custom logic while importing code. This is python3 could support loading modules from zipfile, from binary code, etc...

The idea is to write a importer object which has 2 parts:

- The module finder: an object with method `find_module(self, fullname, path)` whose job is when given a module name and module path, return a module loader (could be itself) to say that it can process this module. If it returns `None`, it means that this module finder can not process this module
- The module loader: an object with method `load_module(self, name)` whose job is given a module name, load and return a module object. Reaching this step, the importer object has to do the importing (and put the module object into `sys.modules`) or else the module is ignored. That module will not be processsed by other importer in the importer list. This is where most of the magic happens

Some existing application of the import hooks are loading modules from database, from binary code, from zip files, loading over the network and so on. 

Concretely, the code is below:

{% highlight python %}
"""
This file is used to fix syntax error caused by generated thrift code 
on the fly when importing
"""

import logging
import imp
import sys
import pdb
import re
import importlib
from importlib.machinery import BuiltinImporter


class ThriftImporter(object):
    def __init__(self, *args):
        self.module_names = args
        self.path = None

    def find_module(self, fullname, path=None):
        """
        Module finder. If we are to handle the module, return ourself so that we 
        can continue to load the module. Returning None means we don't handle 
        this module
        :param fullname:
        :param path:
        :return:
        """
        # 'ttypes' is the module name where generated code will live in
        if fullname in self.module_names or 'ttypes' in fullname:

            # save the path so that it could be used later by `load_module`
            self.path = path
            return self
        return None

    def repair_code(self, source):
        """
        Fix source code error. This code prefixes 'Service' into each 
        python keyword found in the source code
        TODO: use re instead of code replacing
        :param source:
        :return:
        """
        source = source.replace("None = 0", "ServiceNone = 0")
        source = source.replace("False = 0", "ServiceFalse = 0")
        source = source.replace("True = 1", "ServiceTrue = 0")
        return source

    def load_module(self, name):
        """
        Load the module source code, fix the source and import manually
        :param name:
        :return:
        """
        if name in sys.modules:
            return sys.modules[name]
        # TODO: use os.path instead of manual path manipulation
        module_name = name.split('.')[-1]
        module_path = self.path[0] + '/' + module_name + '.py'

        # create the module spec object
        spec = importlib.util.spec_from_file_location(name, module_path)

        # read the source code and modify on-the-fly
        source = open(module_path).read()
        new_source = self.repair_code(source)

        # create the module object based off the module spec
        module = importlib.util.module_from_spec(spec)

        # compile the source code into a code object where it 
        # could be imported with `exec` call.
        codeobj = compile(new_source, module.__spec__.origin, 'exec')

        # module.__dict__ is required for referencing variables in the module
        exec(codeobj, module.__dict__)

        # put the loaded module into sys.modules so that if the module is imported
        # again it could be found.
        sys.modules[name] = module

        # return the module itself so that it could be used
        return module


# sys.meta_path is where we hook up our thrift importer:
# >>> print(sys.meta_path) => [<class '_frozen_importlib.BuiltinImporter'>, <class '_frozen_importlib.FrozenImporter'>, <class '_frozen_importlib_external.PathFinder'>]
# we need to put our importer just before the last importer so that we don't have
# to handle importing built-in modules
sys.meta_path.insert(-1, ThriftImporter('ttypes'))
{% endhighlight %}

What I was surprised to find was that there was not an existing class or module to do this already, as you can see that we are not extending our `ThriftImporter` from anywhere besides our plain old `object`.

Reference: 
- [Main Stackoverflow link](https://stackoverflow.com/questions/41858147/how-to-modify-imported-source-code-on-the-fly)
- [python3 documentation](https://docs.python.org/3/reference/import.html)
- [Stackoverflow link 1](https://stackoverflow.com/questions/43571737/how-to-implement-an-import-hook-that-can-modify-the-source-code-on-the-fly-using)
- [xion.log blog post](http://xion.org.pl/2012/05/06/hacking-python-imports/)
