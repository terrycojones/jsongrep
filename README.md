# jsongrep

Python for extracting pieces of JSON objects

Lots of APIs these days return JSON objects. I love JSON, but reading a raw
JSON dump can be awkward. You can print the whole thing using pprint in
Python (or your favorite language), but what if you want to grep out and
print only parts of the JSON? I was thinking tonight that it would be easy
and useful to write a recursive script to do that. It took about half an hour
to arrive at this solution:

    #!/usr/bin/env python

    import sys
    import re
    import json
    from pprint import pprint

    def jsongrep(d, patterns):
        try:
            pattern = patterns.pop(0)
        except IndexError:
            pprint(d)
        else:
            if isinstance(d, dict):
                keys = filter(pattern.match, d.keys())
            elif isinstance(d, list):
                keys = map(int,
                           filter(pattern.match,
                                  ['%d' % i for i in range(len(d))]))
            else:
                if pattern.match(str(d)):
                    pprint(d)
                return
            for item in (d[key] for key in keys):
                jsongrep(item, patterns[:])

    if __name__ == '__main__':
        try:
            j = json.loads(sys.stdin.read())
        except ValueError, e:
            print >>sys.stderr, 'Could not load JSON object from stdin.'
            sys.exit(1)

        jsongrep(j, map(re.compile, sys.argv[1:]))

Usage is really simple. Let's look at a couple of easy examples from the command line:

    $ echo '{"hey" : "you"}' | jsongrep.py hey
    u'you'

`jsongrep.py` has matched the "hey" key in the JSON object and printed its
value. Let's do the same thing with a 2-level JSON object:

    $ echo '{"hey" : { "there" : "you"}}' | jsongrep.py hey
    {u'there': u'you'}

Again, we see the entire object corresponding to the "hey" key. We can add
another argument to drill down into the object

    $ echo '{"hey" : { "there" : "you"}}' | jsongrep.py hey there
    u'you'

As you might hope, you can use a regular expression for an argument:

    $ echo '{"hey" : { "there" : "you"}}' | jsongrep.py 'h.*' '.*'
    u'you'

which in this case could have been given more concisely as

    $ echo '{"hey" : { "there" : "you"}}' | jsongrep.py h .
    u'you'

So you can drill down into nested dictionaries quite easily. When
`jsongrep.py` runs out of patterns it just prints whatever's left. A special
case of this is if you give no patterns at all, you get the whole JSON
object:

    $ echo '{"hey" : { "there" : "you"}}' | jsongrep.py
    {u'hey': {u'there': u'you'}}

The regex patterns you pass on the command line are being matched against
the keys of JSON objects (Python dicts). If `jsongrep.py` runs into a list,
it will instead match against the list indices like so:

    $ echo '{"hey" : { "joe" : ["blow", "xxx" ]}}' | jsongrep.py hey joe 1
    u'xxx'

You can see we've pulled out just the first list element after matching
"hey" and "joe". So `jsongrep.py` regex args can be used to navigate your way
through both JSON objects and lists.

Now let's do something more interesting.

Twitter's API can give you JSON, and the JSON is pretty chunky. For
example, if I get my first 100 followers with this command:

    $ curl 'http://api.twitter.com/1/statuses/followers.json?screen_name=terrycojones'

there's 164Kb of output (try it and see). What if I just want the Twitter
user names of the people who follow me? Looking at the JSON, I can see it
starts with:

    [{"profile_background_color":"131516","description":null

Hmm… looks like it's a list of dictionaries. Let's print just the first dictionary in the list:

    $ curl 'http://api.twitter.com/1/statuses/followers.json?screen_name=terrycojones' |
    jsongrep.py 0

which starts out:

    {u'contributors_enabled': False,
     u'created_at': u'Wed Jul 19 00:29:58 +0000 2006',
     u'description': None,
     u'favourites_count': 0,
     u'follow_request_sent': False,
     u'followers_count': 178,
     u'following': False,
     u'friends_count': 67,
     u'geo_enabled': False,
     u'id': 2471,
     u'id_str': u'2471',
     u'lang': u'en',
     u'listed_count': 3,
     u'location': None,
     u'name': u'Roy',
     u'notifications': False,
     u'profile_background_color': u'131516',
     u'profile_background_image_url': u'http://s.twimg.com/a/1288470193/images/themes/theme14/bg.gif',
     u'profile_background_tile': True,
     u'profile_image_url': u'http://a3.twimg.com/profile_images/194788727/roy_with_phone_normal.jpg',
     u'profile_link_color': u'009999',
     u'profile_sidebar_border_color': u'eeeeee',
     u'profile_sidebar_fill_color': u'efefef',
     u'profile_text_color': u'333333',
     u'profile_use_background_image': True,
     u'protected': False,
     u'screen_name': u'wasroykosuge',

and you can see a "screen_name" key in there which looks like what we want. Let's see the first few:

    $ curl 'http://api.twitter.com/1/statuses/followers.json?screen_name=terrycojones' |
    jsongrep.py . screen_name | head
    u'wasroykosuge'
    u'Piiiu_piiiu'
    u'350'
    u'KardioFit'
    u'jrecursive'
    u'doodlesockingad'
    u'revinprogress'
    u'cloudjobs'
    u'PointGcomics'
    u'lucasbuchala'

Finally, here's an example using FluidDB‘s new /values HTTP call. I'll ask FluidDB for all 
objects matching the query has `unionsquareventures.com/portfolio` and from those matching
objects I'll pull back the value of the FluidDB tag named `fluiddb/about`.
The result is JSON that starts out like this:

    {"results": { "id" : {"93989942-b519-49b4-87de-ac834e6a6161": {"fluiddb/about": {"value": "http://www.outside.in"}}

You can see there's a 5-level deep nesting of JSON objects. I just want the "value" key on all matching objects. Easy:

    $ curl 'http://fluiddb.fluidinfo.com/values?query=has%20unionsquareventures.com/portfolio&tag=fluiddb/about' |
    jsongrep.py results . . fluiddb/about value | sort
    u'http://amee.cc'
    u'http://getglue.com'
    u'http://stackoverflow.com'
    u'http://tumblr.com'
    u'http://www.10gen.com'
    u'http://www.boxee.tv'
    u'http://www.buglabs.net'
    u'http://www.clickable.com'
    u'http://www.cv.im'
    u'http://www.disqus.com'
    u'http://www.etsy.com'
    u'http://www.flurry.com'
    u'http://www.foursquare.com'
    u'http://www.heyzap.com'
    u'http://www.indeed.com'
    u'http://www.meetup.com'
    u'http://www.oddcast.com'
    u'http://www.outside.in'
    u'http://www.returnpath.net'
    u'http://www.shapeways.com'
    u'http://www.simulmedia.com'
    u'http://www.targetspot.com'
    u'http://www.tastylabs.com'
    u'http://www.tracked.com'
    u'http://www.twilio.com'
    u'http://www.twitter.com'
    u'http://www.workmarket.com'
    u'http://www.zemanta.com'
    u'http://zynga.com'

And there you have it, a sorted list of all Union Square Ventures portfolio companies,
from the command line, as listed here.

`jsongrep.py` will also try to match on things that are not objects or lists if it
runs into them, so we can refine this list a little. E.g.,

    $ curl 'http://fluiddb.fluidinfo.com/values?query=has%20unionsquareventures.com/portfolio&tag=fluiddb/about' |
    jsongrep.py results . . fluiddb/about value '.*ee' | sort
    u'http://www.meetup.com'
    u'http://amee.cc'
    u'http://www.indeed.com'
    u'http://www.boxee.tv'

being the USV companies with "ee" somewhere in their URL. Or, for some advanced regex fun,
the USV companies whose URLs don't end in ".com":

    $ curl 'http://fluiddb.fluidinfo.com/values?query=has%20unionsquareventures.com/portfolio&tag=fluiddb/about' |
    jsongrep.py results . . fluiddb/about value '.*(?<!\.com)$'
    u'http://www.outside.in'
    u'http://amee.cc'
    u'http://www.cv.im'
    u'http://www.buglabs.net'
    u'http://www.returnpath.net'
    u'http://www.boxee.tv'
