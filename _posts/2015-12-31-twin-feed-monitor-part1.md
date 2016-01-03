---
layout: post
title: The Twin Feed Monitor (Part 1 of 2)
comments: true
---

In order to ease into ionic development, I came up with an idea for a display that shows the time elapsed since my twin boys last ate. My wife and I both have the [Baby Connect](https://www.baby-connect.com/) app which we use to track feedings (and sometimes poopy diapers.)

Whenever one of the boys gets fussy, the first question asked is usually, "when did he eat last?" This is exacerbated when family is over helping to hold and feed babies. We have to reach for our phone, open the app, navigate to the appropriate child, and check the time of the last feeding. I know, first world problem.

Having done this enough times to go into problem solving mode, I thought it would be cool to have a display in the room which continously showed the last feeding times of each boy. Then, whenever someone feels inclined to ask said question, instead they could just look to the **baby dashboard**!

### How is Babby *Connect* Formed?

As you might have guessed from the ([~~clever~~](https://www.youtube.com/watch?v=Ll-lia-FEIY)) heading, I first wanted to see if I could reverse engineer the app and get the latest feeding times.

My first idea was to have the ionic app scrape the website. This would probably be pretty messy, so I decided to instead write a proxy service to do the [scraping](https://github.com/lapwinglabs/x-ray) instead. On second thought, I decided the mobile app must be calling into a service that I should be able to call directly.

Since I can't use [fiddler](http://www.telerik.com/fiddler) on OSX, I had to find a replacement. I discovered a fantastic little "interactive, SSL-capable man-in-the-middle proxy for HTTP with a console interface," called [mitmproxy](https://mitmproxy.org/). On my phone, I just needed to enable the proxy settings to point at my laptop, install a CA Certificate, and now I can sniff the encrypted network traffic as I interact with the phone app. [Here](http://jasdev.me/intercepting-ios-traffic/) is the tutorial I used.

After some poking, I realized the app only synchronizes updates from other users (e.g. my wife adding a new feed time.) I was able to get a complete data dump after deleting/reinstalling the app, but that was a non-starter. So I had to figure out how to get every update including mine, not just those added by my wife.

The solution? Create a new user!

I registered a "bot" account, and gave it caregiver rights on my account. Now, I can create a service that polls for updates from both me and my wife.

### I Feel Hapi

Now that I have the service request figured out, I could just called it from the ionic app. But their service requires things like authentication, and I'd have to track the last synchronized entries, so I still wanted to create a proxy that was simple to consume by my app. Another benefit is being able to patch the proxy service without having to deploy a new app. Separation of concerns and all that.

The easiest way I know to spin up a simple rest service is to use [hapi.js](http://hapijs.com/). So let's give it a shot.

#### Creating the Service

You can find the server code [here](https://github.com/josephgardner/feed-timer/tree/master/src/proxy). Each kid is given a unique ID which you can set in the `config.json`. The service I'm calling is used for synchronization, so for each kid, you essentially need to ACK the ID of the last entry you received. I accomplished this by seeding a `/persist/kid-map.json` lookup file with a `last` property. Then you zip the 2 values together in the query string like so:

`&kids=<kid1-id>,<kid1-last>,<kid2>,<kid2-last>`

The response body includes any new entries for each kid since the specified `last` entry. I update the `kid-map.json` with these values to be used in the next call. I also store the feeding data.

My service then just dumps the `kid-map.json` service which now has the last feeding entry for each kid. This is much easier to consume than having to deal with auth and the sync logic. Every time you call my service, you are guaranteed to just get the last feeding entry for each kid.

```json
{
  "<kid1-id>": {
    "last": "<kid1-last>",
    "item": {
      ...
      "e": "1/3/2016 12:07",
      "Txt": "Kid1 drank 4 oz of milk",
    }
  },
  "<kid2-id>": {
    "last": "<kid2-last>",
    "item": {
      ...
      "e": "1/3/2016 12:37",
      "Txt": "Kid2 drank 2 oz of formula",
    }
  }
 }
```

Stay tuned for Part 2 where I create the ionic app to display the dashboard.
