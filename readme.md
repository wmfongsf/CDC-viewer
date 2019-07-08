# SFDC Simple CDC + Platform Event Viewer
### Things this is: Open sourced, free
### Things this isn't: Didn't do extensive disconnect logic, retries, that sort of thing, so it's a baseline to show stuff, but it's not complex. Also probably isn't efficient to go JSFOrce-Faye-Socket, but it was suuuuper easy to set up quickly.
### Status: Actually it works just fine. Maybe a 5-10m spinup process for localhost. Can push to Heroku too if ya wanted, though I haven't really written it for multiple users/multiple orgs concurrently tbh.

## Ramblin's

Basically what you have here is a quick node app that will let you consume from Platform Events and CDC.

For CDC: What I mean is all the CDC. I didn't add the ability to pick your objects, it's simple and just hitting `data/ChangeEvents`, to capture everything and anything.

For Platform Events: There's a form in there that'll let you specify the Platform Event you want to sub to. Make sure you __e that sucker, and use the API name, or it will fail to connect. I even threw a pretty Toast in there for ya. Fun times with SLDS, right?

For EM Realtime Beta: Simply put in the name of the EM log you want to track in the Platform Event tab (such as `UriEventStream`), and it should get picked up! 

Now, I didn't want any of the CDC logic in the client, so we're basically using two different methods to event in real-time. Basically, the dance is such - when you hit index, it'll check the **NForce** object to see if we've got a live org. If not, use **NForce** to quickly redirect to an oAuth page from SF to log in, and authorize the app. Then redirected back to the live feed, newly connected (the redirect URL expected is `/auth/sfdc/callback`, found in *routes/index.js*[13] and routed at *routes/index.js*[35]). At this point, **Faye** will kick off with your token, and set up a subscription to your org's `data/changeEvents` CDC endpoint, which gives you everything.

Now, to get the feed to the clientside, we need to add a server for the client to listen to. I probably coulda created a **Faye** client, but I didn't. Once the **Faye** client is in, we activate a channel in **socket.io** in order to publish to. Once you're redirected to the live index.pug, we activate a client-side **socket.io** subscriber to listen to the messages routed via **Faye** to **socket.io**. 

In theory, say you wanted a page that was serving up realtime updates to multiple users, you'd actually want this sorta decoupled 2-step dance. SFDC limits events on messages x subscribers - so 50 people subbing to 50 events is 50x worse than if you had one 'server' subscribing and then pushing out on its own bus to the 50 users. If you wanted to move to that model, basically you just break this into two separate apps, and point the base.js[7] io() command to your real app. Tests real pretty on localhost, just use different ports and you're all set.

Styling? We just downloaded the SLDS package and are using that to make it look pretty and Salesforcey.

## Setup

Get a salesforce org. Create a connected app. Set the OAuth endpoint to `http://localhost:3000/auth/sfdc/callback`. Wait like 5 minutes. Srs, it's gotta propagate throughout all the servers, really let it breathe.

While you're waiting - go into **Setup** and **Change Data Capture**. Activate this for the objects you want to play with. It's dirt simple, seriously. If you want Platform Events, search that in **Setup** and create your event definitions. I did a quick search and didn't see a super easy way to just query for all existing custom events to throw you a bone with a picklist, so just write down yer API name for the event (should end in __e).

### Local Install/Execution:

`git clone this`
`npm install`
`touch .env`

Env Vars Of Note to put either in your local .env file, or in Heroku's dashboard.
* CLIENTID / CLIENTSECRET - Standard OAuth things
* ENVIRONMENT - 'sandbox' or 'production' - hint: if a scratch org, dats a sandbox
* PORT - I like 3000, if on Heroku this'll get auto-set for you.

The format is literally VARNAME=VARVALUE. No other shenanigans needed in the .env file.

`heroku local` - use this, NOT npm start, in order to pick up all them pretty environment variable bits in the .env var. What's that? no Heroku CLI? Come on (wo)man. Get your priorities in order.

At that point you can go to `http://localhost:PORT` and you'll get redirected to log in. Log in, authorize your app, and you'll be dumped in the index page. At this point you should be able to make changes to the CDC-tracked objects and see all data changes come through. It handles multiple-record changes, and will identify each object type for you as well. 

For Platform Events, hit the Platform Events tab, then pipe in your custom event name. That's it - now it should consume anything that hits that event topic.

### Heroku Install/Execution
Supes lazy, will get to this - basically just create an app, push the source, create the env vars, and make sure your Connected App is pointing to your heroku endpoint not localhost and I think you should be good to go. I'm not gonna lie - I don't think this will effectively work with more than one org/user at a time, so don't go too public with it, ya know? 

## What would I need to do to make this real?
I mean really, it *is*, but it is just a quick visualizer, right? You probably need to add retry logic and other elements, but since this is oriented around a client view of the messages flowing through, I imagine it'd be tying the message logic into whatever js you're doing, instead of just creating dom elements. Trigger things on the page, etc. Really float yer boat. That said - be careful here, it's not really...built for like 80 viewers of the same thread. You might want to break this apart if you're building a streaming system against CDC, basically one Heroku App to be what Faye does here and set up a socket.io bit, and then everything hitting that. You can't trust that everyone'll be on the same dyno, so you need to build this right, dig?

You could probably not use socket if you didn't want to, or swap out JSForce instead of NForce + Faye. That probably isn't the best way, but again - lazy, ya know?

## What can I learn from this sucker?
Ehh probably not too much, I mean it is what it is.

## ToDo / Asks
* Probably should tab out or something to allow for easier segmentation of different objects/events.
* Didn't do a TON of Gap/Overflow testing, nor a ton of testing on the small disconnect logic that's in there. 