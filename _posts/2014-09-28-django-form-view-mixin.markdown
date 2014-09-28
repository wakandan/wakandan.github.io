---
layout: post
title:  "Django FormView Mixin Confusion"
date:   2014-09-28 
categories: programming django
---
According to [django documentation][mixin] for using mixin, the FormMixin was supposed to handle POST request with an attached form body or at least that was the impression I've got from the example they used. Just to brieftly summary, the usecase was to code an url to display AuthorDetail. GET request will render the details for an author _and_ a form to enter AuthorInterest which is basically a text field. One saves an AuthorInterest by POSTing the comment to that same URL. 

{% highlight python %}

class AuthorInterestForm(forms.Form):
    message = forms.CharField()

class AuthorDisplay(DetailView):
    model = Author

    def get_context_data(self, **kwargs):
        context = super(AuthorDisplay, self).get_context_data(**kwargs)
        context['form'] = AuthorInterestForm()
        return context

class AuthorInterest(SingleObjectMixin, FormView):
    template_name = 'books/author_detail.html'
    form_class = AuthorInterestForm
    model = Author

    def post(self, request, *args, **kwargs):
        if not request.user.is_authenticated():
            return HttpResponseForbidden()
        self.object = self.get_object()
        #What I used to debug
        #import pdb; pdb.set_trace()

        return super(AuthorInterest, self).post(request, *args, **kwargs)

    def get_success_url(self):
        return reverse('author-detail', kwargs={'pk': self.object.pk})

    def form_valid(self, form): #<-- What I ended up doing
        ...
        if form.is_valid():
            form.save()
        ...
        return super(AuthorInterest, self).form_valid(self, form)

class AuthorDetail(View):

    def get(self, request, *args, **kwargs):
        view = AuthorDisplay.as_view()
        return view(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        view = AuthorInterest.as_view()
        return view(request, *args, **kwargs)
{% endhighlight %}

It was interesting to finally understand how this whole mixing thing works in django. Here's my take: 

* `AuthorDetail` will be the entry point (controller) for the intended URL. `get` and `post` methods are to delegate the request to `AuthorDisplay` and `AuthorInterest` respectively
* `AuthorDisplay` inherits from `DetailView` which is to handle displaying a specific object, so no surprise there. They override the `get_context_data` to also querying associated interest with the author. One mistake I had there was *NOT* returning the context itself, which caused the detail page to go blank and blow up because one of `url` reverse links had an empty param in it. 
* Now the interesting bit is in `AuthorInterest` class. It inherits from `SingleObjectMixin` which is (again) to handle a single object - it has `get_object()`, `template_name` and `model` to do that. It inherits from FormView to handle form posting, which is what we needed for the `AuthorInterest` - `form_class`, `get_success_url()` are for that. 

Now, the above code *won't* work when creating an author interest. The reason was, in `FormView` class, the form is not being saved anywhere! It took me quite sometimes to figure out, both by reading the source of `FormView` and by using `pdb`

To use `pdb` to debug django code, put 

{% highlight python %}
import pdb; pdb.set_trace()
{% endhighlight %}

wherever you want to debug, and then run 

{% highlight python %}
python -m pdb manage.py runserver
{% endhighlight %}

(Credit goes [here][pdb debug] for this piece of handy work)

In my application, I wanted to set the interest's owner (which is the logged in user) and also the author this interest is for (which is the current author object). To do that, I overriden the `form_valid` method as above. While it works, there's something I'm still wondering: 

* The above method is not so nice because it does the saving *before* its super class. Even though in this case the super class doesn't save the form yet, it's still kind of awkward
* Why doesn't this case being handled automatically in the `FormView` class? 

In this particular example, even though I finally see how these pieces fit together - the views, the mixin, pdb, method overriden; I feel django documentation was a bit lacking. It didn't menion that the `FormView` doesn't actually do the saving. Hope that piece of text would be added later, or they would use a more realistic use case where this assumption would be demonstrated more clearly.

[mixin]: https://docs.djangoproject.com/en/1.6/topics/class-based-views/mixins/
[pdb debug]: http://ogirardot.wordpress.com/2011/03/15/how-to-debug-django-using-the-python-debugger-pdb/
