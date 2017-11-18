# Resource Discovery with Activity Streams

Using standard, semantic technologies for easy optimization of resource discovery, crawling and indexing.

## The Approach

Following the [design principles](http://iiif.io/api/annex/notes/design_patterns/) of IIIF:
 * As Simple as Possible, and No Simpler
 * Intelligently Manage Ramping Up
 * Avoid Specific Technology implementations
 * Follow Resource-Oriented Design
 * Follow Linked Data
 * Design for JSON-LD First
 * Use Standards
 * Follow Best Practices
 * Loosely couple APIs
 * Define Success, not Failure

And the useful patterns established by the [ResourceSync](http://openarchives.org/rs/1.1/resourcesync) effort:
  * List of Resources
  * List of Changes
  * Notification of Changes

As scoped by the [IIIF Discovery charter](http://iiif.io/community/groups/discovery/charter/) and [use cases](https://github.com/iiif/iiif-stories/issues?utf8=%E2%9C%93&q=is%3Aissue%20is%3Aopen%20label%3Adiscovery), we can then design for solving the challenges faced by the community.

## Activity Streams

[Activity Streams](https://www.w3.org/TR/activitystreams-core/) (AS2) is a "model for representing potential and completed activities". IIIF already uses the AS2 Collections and Paging model, as does the [Web Annotation](https://www.w3.org/TR/annotation-model/#collections) work that we depend on.  AS2 is designed to be JSON-LD first while still respecting the Linked Open Data paradigm, and follows the best practices defined in the W3C.

### List of Resources

As IIIF uses the AS2 Collection model, a IIIF Collection is very close to an AS2 Collection of Resources already!  AS2 Collections for lists of this size will always be paged, however, and the contents in the pages.  Thus for all of the options that follow, the top resource will be a single, paged Collection document.

The Collection does not directly contain any of the activities or resources, instead it refers to the `first` page of such a list.  The pages are ordered both from page to page by following `next` relationships, and internally within the page as the `items` list is also ordered.  The number of entries in each page is up to the implementer, and cannot be requested by the client.

* [Collection](/json/collection.json)

#### Too Simple

The simplest possible page layout would just be a list of Manifests, identical in structure to a IIIF Collection.  If we were going to recommend this, we would just use IIIF Collections directly.  It does not ramp up to a full activitystream well, but this level demonstrates the similarity of the AS2 approach with existing IIIF patterns.

Usage:  Crawl the entire set of pages and dereference each manifest to see if it has changed.

* [Page: Too Simple](/json/0_page-toosimple.json)


#### Level 0: Basic Resource List

However with the addition of just a little boilerplate in the JSON, we can be on the path towards a robust set of changes.  We add an "Update" activity wrapper around the Manifest, and do not set a date for when it was created.  The order of the Manifests in the pages is unimportant, but each should only appear once. In terms of optimization, it provides no additional benefit over the IIIF Collection pattern, but is compatible with a system where optimization is possible, such as the following levels.  This is thus the minimum level for interoperability, but further levels are significant improvements in terms of efficiency.

Usage:  Crawl the entire set of pages and dereference each manifest to see if it has changed.

* [Page: Basic Resource List](/json/1_page-resourcelist.json)


#### Level 1: Basic Change List

If we know dates, such as when the resource was created or updated, we should instead order the list by those dates, with the most recent activities occuring last.  Each Manifest 
should still appear only once in the full list.  This could be accomplished with a IIIF Collection, if created and modified timestamps were part of the Manifests themselves.

The rationale for crawling backwards is that the first pages, once finished, become static resources.  If the list were ordered from most recent to least, then either the pages would change their URIs, reducing the ability to cache them or the first page would be constantly changing size rather than the last page, which is more familiar.

Usage: Record each time the list is crawled. Start from the `last` page and work backwards through the list until a datestamp before the previous time a crawl occurred is encountered.

* [Page: Basic Change List](/json/2_page-basicchangelist.json)

#### Level 1b: Change List with Metadata

Additional metadata can be added to the basic change list without affecting the overall crawl behavior.  This metadata can include who did the change, more information about when the change occured as opposed to when the change activity was published, and so forth.  This level (and subsequent) cannot be achieved using just IIIF Collections without significant extensions.

Usage: Record each time the list is crawled. Start from the `last` page and work backwards through the list until a datestamp before the previous time a crawl occurred is encountered.

* [Page: Change List with Extra Information](/json/3_page-changelistmetadata.json)

#### Level 2: Complete Change List

At the most complex level, a log of all of the activities that have taken place can be recorded, with multiple events per resource.  This would include Deletes, allowing a synchronization process to remove resources as well as add them. The list might thus end up very long if there are many changes to resources, however this is typically infrequent in known IIIF community.  This would also allow for the complete history of a resource to be reconstructed, if each version has an archived representation.  This level cannot be achieved using IIIF Collections or the previous levels at all, as the deleted Manifest would have been deleted and not appear in the list.

Usage:  Record each time the list is crawled. Start from the `last` page and work backwards through the list until a datestamp before the previous time a crawl occurred is encountered. Only process the most recent change per resource, including deleting resources.

* [Page: Complete Change List](/json/4_page-fullchangelist.json)


#### Level 3:  External Activities

Once there is a framework for recording and publishing activities, the next level of complexity would be to allow third parties to send activities that use the publisher's resources.  In particular, creating links to the resource or just that the resource was used.  This requires both an inbox and an outbox -- the inbox allows requests to be delivered to the publisher, and the outbox is for the publisher's own events as above.  This is discussed below in the Notifications of Change section.

#### Level 4: Distributed System Synchronization

Finally, and far beyond the requirements for IIIF Discovery, the same patterns scale to a full distributed transaction log system, along [these lines](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying).  This degree of scalability is encouraging that the approach is the correct one.


## Notifications of Change

[Linked Data Notifications](https://www.w3.org/TR/ldn/) allow for these notifications to be sent to and made available from an "inbox", and thus we would be consistent with section 3 of the charter around Notification.  Further, the [ActivityPub](https://www.w3.org/TR/activitypub/) specification introduces the additional notion of an "outbox".

Sending Manifests to an Inbox is not a "notification", it's just moving data around. Instead we send a description of the activity, which the reciever can process or not as it chooses.
 
LDN provides a storage and publication mechanism, but clients are still expected to come and retrieve the set of changes in the same way as above.  Assuming that either (a) systems register their interest in a resource, and the publisher notifies them directly or (b) the publisher writes to the inbox, and the inbox then auto-forwards the notification to the subscribers ... then there needs to be a method of subscribing to have the changes pushed to a remote inbox.

### Linked Data Notifications: Subscription

While there is [WebSub](https://www.w3.org/TR/websub/), this does not interoperate with LDN and is from a service oriented mindset. WebSub makes various requirements that are not appropriate for Notifications (you MUST send the entire contents of the resource that you have subscribed to) which prevents subscribing either to binary resources (you don't want the entire gigabyte TIFF) or or aggregation-level containers (you don't want every change, every time, just the new one).

ActivityPub is a method for transferring the notifications and subscribing, but does not follow REST patterns. Instead it uses side effects associated with particular activities. For example, the side effect of recieving a "Follow" activity is to add the sender to the "followers" of the resource. A REST based approach would be to create the subscription resource within the followers container.

A more LOD / LDP / LDN appropriate pattern would be to use a REST pattern with Containers, that allow subscription to the inbox, manage filtering and the callback endpoint.  One way to do this could be as [described here](https://docs.google.com/document/d/1JsQS1LVFt8wuJSYo_XsOzyP28pS8hfSLEhAC3BFuN6o/edit).

For the purposes of IIIF discovery, the above decision is firmly in the Notification section of the charter and does not impact the publication of the Activitystreams Collections as resources.

---

## Types of Change

### Single-Resource Publisher Operations

#### Create

Level: 0

The object resource came into existence. Before this time it did not exist, and the state of the resource is the first state.

```json
{
	"id": "http://data.getty.edu/iiif/discovery/event/54627",
	"type": "Create",
	"object": {
		"id": "https://data.getty.edu/iiif/museum/1/manifest.json",
		"type": "Manifest"
	},
	"startTime": "2017-06-08T00:00:00Z"
}
```


#### Update / Modify

Level: 1

A resource that was already created was changed.  There is a new state of the resource that replaces the previous state.  The previous state may be available at another URI.

```json
{
	"id": "http://data.getty.edu/iiif/discovery/event/100001",
	"type": "Update",
	"object": {
		"id": "https://data.getty.edu/iiif/museum/1/manifest.json",
		"type": "Manifest"
	},
	"startTime": "2017-09-19T20:00:00Z"
}
```

#### Delete

Level: 2

A resource that was already created was taken out of existence. After this time, the resource does not exist, and previous states may be available at other URIs.

```json
{
	"id": "http://data.getty.edu/iiif/discovery/event/8172645",
	"type": "Delete",
	"object": {
		"id": "https://data.getty.edu/iiif/museum/1/manifest.json",
		"type": "Manifest"
	},
	"startTime": "2017-09-19T20:00:00Z"
}
```


### Multi-Resource Publisher Operations

These operations are compositions of the above in terms of the effects on different resources, but should be treated as a single operation for the purpose of notification and replay.  These are more appropriate for true synchronization systems, rather than simple crawling and discovery.

#### Add 

Level: 4

The source resource was added to the target Collection.

```json
{
	"id": "http://data.getty.edu/iiif/discovery/event/123462",
	"type": "Add",
	"object": {
		"id": "https://data.getty.edu/iiif/museum/1/manifest.json",
		"type": "Manifest"
	},
	"target": {
		"id": "https://data.getty.edu/iiif/museum/paintings/collection.json",
		"type": "Collection"
	},
	"startTime": "2017-09-19T20:00:00Z"
}
```

#### Remove

Level: 4

The source resource was removed from the target Collection.

```json
{
	"id": "http://data.getty.edu/iiif/discovery/event/123462",
	"type": "Remove",
	"object": {
		"id": "https://data.getty.edu/iiif/museum/1/manifest.json",
		"type": "Manifest"
	},
	"origin": {
		"id": "https://data.getty.edu/iiif/museum/paintings/collection.json",
		"type": "Collection"
	},
	"startTime": "2017-09-19T20:00:00Z"
}
```


#### Copy

Minimum Level: 4

The source resource was duplicated to create or modify the target resource.  This activity is not a core ActivityStreams type.

```json
{
	"id": "http://data.getty.edu/iiif/discovery/event/123462",
	"type": "Copy",
	"object": {
		"id": "https://data.getty.edu/iiif/museum/1/manifest.json",
		"type": "Manifest"
	},
	"target": {
		"id": "https://data.getty.edu/iiif/publications/102314/manifest.json",
		"type": "Manifest"
	},
	"startTime": "2017-09-19T20:00:00Z"
}
```

#### Rename / Move

Minimum Level: 4

The source resource is duplicated to create or modify the target resource, and the original was deleted as part of the operation.

```json
{
	"id": "http://data.getty.edu/iiif/discovery/event/123462",
	"type": "Move",
	"object": {
		"id": "https://data.getty.edu/iiif/museum/1/manifest.json",
		"type": "Manifest"
	},
	"target": {
		"id": "https://data.getty.edu/iiif/publications/102314/manifest.json",
		"type": "Manifest"
	},
	"startTime": "2017-09-19T20:00:00Z"
}
```

#### Merge

Minumum Level: 4

Two or more source resources were combined to create or modify the target resource.  This activity is not a core ActivityStreams type.

```json
{
	"id": "http://data.getty.edu/iiif/discovery/event/123462",
	"type": "Merge",
	"object": [
		{
			"id": "https://data.getty.edu/iiif/museum/1/manifest.json",
			"type": "Manifest"
		},
		{
			"id": "https://data.getty.edu/iiif/museum/2/manifest.json",
			"type": "Manifest"
		}
	],
	"target": {
		"id": "https://data.getty.edu/iiif/museum/1726/manifest.json",
		"type": "Manifest"
	},
	"startTime": "2017-09-19T20:00:00Z"
}
```

#### Split

Minumum Level: 4

A single source resource was divided to create or modify multiple target resources.

```json
{
	"id": "http://data.getty.edu/iiif/discovery/event/123462",
	"type": "Split",
	"object": {
		"id": "https://data.getty.edu/iiif/museum/1726/manifest.json",
		"type": "Manifest"
	},
	"target": [
		{
			"id": "https://data.getty.edu/iiif/museum/1/manifest.json",
			"type": "Manifest"
		},
		{
			"id": "https://data.getty.edu/iiif/museum/2/manifest.json",
			"type": "Manifest"
		}
	],
	"startTime": "2017-09-19T20:00:00Z"
}
```

### Third Party Operations

These operations are carried out by third parties, rather than the publisher.  As such they would be written to the Inbox of the resource as either a request to modify the resource, or just notification that the resource was used.


#### Reference

Minimum Level: 3

A remote source resource has a new reference or link to the target local resource.

```json
{
	"id": "http://data.getty.edu/iiif/discovery/event/8172645",
	"type": "Reference",
	"object": {
		"id": "https://example.edu/iiif/9/manifest.json",
		"type": "Manifest"
	},
	"target": {
		"id": "https://data.getty.edu/iiif/museum/1/manifest.json",
		"type": "Manifest"
	},
	"startTime": "2017-09-19T20:00:00Z"
}
```


#### Use

Minimum Level: 3

The target resource was used by the remote agent in some way.

```json
{
	"id": "http://data.getty.edu/iiif/discovery/event/8172645",
	"type": "Use",
	"object": {
		"id": "https://data.getty.edu/iiif/museum/1/manifest.json",
		"type": "Manifest"
	},
	"startTime": "2017-09-19T20:00:00Z"
}
```

#### Replace

Minimum Level: 4

The third party is requesting that the publisher replace their copy of the target resource with the source resource.

```json
{
	"id": "http://data.getty.edu/iiif/discovery/event/8172645",
	"type": "Replace",
	"object": {
		"id": "https://example.edu/iiif/9/manifest.json",
		"type": "Manifest"
	},
	"target": {
		"id": "https://data.getty.edu/iiif/museum/1/manifest.json",
		"type": "Manifest"
	},
	"startTime": "2017-09-19T20:00:00Z"
}
```





