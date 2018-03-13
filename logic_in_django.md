# How I Django

# Keeping views lean

## I use custom Querysets a lot

```
# query.py
class MemberQuerySet(models.QuerySet):
    """Member custom queryset."""

    def active(self):
        """Return any member that is active."""
        return self.filter(activate_date__isnull=False, deactivate_date__isnull=True)

    def with_num_books(self):
        """Return queryset with annotated book number."""
        return self.annotate(amount=Sum('books'))

```

```
# views.py
class Members(ListView):
    """Show members."""

    def get_queryset(self):
        """Only show members who are active. Also display number of books for each"""
        return Member.objects.active().with_num_books()

```

Sometimes I also use custom Managers in conjunction with QuerySets, particularly when I am returning aggregates, since QuerySets should return querysets and in a manager anything goes.


# When using forms, keep the processing in the forms not the view.
For standard forms I usually create a process() method. For ModelForms I 
hook into save.

```
# forms.py
class SomeForm(forms.Form):
    blah = fields.CharField()

    def process():
        """Do something with the fields"""
        pass
```


```
# views.py
 
 class SomeBlahView(FormView):
    form_class = SomeForm

    def form_valid(self, form):
        response = super().form_valid()
        form.process()
        return response
```


## Use Class Based Views
I use CBVs as the code is much cleaner, each method is self explanatory. I can also use mixins to keep code DRY

```
class MemberDashboard(MustBeMemberMixin, TemplateView):
    pass
```


# Keep models thin

I include fields and a few properties here and there.

```
# models.py

class Member(models.Model):
    first_name = models.CharField(max_length=255)
    last_name = models.CharField(max_length=255)

    def __str__(self):
        """Return first name as string repr."""
        return self.first_name

    @property
    def name(self):
        """Return full name."""
        return '{} {}'.format(self.first_name, self.last_name) 
```

# But where does the logic go?

*Utility classes*

I call my utility classes *Agents*. The name **Agent** seems to give more
importance then just utility. I also frequently have a utils.py file
with some helpers, so I want to separate that bit.

I don't use the word Service since that seems to be an established pattern that doesn't fit into Django the way people think (from what I understand).
Though I hear agent is also a pattern, so use whatever word works best for you.

I usually have an app that holds these agent files. I prefer not to have them 
in the model they belong to since they may need to work together with lots of 
other models from other apps.

Here is an example:

```
agents.py
from notifications import notify_member, notify_staff

class MemberAgent(object):
    """Business logic for members."""

    def __init__(self, member):
        self.member - member

    def new_member(self):
        notify_member.member_activated(self.member)
        notify_staff.member_activated(self.member)
        self.create_additional_settings()
        self.offer_free_stuff()

    def create_additional_settings(self):
        """Do some logic here.""
        pass

    def offer_free_stuff(self):
        """Do some logic here.""
        pass

```

I then call these agents wherever I need to.
In forms, in views, in serializers, in admin forms

The best thing about these agents is that it is very easy to hook into any Django part. We can still let Django handle the heavy lifting of model creation using ModelForms or API Serializers, and then just trigger what needs to be done after.

No longer do you need to search through signals all over the place.

```
# forms.py

class MemberCreateForm(forms.ModelForm):
    """Form for creating members."""

    class Meta:  # noqa D201
        model = Member
        fields = ['first_name', 'last_name']

    def save(self, commit=True):
        """Save the member and delegate to agent to handle the rest."""
        instance = super().save(commit)
        member_agent = MemberAgent(instance)
        member_agent.new_member()
        return instance
```

or in views

```
class SomeView(FormView):
    """Some fancy view."""

    def form_valid(self, form):
        """When the member fills in the form they get free stuff."""
        response = super().form_valid()
        form.process()
        member_agent = MemberAgent(self.request.user.member)
        num_free_stuff = member_agent.offer_free_stuff()
        messages.success(self.request, 'You got {} free thing(s).'.format(num_free_stuff))
        return response
```

The other benefit is that you can start with putting logic in these classes and as you app grows you can delegate the work to other modules. You can even establish a full on design pattern.
Start small,and let it grow.

A really easy example is:

Let's say at the start you have to send a lot of notitications to staff, so you have this kind of code sprinkled all over the place:

```
send_mail("New member", msg, from_email, to_email)
```

Your app has been growing and now you need to use celery to send emails, or maybe you've switched to some custom mail that sends templates emails but you need to use a different method.
Now you need to go back through all of your code and find and replace these.

or

If you used an utils/agent class, then you're changing things in one place.
Or maybe you even used a utils/agent class to call a notifications app since you had a lot of notifications in the first place, and now it's even easier.


# Seperate models apps from view apps

Organize your apps around actions

For example, if you have a product app with some models, and a categories app with some category models, then create a catalogue app to handle the views.

The larger your project grows the more cross referencing tends to happen

```
catalogue
    forms.py
    views.py
    urls.py
product
    models.py
    query.py
    manager.py
categories
    models.py
    query.py
```

This way we don't have too many cross imports.


# Have a core/common app

There are always files/code that needs to be shared across all the apps.
```
core
    templatetags
    management/commands
    abstract_models.py
    fields.py
    mixins.py
    pdf.py
    utils.py
    widgets.py
```
