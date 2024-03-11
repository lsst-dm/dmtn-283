# Butler Client/Server Implementation and Deployment Strategies

```{abstract}
There are many approaches that can be taken when considering where to deploy the Butler servers and how to implement a server in a way that can support the performance requirements.
This document proposes some strategies that came out of the recent Joint Technical Meeting held at SLAC in Februrary 2024.
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

## Core Assumptions

There are some core assumptions made by the developer team that should be spelled out explicitly to ensure that there is a shared understanding.

* A `Butler` can be instantiated from a URI to a configuration file without the user having to know whether they are connecting directly to a butler registry or via a client/server.
* Data Release Processing, including multi-site processing {cite:p}`DMTN-213`, will use direct butler.
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

When Data Preview 0.2 is relocated from the Interim Data Facility to the USDF in August it will no longer be generally accessible to the data preview delegates.
Staff will still be able to access it from the USDF RSP using a direct butler connection.
The dataset can be used as a client/server testbed for the hybrid data access center approach where the client is on Google but the data are at SLAC.

### ComCam Data

* ComCam data (both raw and reduced) will not be made available to the Data Rights community until Data Preview 1 (DP1) {cite:p}`RTN-011` in early 2025.
  This includes access to prompt processing products.
* DP1 will use Butler client/server.
* The initial plan is for DP1 to be read-only with no ability to run `Butler.put()`.
* Since this is a static read-only dataset the ObsCore tables can be populated by exporting from the Butler registry and importing into Qserv as was done for DP0.2.
* Since this dataset is read-only there is no need for group or user collection management.

### Prompt Processing During Survey Operations

As soon as we start taking survey data the Data Rights community will expect to be able to retrieve the raw data and prompt processing outputs (once embargo period for that night has elapsed).
This will use a Butler client/server, likely with the registry and data being located at SLAC.
To support VO services "live" ObsCore will be used with Butler updating the ObsCore tables as needed {cite:p}`DMTN-236`.

## Implementation Decisions

### Querying

With potentially 10,000 simultaneous users running queries for datasets and dimension records, it is impossible for one server to be able to handle that load with one database.
To solve this problem there will be multiple strategies:

* Servers will cache some database dimension records.
* Multiple server instances will be deployed with load balancing.
* Multiple database server instances will be deployed.
* Servers will not run database queries directly, instead they will spawn workers that will run the queries.


#### Record Caching

For a static data release the dimension coordinate space is pre-defined and will no new records will be added.
Even for a live data repository, such as the Prompt Processing repository, certain record categories are only rarely augmented and are relatively small.

For example, the number of `instrument` records is of order 10, the number of `physical_filter` records is of order 1,000, and the number of `detector` records is of order 1,000.
These are very common dimensions and caching them in the server is very important to reduce database load.
If a new filter is added, this event can be planned for and a mechanism can be in place to invalidate the cache to allow it to be recreated.

Even for `day_obs` the total number of records is only about 4,000 for LSSTCam over the ten year survey, and for a static data release we could consider caching those.

The situation is different for `exposure`, `visit` and `group` where there can be of order 1,000 of each of those created per night.
Those records can not be cached.

#### Multiple Servers

A single server, however large, can not reliably handle thousands of simultaneous requests.
This is not helped by the butler APIs being designed before `async` was considered stable, requiring the server to create threads for all incoming requests.

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
Once a job completes the client will receive [question: are we using keep-alive websockets or something, or polling?] a URI to the results in either JSON or parquet format.
These results will be written with an expiry date to allow for automatic cleanup of results that are no longer needed.
If the query resulted in many results they can be written as multiple files and multiple URIs can be returned to allow the client to paginate.

### Writing to a Butler Registry

The biggest complication in implementing client/server Butler is handling writes.
Clients will never need to update dimension records (those are created by instrument registration, raw data ingest, or visit definitions, which are generally thought of as administrative tasks that can, for now, use direct butler connections) but Data Rights holders will want to read datasets, construct new datasets from them, and store them back in a butler.

There is still some debate over where user-generated products will be stored.

We have considered multiple options.

#### Use primary database

In this scenario, whilst queries can be spread across multiple database servers, reads always go to a single database server.
Those updates are then replicated to the other servers, albeit with no immediate consistency guarantees and the possibility that someone could put a dataset and not be able to retrieve it straightaway due to replication latency.

This is the simplest approach and closely matches how writes are handled in the direct butler.

#### Use Butler Workspaces

An option explored in {cite:p}`DMTN-271` is to use the concept of butler workspaces.
In this scenario people use a local SQLite registry with their POSIX disk allocation, transferring dataset registry entries to their local butler, processing them locally, and keeping the output datasets local.
This could be expanded to be a more permanent local storage solution, although there are limits as to how big a SQLite registry can usefully be.

This approach does not work with the concept of allowing scientists to collaborate on their research since the datasets and registry would be completely private.
It also does not work well with the concept of user batch {cite:p}`DMTN-223` where data are processed at the USDF but there is no way for a cloud RSP user to see the outputs if they are using a private Butler in the cloud.

#### Butlers for Everyone

An option proposed at the Joint Technical Meeting in February was to give everyone a personal butler PostgreSQL registry along with a personal object store bucket.[^oprah]
This is intriguing because it allows for everyone to be completely independent, allows us to allocate users to specific servers, and would not require us to replicate all registry entries to every server.
It would also be possible to create group-based butlers for people to collaborate on -- if a scientist wants to make a dataset available to the group they can transfer the dataset to that butler using `Butler.transfer_from` in the normal way without changing the UUID of the dataset.

Seeding these personal butlers would involve using `Butler.transfer_from` from the primary butler (but not copying any of the files, instead somehow enforcing a `direct` transfer mode be used), which leads to the possibility of someone seeding their butler with all of the datasets from the primary one.
At that point you might have 10,000 or more personal replicas of the entire data release, which is less than ideal.
It might be necessary to explicitly block the transfer of datasets from the primary butler to the user butler using `transfer_from` and only allow new datasets to be stored -- a `butler.get` from the primary followed by a `butler.put` to the user butler can not be prevented, but this would result in the copied dataset using the user's quota and so would be self-limiting.
If a user wants to query multiple collections across both butlers they would have to do that as independent queries.

A more general problem with separate registries is that Butler can not currently query two registries simultaneously.
To build a quantum graph involving some datasets from a data release and some from a user registry is not yet possible and would require considerable thought.

#### A single user registry

A read-only registry has many advantages over a registry that is constantly receiving updates.
In particular deployments of new replicas or restoring from backups or disaster recovery is very easy.
Given that user registries still have to be backed up this is not a particularly useful distinction for a formal data release.
For prompt processing it becomes critical in that the prompt processing registry is updated every night and is likely a replica of an internal registry.
If the user-facing registry has user data products in it, this implies that it can't be a simple backup of the internal registry and instead must be backed up independently.
If there is some issue with the internal registry that is fixed, it is likely that the user-facing version would need to be fixed as well -- for a replica this is trivial but for a distinct registry this becomes an operational overhead.

For this reason we must consider the option of a read-only user-facing registry containing the formal observatory data products, and a single user registry.

An alternative to the single database registry is for there to be a read-only Data Release butler registry and a distinct user registry where all user and group collections live.



### Writing Datasets

Where?
How to quota?
User batch complication.



## References

```{bibliography}
  :style: lsst_aa
```

[^oprah]: This was dubbed the "Oprah
