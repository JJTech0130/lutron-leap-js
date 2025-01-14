# lutron-leap

This library is an implementation of Lutron's unpublished LEAP protocol. It is, in large part, a port of [pylutron-caseta](https://github.com/gurumitts/pylutron-caseta), without which this would not have been possible. It was written to support the [homebridge-lutron-caseta-leap](https://github.com/thenewwazoo/homebridge-lutron-caseta-leap) Homebridge plugin, but it exists independently of it and has been tested and improved in non-Caseta contexts.

The code is kinda messy. Sorry about that.

## Code structure

This code has three major components: the message parsing stack, the LEAP client, and a Caséta Smart Bridge 2 abstraction.

### Message parsing

At the top level, [LEAP messages](https://github.com/thenewwazoo/lutron-leap-js/blob/main/src/LeapClient.ts#L15) have three parts:
* A communique type, which indicates the function (subscription, information read, update, etc) and direction (response or request)
* A header, which includes a status code, a client-supplied tag, a URL indicating the resource, and [a body type](https://github.com/thenewwazoo/lutron-leap-js/blob/main/src/MessageBodyTypes.ts#L12)
* A body, of the specified type, containing the data being passed

Most of the code is concerned with parsing responses, as most of the job of this library is passing messages from the Lutron hub to whoever cares to listen.

Updating this code is pretty mechanical at this time, and the best way to understand it is to look at [a commit that adds newly-observed objects](https://github.com/thenewwazoo/lutron-leap-js/commit/0eaeeac217cc0fbe160a466ccdbf97083b607e18). In short, you'll want to:
* name [the new body type](https://github.com/thenewwazoo/lutron-leap-js/blob/0eaeeac217cc0fbe160a466ccdbf97083b607e18/src/MessageBodyTypes.ts#L13) (from the header)
* define the [shape of the body object itself](https://github.com/thenewwazoo/lutron-leap-js/blob/0eaeeac217cc0fbe160a466ccdbf97083b607e18/src/MessageBodyTypes.ts#L24)
* define the structure of [the element(s) that comprise the body](https://github.com/thenewwazoo/lutron-leap-js/blob/0eaeeac217cc0fbe160a466ccdbf97083b607e18/src/MessageBodyTypes.ts#L239)
* add the type to the [set of known parse results](https://github.com/thenewwazoo/lutron-leap-js/blob/0eaeeac217cc0fbe160a466ccdbf97083b607e18/src/MessageBodyTypes.ts#L67)
* add a [case to the parser](https://github.com/thenewwazoo/lutron-leap-js/blob/0eaeeac217cc0fbe160a466ccdbf97083b607e18/src/MessageBodyTypes.ts#L105)
* add a test to parse a from-the-wild message to make sure it all works

### The LEAP client

The LEAP client handles reading and writing from the secure socket, as well as routing messages to subscriber callbacks. Requests are submitted with a user-supplied, arbitrary tag. This tag is returned with relevant responses. For example, if you subscribe to some event and provide tag `1`, event messages will also include the `1` tag. You, dear user, don't actually care about this. All you care about is having your callback called, which the LEAP client does.

The LEAP client also passes messages that are "unsolicited", and do not have a tag. These messages are [emitted by the LEAP client](https://github.com/thenewwazoo/lutron-leap-js/blob/851f87c4941c19acfff86ee39317ca5f365e027c/src/LeapClient.ts#L32). Any listener can receive them and process them.

Messages that have a tag that is _not_ recognized by the client are noisily dropped.

**WARNING**: The socket handling in the LEAP client class is super duper ugly. If you feel like refactoring it, please do.

### Bridge-related abstractions

Because this code was written to support Caseta devices, the following section only applies to the Caseta Smart Bridge 2. I wholeheartedly welcome improvements related to other LEAP technologies.

Caseta Smart Bridge 2 and Smart Bridge 2 Pro devices are [discovered by using mDNS](https://github.com/thenewwazoo/lutron-leap-js/blob/851f87c4941c19acfff86ee39317ca5f365e027c/src/BridgeFinder.ts#L40). The BridgeFinder [handles discovery](https://github.com/thenewwazoo/lutron-leap-js/blob/851f87c4941c19acfff86ee39317ca5f365e027c/src/BridgeFinder.ts#L157) and creates [SmartBridge](https://github.com/thenewwazoo/lutron-leap-js/blob/851f87c4941c19acfff86ee39317ca5f365e027c/src/SmartBridge.ts#L37) objects that contain the actual client itself.

The SmartBridge class abstractions for relevant operations like subscribing to known device types. The goal is to relieve the user of having to know how to construct URLs and post bodies, but instead to encode that information in this library.

## Real-world testing

This library has been extensively tested on Caseta Smart Bridge 2 (pro and non-pro) devices, and gracious contributors have tested it with RA3. If you have used it elsewhere, please drop a note and let me know. So far, this library has been tested with:

* Caseta Smart Bridge 2
* Caseta Smart Bridge 2 Pro
* Pico remote
* Caseta occupancy sensor
* Serena wood blinds
* RA3 processor
* Sunnata Dimmer
* Sunnata switch
* RA3 wall motion sensor
* RA3 ceiling mounted motion sensor
