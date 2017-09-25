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

As IIIF uses the AS2 Collection model, a IIIF Collection is very close to an AS2 Collection of Resources already!  AS2 Collections for lists of this size will always be paged, however, and the contents in the pages.  Thus for any of the options that follow, the top resource will be a single Collection document.

* [Collection](/json/collection.json)

#### Too Simple

The simplest possible page layout would be a list of Manifests, identical in structure to a IIIF Collection.  If we were going to recommend this, we should just use IIIF Collections directly.  It does not ramp up to a full activitystream well, but this level demonstrates the similarity of the AS2 approach with existing IIIF patterns.

Usage:  Crawl the entire set of pages and dereference each manifest to see if it has changed.

* [Page: Too Simple](/json/0_page-toosimple.json)

#### Level 0: Basic Resource List

However with the addition of just a little boilerplate in the JSON, we can be on the path towards a robust set of changes.  We add a "Create" activity wrapper around the Manifest, and do not set a date for when it was created.  The order of the manifests in the pages is unimportant, but each should only appear once (as it was only created once). In terms of optimization, it provides no additional benefit over the IIIF Collection pattern, but is compatible with a system where optimization is possible.

Usage:  Crawl the entire set of pages and dereference each manifest to see if it has changed.

* [Page: Basic Resource List](/json/1_page-resourcelist.json)


#### Level 1: Basic Change List

If we know dates, such as when the resource was created or modified, instead we can order the list by those dates, with the most recent activities occuring first.  Each Manifest 
should still appear only once in the full list.

Usage: Record each time the list is crawled, and stop crawling when a datestamp before the previous crawl time is encountered.

* [Page: Basic Change List](/json/2_page-basicchangelist.json)

#### Level 1b: Change List with Metadata

Additional metadata can be added to the basic change list without affecting the overall crawl behavior.  This metadata can include who did the change, more information about when the change occured as opposed to when the change activity was published, and so forth.

Usage:  Record each time the list is crawled, and stop crawling when a datestamp before the previous crawl time is encountered.

* [Page: Basic Change List](/json/3_page-changelistmetadata.json)

#### Level 2: Complete Change List


## LDN

[Linked Data Notifications](https://www.w3.org/TR/ldn/)


## Linked Data Notifications: Subscription

While there is [WebSub](https://www.w3.org/TR/websub/), this does not interoperate with LDN and is from a service oriented mindset. WebSub makes various requirements that are not appropriate for Notifications (you MUST send the entire contents of the resource that you have subscribed to) which prevents subscribing either to binary resources (you don't want the entire gigabyte TIFF) or or aggregation-level containers (you don't want every change, just the new one).


