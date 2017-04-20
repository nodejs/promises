# promises

Promises Working Group Repository

This repo serves as a central place for Node.js Promises work coordination.

The Promises WG is dedicated to the support and improvement of Promises in Node.js
core as well as the larger Node.js ecosystem.

## Lifecycle

The concerns and responsibilities of this Working Group will change over time. There
are three distinct stages of this WG:

### Stage 0: Landing a Flagged Promises API

The first stage of this Working Group is focussed on the specifics of exposing a Promise
API from Node core. The focus at this stage is in identifying blocking concerns raised
by stakeholders, and triaging them into "blocking the landing of a flagged implementation",
"blocking the unflagging of an implementation", and "non-blocking."

You can track the status of these issues with the
[`blocks-landing-pr`][issues-blocked-landing],
[`blocks-unflagging`][issues-blocked-unflagging], and
[`not-blocking`][issues-non-blocking] labels.

**If you feel an issue is not appropriately blocking**, please raise your concern in the
issue.

When the issues marked [`blocks-landing-pr`][issues-blocked-landing] are all
resolved, and the PR has been merged, this WG will move on to stage 1.

### Stage 1: Creating a Litmus test for Unflagging

In this stage we will attempt to address all issues marked as
[`blocks-unflagging`][issues-blocked-unflagging]. The focus of this stage will be
to present an acceptable "go / no go" check for unflagging promises to the Node CTC.
If accepted, once the "go / no go" check has passed, we will unflag the Promises API.

Once the Promises API has landed, this WG will proceed to stage 2.

### Stage 2: Continued Support

The WG will be available to support and advise the Core repository on matters
related to promises.

### Membership

Working Group Members:
 - @benjamingr
 - @refack
 - @petkaantonov
 - @vkurchatkin
 - @trevnorris
 - @kriskowal
 - @MadaraUchiha
 - @omsmith
 - @thefourtheye
 - @erights
 - @chrisdickinson
 
# Participation

We are currently looking for more participants in the working group - see https://github.com/nodejs/promises/issues/1

[issues-blocked-landing]: https://github.com/nodejs/promises/issues?q=is%3Aopen+is%3Aissue+label%3Ablocks-landing-pr
[issues-blocked-unflagging]: https://github.com/nodejs/promises/issues?q=is%3Aopen+is%3Aissue+label%3Ablocks-unflagging
[issues-non-blocking]: https://github.com/nodejs/promises/issues?utf8=%E2%9C%93&q=is%3Aopen+is%3Aissue+label%3Anot-blocking
