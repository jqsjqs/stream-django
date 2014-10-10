##Stream Django

This package helps you create activity streams & newsfeeds with Django and GetStream.io.

###What can you build?

<p align="center">
  <img src="https://dvqg2dogggmn6.cloudfront.net/images/mood-home.png" alt="Examples of what you can build" title="What you can build"/>
</p>

* Activity streams such as seen on Github
* A twitter style newsfeed
* A feed like instagram/ pinterest
* Facebook style newsfeeds
* A notification system

### Demo

You can check out our example app built using this library on [https://exampledjango.getstream.io](https://exampledjango.getstream.io) the code of the example app is available on Github [https://github.com/GetStream/Stream-Example-Py](https://github.com/GetStream/Stream-Example-Py)

###Table of Contents
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [What can you build?](#what-can-you-build)
- [Demo](#demo)
- [Installation](#installation)
- [Model integration](#model-integration)
- [Feed manager](#feed-manager)
- [Showing the newsfeed](#showing-the-newsfeed)
  - [Activity enrichment](#activity-enrichment)
  - [Templating](#templating)
- [Settings](#settings)
- [Temporarily disabling the signals](#temporarily-disabling-the-signals)
- [Customizing enrichment](#customizing-enrichment)
- [Low level APIs access](#low-level-apis-access)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

###Installation

Install stream_django package with pip:

```pip install stream_django```

add stream_django to your ```INSTALLED_APPS```

```
INSTALLED_APPS = [
    ...
    'stream_django'
]
```

Login with Github on getstream.io and add
```STREAM_API_KEY``` and ```STREAM_API_SECRET``` to your Django settings module (you can find them in the dashboard).

###Model integration

Stream Django can automatically publish new activities to your feed. Simple mixin the Activity class on the models you want to publish.

```
from django_stream.activity import Activity

class Tweet(models.Model, Activity):
    ...
    
class Like(models.Model, Activity):
    ...
```

Every time a Tweet is created it will be added to the user's feed. Users which follow the given user will also automatically get the new tweet in their feeds.

####Activity fields

Models are stored in feeds as activities. An activity is composed of at least the following fields: **actor**, **verb**, **object**, **time**. You can also add more custom data if needed.
The Activity mixin will try to set things up automatically:

**object** is a reference to the model instance  
**actor** is a reference to the user attribute of the instance  
**verb** is a string representation of the class name

By default the actor field will look for an attribute called user or actor.
If you're user field is called differently you'll need to tell us where to look for it.
Below shows an example how to set things up if your user field is called author.

```
class Tweet(models.Model, Activity):
    author = models.ForeignKey(settings.AUTH_USER_MODEL)

    @property
    def actor_id(self):
        return self.author_id

```

####Activity extra data

Often you'll want to store more data than just the basic fields. You achieve this by implementing the extra_activity_data method in the model.

NOTE: you should only return data that json.dumps can handle (datetime instances are supported too).

```
class Tweet(models.Model, Activity):

    @property
    def extra_activity_data(self):
        return {'is_retweet': self.is_retweet }

```


###Feed manager

Django Stream comes with a feed_manager class that helps with all common feed operations.  

####Feeds bundled with feed_manager

To get you started the manager has 4 feeds pre configured. You can add more feeds if your application needs it.
The three feeds are divided in three categories.

#####User feed:
The user feed stores all activities for a user. Think of it as your personal Facebook page. You can easily get this feed from the manager.  
```
feed_manager.get_user_feed(user_id)
```  
#####News feeds:
The news feeds store the activities from the people you follow. 
There is both a flat newsfeed (similar to twitter) and an aggregated newsfeed (like facebook).

```
flat_feed = feed_manager.get_news_feed(user_id)['flat'] 
aggregated_feed = feed_manager.get_news_feed(user_id)['aggregated'] 

```
#####Notification feed:
The notification feed can be used to build notification functionality. 

<p align="center">
  <img src="http://feedly.readthedocs.org/en/latest/_images/fb_notification_system.png" alt="Notification feed" title="Notification feed"/>
  
Below we show an example of how you can read the notification feed.
```
notification_feed = feed_manager.get_notification_feed(user_id)

```
By default the notification feed will be empty. You can specify which users to notify when your model gets created. In the case of a retweet you probably want to notify the user of the parent tweet.

```
class Tweet(models.Model, Activity):

    @property
    def notify(self):
        if self.is_retweet and self.parent_tweet.author != self.author:
            return [self.parent_tweet.author_id]

```

Another example would be following a user. You would commonly want to notify the user which is being followed.

```
class Follow(models.Model, Activity):

    @property
    def notify(self):
        return [self.target_user.id]

```


####Follow a feed
The create the newsfeeds you need to notify the system about follow relationships. The manager comes with APIs to let a user's news feeds follow another user's feed. This code lets the current user's flat and aggregated feeds follow the target_user's personal feed.

```
feed_manager.follow_user(request.user.id, target_user)

```

### Showing the newsfeed

####Activity enrichment

When you read data from feeds, a like activity will look like this:

```
{'actor': 'core.User:1', 'verb': 'like', 'object': 'core.Like:42'}
```

This is far from ready for usage in your template. We call the process of loading the references from the database enrichment. An example is shown below:

```
from stream_django.enrich import Enrich

enricher = Enrich()
feed = feed_manager.get_feed('flat', request.user.id)
activities = feed.get(limit=25)['results']
enriched_activities = enricher.enrich_activities(activities)
``` 



####Templating

Now that you've enriched the activities you can render the template.
For convenience we include the render activity template tag:

```
{% load stream_django %}

{% for activity in activities %}
    {% render_activity activity %}
{% endfor %}

```

The render_activity template tag will render the template activity/[aggregated]/[verb].html with the activity as context.

For example activity/tweet.html will be used to render an normal activity with verb tweet

```
{{ activity.actor.username }} said "{{ activity.object.body }} {{ activity.created_at|timesince }} ago"
```

and activity/aggregated/like.html for an aggregated activity with verb like

```
{{ aggregated.actor_count }} user{{ aggregated.actor_count|pluralize }} liked {% render_activity aggregated.activities.0 %}
```

If you need to support different kind of templates for the same activity, you case send a third parameter to change the template selection.  

The example below will use the template activity/[aggregated]/homepage_%(verb)s.html
```
{% render_activity activity 'homepage' %}
```


###Settings

**STREAM_API_KEY**
Your stream site api key. Default ```''```

**STREAM_API_SECRET**
Your stream site api key secret. Default ```''```

**STREAM_FEED_MANAGER_CLASS**
The path to the feed manager class. Default ```'stream_django.managers.FeedManager'```

**STREAM_USER_FEED**
The name of the feed (as it is configured in your GetStream.io Dasboard) where activities are stored. Default ```'user'```

**STREAM_USER_FEEDS**
The name of the news feed (as they are configured in your GetStream.io Dasboard) where activities from followed feeds are stored. Default ```{'flat':'flat', 'aggregated':'aggregated'}```

**STREAM_PERSONAL_FEED**
The name of the feed (as it is configured in your GetStream.io Dasboard) where activity notifications are stored. Default ```'notification'```

**STREAM_DISABLE_MODEL_TRACKING**
Disable automatic tracking of Activity models. Default ```False```

###Temporarily disabling the signals

When running data imports or other type of one of operations you might want to disable publishing the items to the user's feeds. You can do this via the ```STREAM_DISABLE_MODEL_TRACKING``` django setting.


###Customizing enrichment

Sometimes you'll want to customize how enrichment works. The documentation will show you several common options.

####Enrich extra fields

If you store references to model instances in the activity extra_data you can use the Enrich class to take care of it for you

```
from stream_django.activity import create_model_reference

class Tweet(models.Model, Activity):

    @property
    def extra_activity_data(self):
        ref = create_model_reference(self.parent_tweet)
        return {'parent_tweet': self.parent_tweet }


# instruct the enricher to enrich actor, object and parent_tweet fields
enricher = Enrich(fields=['actor', 'object', 'parent_tweet'])
feed = feed_manager.get_feed('flat', request.user.id)
activities = feed.get(limit=25)['results']
enriched_activities = enricher.enrich_activities(activities)
```

####Change how models are retrieved

The enrich class that comes with the packages tries to minimise the amount of database queries. The models are grouped by their class and then retrieved with a pk__in query. You can implement a different approach to retrieve the instances of a model subclassing the ```stream_django.enrich.Enrich``` class.

To change the retrival for every model you should override the ```fetch_model_instances``` method; in alternative you can change how certain models' are retrieved by implementing the hook function ```fetch_<model_name>_instances```

```
class MyEnrich(Enrich):
    '''
    Overwrites how model instances are fetched from the database
    '''

    def fetch_model_instances(self, modelClass, pks):
        '''
        returns a dict {id:modelInstance} with instances of model modelClass
        and pk in pks
        '''
        ...

class AnotherEnrich(Enrich):
    '''
    Overwrites how Likes instances are fetched from the database
    '''

    def fetch_like_instances(self, pks):
        return {l.id: l for l in Like.objects.cached_likes(ids)}

```


####Prefetch related data

You will commonly access related objects such as activity['object'].user. To prevent your newsfeed to run N queries you can instruct the manager to preload related objects. The manager will use Django's select_related functionality. (https://docs.djangoproject.com/en/dev/ref/models/querysets/#select-related).

```
class Tweet(models.Model, Activity):

    @classmethod
    def related_models(cls):
        return ['user']

```

###Low level APIs access
When needed you can also use the low level Python API directly.
The full explanation can be found in the [getstream.io documentation](https://getstream.io/docs/).


```
from stream_django.client import stream_client

special_feed = stream_client.feed('special:42')
special_feed.follow('flat:60')

```