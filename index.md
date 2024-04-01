# Butler Client/Server Implementation and Deployment Strategies

```{abstract}
There are many approaches that can be taken when considering where to deploy the Butler servers and how to implement a server in a way that can support the performance requirements.
This document proposes some strategies that came out of the recent Joint Technical Meeting held at SLAC in February 2024.
```

## Introduction

Rubin Observatory's Data Butler {cite:p}`2022SPIE12189E..11J` provides an abstraction layer between the scientists and the data holdings, allowing scientists to form queries based on concepts such as instrument, filter, and observing day and retrieve data without knowing where the data was stored or what file format it used.
The Data Butler delivered from the Rubin construction project in 2022 {cite:p}`DMTR-271` used a direct connection from the client to the PostgreSQL registry database and a direct connection to the file system or object store (without per-user access controls) hosting the data files.

With the US Data Facility switching from NCSA to SLAC and it being understood that SLAC will not be willing to issue user accounts to 10,000 astronomers (and some astronomers may not be eligible for accounts) it became clear that direct connections to databases were not going to be possible.
The solution was to modify the Butler software to have a web service that runs inside the Rubin Science Platform {cite:p}`LDM-542` and which can look at the authorization tokens {cite:p}`DMTN-182` and decide what the user can see in the Butler repository wherever the database and data reside.

A client/server butler was prototyped in {cite:p}`DMTN-242` and a design meeting was held in Princeton in 2023 October {cite:p}`DMTN-282` to decide on an initial implementation plan.
Since that meeting, work on client/server has progressed to the extent that a core system can be deployed on the RSP and there is support for retrieving datasets using signed URLs.
A meeting was held at SLAC at the Joint Technical Meeting in early 2024 February with representatives from the Data Facilities and architecture teams to discuss different options for deployment and implementation strategies for client/server at the various sites.
This document summarizes the outcomes of that meeting.

In this document the following terms are used:

* "direct butler" is a butler instance that communicates directly with a database and a file store.
* "remote butler" is a butler instance that communicates with a web server process and might require a proxy or signed URL to retrieve file contents.

## Core Assumptions

There are some core assumptions made by the developer team that should be spelled out explicitly to ensure that there is a shared understanding.

* A `Butler` can be instantiated from a URI to a configuration file without the user having to know whether they are connecting directly to a butler registry or via a client/server.
* Data Release Processing, including multi-site processing {cite:p}`DMTN-213`, will use a direct butler.
  Campaign processing is run by trusted users doing large graph builds.
* Any transfers of Prompt Processing products to butler repositories at the USDF will use direct butler.
* Developers logged in directly to the USDF will use direct butler.
  Permissions and auditing will be handled by creating user accounts in the PostgreSQL databases rather than the current use of shared accounts.
* Summit users do not need to access summit butlers using client/server and their notebooks can connect directly to Butler.
* All RSP deployments will require a client/server butler to implement VO services, but that server can be restricted to only those services.
* External Data Rights holders must always access butler datasets through client/server.
* It will be possible for a server to sign URLs such that datasets can be retrieved directly by clients without requiring a proxy server.
  (see also {cite:p}`DMTN-284`)

### Data Preview 0

When Data Preview 0.2 {cite:p}`RTN-041` is relocated from the Interim Data Facility to the USDF in August it will no longer be generally accessible to the data preview delegates.
Staff will still be able to access it from the USDF RSP using a direct butler connection.
The dataset can be used as a client/server testbed for the hybrid data access center approach where the client is on Google but the data are at SLAC.

### ComCam Data

* ComCam data (both raw and reduced) will not be made available to the Data Rights community until Data Preview 1 (DP1) {cite:p}`RTN-011` in early 2025.
  This includes access to prompt processing products.
* DP1 will use Butler client/server.
* The initial plan is for DP1 to be read-only with no ability to run `Butler.put()`.
* Since this is a static read-only dataset the ObsCore tables can be populated by exporting from the Butler registry and importing into Qserv {cite:p}`DMTN-243` as was done for DP0.2.
* Since this data repository is read-only there is no need for group or user collection management.

### Prompt Processing During Survey Operations

As soon as we start taking survey data the Data Rights community will expect to be able to retrieve the raw data and prompt processing outputs (once embargo period for that night has elapsed).
This will use a Butler client/server, likely with the registry and data being located at SLAC.
To support VO services "live" ObsCore will be used with Butler updating the ObsCore tables as needed {cite:p}`DMTN-236`.

## Implementation Decisions

### Querying

With potentially 10,000 simultaneous users running queries for datasets and dimension records, it is impossible for one server to be able to handle that load with one database.
To solve this problem there will be multiple strategies:

* Servers will pre-fetch some database dimension records and collection information.
* Multiple server instances will be deployed with load balancing.
* Multiple database server instances will be deployed.
* Servers will not run database queries directly, instead they will spawn workers that will run the queries.
* The number of queries a single user can execute will be limited and a queuing system employed.

#### Regular Expressions

Some of the butler APIs allow a user to specify a search using a regular expression or a glob.
Regular expressions are very useful but they are a very bad idea when the regular expression is coming from a potentially untrusted source and can even be a bad idea if a valid user is trying to do something clever and ends up with something that is very complicated.
It is easy to create a regular expression that can turn into a denial-of-service attack, we are therefore proposing that regular expressions be removed from the public APIs and that we only allow globs.

#### Dimension Record Pre-fetching

For a static data release the dimension coordinate space is pre-defined and no new records will be added.
Even for a live data repository, such as the Prompt Processing repository, certain record categories are only rarely augmented and are relatively small.

For example, the number of `instrument` records is of order 10, the number of `physical_filter` records is of order 1,000, and the number of `detector` records is of order 1,000.
These are very common dimensions and pre-fetching them into the server is very important to reduce database load.
If a new filter is added, this event can be planned for and a mechanism can be in place to invalidate the server copy to allow it to be recreated.

Skymaps and tract/patch definitions are also something that can be considered for pre-fetch.

Even for `day_obs` the total number of records is only about 4,000 for LSSTCam over the ten year survey (twice that if LATISS is included) with limited metadata attached to those records, and for a static data release we could consider pre-fetching those.

The situation is different for `exposure`, `visit` and `group` where there can be of order 1,000 of each of those created per night (millions of records).
Those records (including metadata fields) can not be pre-fetched into the server and must be obtained from the database every time they are needed.

#### Collection Caching

In a full data release where there can be many chained collections as well as tens of thousands of user collections, it is critical that a client request for a collection definition doesn't result in multiple database queries every single time.
A user will only be allowed to see their own user collections and any group collections they are part of but the total number of collections to manage will be very large.

Caching all the collection definitions in the server might be possible, so long as the cache is shared between server threads, but the issue is knowing when to invalidate the cache.
A data release will have a large number of unchanging collection definitions combined with user and group collections that can be created or disappear at any time.
A prompt processing repository will be getting new collections on a daily basis in addition to the user and group collections.
It might be necessary for specific collections to be marked as permanent to allow the servers to know that those will not be modified, but require database queries for the dynamic collections.
The approaches to collection caching are still to be worked out and we are investigating reorganizing how collections are stored in the database to make queries on them more efficient without the recursion currently used to support chains of datastores with chains of datastores.

It might be simpler for the client to never cache collection information if the server is caching.

#### Multiple Servers

A single server, however large, can not reliably handle thousands of simultaneous requests.
This is not helped by the butler APIs being designed before `async` was considered stable, requiring the server to create threads for all incoming requests and Python not currently being a language that supports good thread performance.

To allow arbitrary scaling we will deploy multiple server instances with some form of load balancing.
We are designing the server to be stateless such that any client can talk to any server at any time without having a client pinned to a specific server.

#### Multiple Database Servers

Deploying multiple Butler services allows client requests to be handled at scale but that needs to be matched by database capacity.
Multiple read-only clones of a data release can scale relatively easily.
There is a complication when clients are allowed to store data in the butler in terms of replication to replicas, and this is discussed later.

#### Query System

Many Butler queries can take minutes or even longer, and it is not desirable to lock up resources in the server process whilst waiting for these queries to complete.
The solution is to use a system where queries are sent to workers in an execution pool.
The client will be issued a job ID and can query the server to obtain the worker status.
The client will initially use polling to determine when a query has completed.
Once a job completes the client will receive a URI to the results in either JSON or parquet format.
These results will be written with an expiry date to allow for automatic cleanup of results that are no longer needed.
Additionally, context managers will be supported to allow the client to send a message to the server when it no longer needs the results.
If the query resulted in many results they can be written as multiple files and multiple URIs can be returned to allow the client to paginate.

Additionally, a queue system makes it possible for individual users who are doing many queries simultaneously to be rate-limited.
The system should not look like it is down for people if a single user is submitting hundreds of queries and is ahead of everyone else in the queue.

### Writing to a Butler Registry

The biggest complication in implementing client/server Butler is handling writes.
Clients will rarely need to update dimension records (those are created by instrument registration, raw data ingest, or visit definitions, which are generally thought of as administrative tasks that can, for now, use direct butler connections, although it might be necessary for users to create their own sky map definitions) but Data Rights holders will want to read datasets, construct new datasets derived from them, and store them back in a butler.

We assume that there will be far fewer writes than there will be reads and most of the writes will involve multi-dataset transfers from personal workspaces or from user-batch outputs rather than people calling `butler.put()` from notebooks.

There is still some debate over where user-generated products will be stored and that is discussed [later in this document](#writing-datasets).

We have considered multiple options.

#### Use primary database

In this scenario, whilst queries can be spread across multiple database servers, writes always go to a single database server.
Those updates are then replicated to the other servers, albeit with no immediate consistency guarantees and the possibility that someone could put a dataset and not be able to retrieve it straightaway due to replication latency.

This is the simplest approach and closely matches how writes are handled in the direct butler.
It does mean that it would be possible for a user to put a dataset and not be able to get it back straightaway due to replication delays -- they will not be able to guarantee that their get uses the initial database server.
Blocking the put until the records have been fully replicated is not desirable.

#### Use Local SQLite

Another option is to use entirely "local" storage.
In this scenario people use a local SQLite registry with their POSIX disk allocation, transferring dataset registry entries to their local butler, processing them locally, and keeping the output datasets local.
This could be expanded to be a more permanent local storage solution, although there are limits as to how big a SQLite registry can usefully be.

This approach does not work with the concept of allowing scientists to collaborate on their research since the datasets and registry would be completely private.
It also does not work well with the concept of user batch {cite:p}`DMTN-223` where data are processed at the USDF but there is no way for a cloud RSP user to see the outputs if they are using a private Butler in the cloud.

#### Butlers for Everyone

An option proposed at the Joint Technical Meeting in February was to give everyone a personal butler PostgreSQL registry along with a personal object store bucket.[^oprah]
This is intriguing because it allows for everyone to be completely independent, allows us to allocate users to specific servers, and would not require us to replicate all registry entries to every server.
It would also be possible to create group-based butlers for people to collaborate on – if a scientist wants to make a dataset available to the group they can transfer the dataset to that butler using `Butler.transfer_from` in the normal way without changing the UUID of the dataset.

Seeding these personal butlers would involve using `Butler.transfer_from` from the primary butler (but not copying any of the files, instead somehow enforcing a `direct` transfer mode be used), which leads to the possibility of someone seeding their butler with all of the datasets from the primary one.
At that point you might have 10,000 or more personal replicas of the entire data release registry, which is less than ideal.
It might be necessary to explicitly block the transfer of datasets from the primary butler to the user butler using `transfer_from` and only allow new datasets to be stored – a `butler.get` from the primary followed by a `butler.put` to the user butler can not be prevented, but this would result in the copied dataset using the user's quota and so would be self-limiting.
If a user wants to query multiple collections across both butlers they would have to do that as independent queries.

A more general problem with separate registries is that Butler can not currently query two registries simultaneously.
To build a quantum graph involving some datasets from a data release and some from a user registry is not yet possible and would require considerable thought.
We also need to understand if this is handled as two separate butlers from the user perspective or if the user has a single butler instance that is combined internally (either through the server being told to use multiple registries or the remote butler client talking to two distinct butlers internally).

Furthermore, schema migrations and dimension universe changes must be handled in a uniform manner to prevent a user from being unable to combine their personal registry with the required data release.
A DR11 schema and universe will most likely be different to a DR1 schema/universe.
One feature of the "butler for everyone" approach is that you would prefer a user to use a single database schema for the duration of the survey so that they do not have 11 registries with their files scattered across them.
This not only implies that there are controlled schema migrations performed by administrative staff throughout the survey lifetime (ensuring that user schemas are administratively under our control and not the user), but that the DR1 release will be updated to use a newer schema and possibly require a patched version of the DR1 software to use it.

#### A single user registry

A read-only registry has many advantages over a registry that is constantly receiving updates.
In particular deployments of new replicas or restoring from backups or disaster recovery is very easy.
Isolating the user data from the data release data also more easily allows us to understand the performance characteristics of the registry.
Additionally, it may be desirable for people to be able to retrieve their derived products after the formal data release has been deleted.
User registries still have to be backed up but they will generally be significantly smaller than the formal data release registry.
For prompt processing it becomes critical in that the prompt processing registry is updated every night and is likely a replica of an internal registry.
If the user-facing registry has user data products in it, this implies that it can't be a simple backup of the internal registry and instead must be backed up independently.
If there is some issue with the internal registry that is fixed, it is likely that the user-facing version would need to be fixed as well – for a replica this is trivial but for a distinct registry this becomes an operational overhead.

For this reason we must consider the option of a read-only user-facing registry containing the formal observatory data products, and a single user registry containing all the other datasets.
As for the previous option, it will be necessary for the graph building to be modified to allow multiple registries (without which user batch processing is not possible) and some clarification as to how the user interacts with the two registries.

One downside of a single user registry relates to the handling of collaborations.
If datasets are written to a user run collection they can not currently be reallocated to a group run collection.
Butler requires that the dataset be retrieved and then stored in the new location, resulting in the dataset UUID being changed and potentially breaking the provenance chain with the original dataset.

### Hybrid Cloud

The proposal for operations is for the users to be in the cloud but the Data Release data to be stored at the USDF. {cite:p}`DMTN-287`
It has been proposed that a cache be used in the cloud such that commonly-accessed datasets will be pulled from cloud storage rather than continually being fetched from USDF.
This would require a server side proxy that would transfer the file to the cloud and then issue a signed URL or else forward the file directly to the user.
The design of this cache is possibly complex.
An alternative is to have a fixed cache for datasets such as the deep coadds, and to always retrieve those from the cloud -- this can be done straightforwardly by configuring the datastore in the server to look at multiple buckets and registering the coadds in the cached datastore explicitly.

### Writing Datasets

Users will want to be able to store datasets in a butler repository.
For Data Preview 0.2 {cite:p}`RTN-041` three buckets on Google were used: one for the raw data, one for the the data release products themselves, and one for user products.
The first two buckets were read-only and the user bucket was accessed via the use of a chained datastore in the butler configuration.

In operations a similar pattern will be used in that the raw data and formal data products must be stored in read-only locations, but with the additional constraint that users will have [quotas](#quotas) on the amount of storage they can use.

There are many options where a dataset could be stored when the user issues `butler.put()`:

1. Their personal POSIX storage space.
2. Per-user and per-group buckets in the cloud or at the USDF.
3. A general user bucket in the cloud or at the USDF per butler where `u/$USER/` is enforced.

Option #1 allows a single storage area for all the user's data products and simplifies the calculation of quotas.
It does not provide any means of supporting collaborative groups and that likely means this option is not tenable other than in the narrow "workspace" concept described in {cite:p}`DMTN-271`.

The remaining options will all involve the server generating signed URLs for writes.

Option #2 solves the collaboration problem at the expense of creating tens of thousands of buckets.
Option #3 is similar to how we currently handle user data products but does potentially require more work to handle quotas than option #2.

The choice between USDF and Cloud storage buckets depends on the cost of cloud storage versus USDF storage (based on the expected size of user data products) and how user batch {cite:p}`DMTN-223` interacts with cloud users.

All user batch processing will take place at the USDF such that any user products to be used as inputs for that processing must be available at the USDF (the generated graph should not use signed URLs to locate the files in the cloud since signing happens in the server and the server is not available at batch execution time and signed URLs can not be stored in the graph because of expiry times) and all the data products created by the processing must be made available to the user and stored in their user or group collection.
This constraint from batch processing suggests that USDF may be the best option since the files can be read from and written to the final location at USDF without requiring transfers.

```{warning}
It has been noted that Weka can only support up to 10,000 buckets.
This is similar to the expected number of users and implies that if Weka is required as the USDF object store interface and user products are to be stored at USDF, then option #2 can not be used.
```

One additional thing to consider is how to handle user storage with multiple data releases.
The first two options listed would need to have additional information in the file hierarchy to reference the relevant parent butler repository.
The final option would use a server configuration to indicate where the specific user bucket is located as is done for DP0.2.

#### Quotas

```{note}
Are users given distinct quota allocations for their RSP POSIX file space and Butler dataset usage?
```

Dataset file quotas for RSP users' POSIX file space will be monitored by the RSP infrastructure itself.
Users will not have direct access to buckets without being mediated by the Butler server issuing signed URLs.

For butler datasets quota management depends on how datasets are stored.
For a shared bucket the butler server has to know (a) how much quota a user has, and (b) how much of the allocation they have utilized, before deciding whether a dataset can be stored.
Care must be taken to deal with a user attempting multiple writes in parallel.

Running a daily "cron" job to scan the bucket to calculate quota allocation does not seem feasible given how much people could store in a single day (especially for the outputs of user batch).
This likely means that there will have to be some lightweight database tracking quotas for each user across all their buckets, with each write or deletion adjusting the quota value as appropriate.

```{note}
Since the system doesn't know the size of the file until it has been written with the signed URL, does that mean that the file can be deleted by the server and the write rejected if it causes the user to exceed their quota?
Are they allowed to write one file over their quota?
Can they write as much as they want until a new day starts?
What about parallel writes?
How strict are the requirements?
```

The per-user/group bucket possibly makes quota tracking somewhat easier in that the storage system knows how much data is in each bucket without having to check the size of individual files.
Unfortunately this information does not seem to be available outside of administrative screens and there may not be a public API for obtaining the size of a bucket.
It may still be necessary to implement a quota tracking system independently of the underlying storage APIs.

Once the current size of the user's storage usage has been determined it is possible in some S3 APIs to specify the maximum size of file that a user can upload, which would allow a quota limit to be enforced without requiring that the file be uploaded and then deleted.

### Prompt Processing Repository

The prompt processing repository differs from a data release repository in that it is being updated on a daily basis for the entire duration of the survey, but also datasets are regularly being deleted from it.
User products stored in the repository will include provenance to datasets that only exist in registry and can not themselves be retrieved.

The long life time of the repository indicates that there will have to be migrations to support the evolving LSST Science Pipelines and evolving butler registry schema migrations.

For example, at the end of the survey there will be no datasets written by prompt processing in the repository but there might be derived user products.
Users are unlikely to want their derived products to be deleted automatically if they have sufficient quota available.

Reading these old files might be difficult given data model evolution, and possibilities for supporting this are discussed in the [next section](#accessing-multiple-data-releases).

```{note}
Will the DR10 data release pipelines software be required to read (without loss of information) datasets written by the DR1 software?
Being able to use outputs from DR1 in pipelines designed for DR11 can not be guaranteed since it is possible that the information content of a early dataset will differ from that of a later dataset.
```

### Accessing Multiple Data Releases

Over time people will be writing derived data products associated with a data release and then a new data release will be issued.
By default those previous datasets will not be visible to the butler containing the new data release.
As long as the previous butler is accessible a user can create two `Butler` instances to access those datasets.
If older data releases are completely removed (including the registry) then this will not work and tooling must be provided to allow people to migrate datasets from older data release butlers to newer data releases.
This tooling does exists (see `Butler.transfer_from()`) but may need to be push-button activated.

If these datasets are migrated from data release to data release it will be necessary to ensure that the formatters used to read files continue to function across every data release.

One proposal being considered is to switch to using labeled formatters in the datastore records rather than the full name of a python class.
The labeled formatter would then be mapped to an explicit python class defined within the butler configuration, allowing the mapping to change as new versions are released.
These labels could then be versioned (e.g., `ExposureFormatterV1`) to allow old files to be read by new software even if there has been a major change of data model, even allowing a formatter to return a different Python type than it did for an earlier data release.

A related issue is Storage Class definitions.
These definitions (mapping a butler dataset type to a python type) are currently assumed to be global for all butlers but it is forseeable that a dataset type in DR1 might use a different python type to that used for the same dataset type in DR11 in a way that is not compatible with the type conversion system that is available.
This may lead to having to have per-butler storage class definitions, much like we have per-butler dimension universes.

### Server Evolution

One approach to deploying multiple data releases is for the Butler server used to access the data release to be the exact same server implementation that the data release was initially shipped with using a `daf_butler` version and dimension universe directly matching that initial version.

This sounds easy but public-facing servers (especially if we make them generally available such that data rights holders can access the butler outside of the RSP) present inherent security risks if they are left as-is over the years.
Tracking security updates means that we can not leave old server deployments unchanged but must keep them up to date.
Since the server code itself does not depend on the LSST Science Pipelines software the simplest approach is to ensure that all servers, regardless of data release, are updated to the same version.

This will require some planning in terms of:

1. Ensuring that old dimension universes are always supported.
2. Managing API version changes such that older client libraries from earlier data releases can still connect to the server until such a time that they can be patched to support the newer API.

Regarding the second point, the simplest approach for the older releases is that the server software always supports the older APIs.
It may, though, be better in the longer term to only support a single server API.
Modifying older DR releases to use new APIs has some difficulties in terms of making patch releases when the Conda environment is effectively pinned to the versions that it was originally built with.
If the back port of support for the new server to the older client can work within the constraints of the older package versions this can probably work, but if some update is needed to a dependent package it may well be difficult to solve the Conda environment.

```{warning}
If there is a policy to drop support completely for older data release software versions this will affect how we approach the question of server evolution.
Even though the project is only required to keep the previous two data releases available it is possible for other locations to host an older data release, it is possible that someone would like to use the DR1 software to process a raw file, and it is possible that we would keep the DR1 registry accessible even with the data files removed.
```

## References

```{bibliography}
  :style: lsst_aa
```

[^oprah]: This was dubbed the "Oprah" model of butler allocations: "You get a butler! You get a butler! Everybody gets a butler!"
