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
Since that meeting work on client/server has progressed to the extent that a core system can be deployed on the RSP and there is support for retrieving datasets using signed URLs.
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



## References

```{bibliography}
  :style: lsst_aa
```
