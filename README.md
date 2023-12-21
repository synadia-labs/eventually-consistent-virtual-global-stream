Collection of config files and scripts to run locally a simple example of a global eventually consistent stream `foo` that listens on subjects `foo.>` over a multi-region NATS JetStream Super-Cluster.

# Walkthrough
Make sure to install (or upgrade to) the latest version of the [NATS server](https://github.com/nats-io/nats-server/releases/latest) and of the [`nats` CLI tool](https://github.com/nats-io/natscli#installation) on your local machine.

## The setup
This walkthrough will create and start locally a total of 9 nats-servers organized in 3 clusters `east`, `central` and `west` of 3 nodes each interconnected as a Super-Cluster. Once those servers are started it will create all of the 'read' and 'write' streams for all 3 regions.

You will then be able to play with the virtual stream `foo` using `nats` by connecting to different local clusters and using and publishing or reading with the (virtual) stream `foo` as if it were a single globally replicated stream.

## Server configurations
The individual server configuration files are straightforward. Each server establishes route connections to its 2 other peers in the cluster, and the clusters are connected via gateway connections. In this example, all of the individual server's configuration files import a single `mappings.cfg` file containing all of the Core NATS account level subject mapping transforms, which in this case are all cluster-scoped. If you were running your servers in the 'operator' security mode, those mappings would be stored (in the account resolver) as part of the account(s) JWT(s) instead.

## Start the servers
You can start the entire Super-cluster using the provided simple script.
```bash
source startservers
```
This script also defines 3 `nats` contexts to allow you to easily select which cluster you want to connect to: `sc-west`, `sc-central` and `sc-east`.
## Defining the local streams
After a few seconds the Super-Cluster should be up and running, and then define for the first time all of the required local streams that are configured using JSON files and there is a simple convenience script to define them all.
```bash
source definestreams
```
The local 'write' streams are quite straightforward: they are named `"foo-write-<region>"` and all they need to do is listen on the subjects `"foo.<region>.>"`:

Note that in this example a max-age limit of `3600000000000` (1 hour) set on the 'write' streams, meaning that the maximum length of a regional outage or split-brain that can be recovered from without any message write loss is 1 hour. You need a limit to ensure that the 'write' streams don't just grow forever as they only need to hold data for as long as the outage lasts, adjust this limit to fit your specific requirements.

The local 'read' streams don't listen to any subjects and source all of the 'write' streams (see the `sources` array) and perform a simple subject transformation to drop the token in the subject name that contains the name of the region of origin (see the `subject_transform` stanza).

So using the region 'west' as an example a message published on `foo.test` by an application connected to the 'west' cluster will be first stored with the subject `foo.west.test` in the `foo-write-west` stream and the stream `foo-read-west` sources from `foo-write-west` and strips the second token of the subject such as the message ends up being stored in that stream with the subject `foo.test`.

![Subject Transformation and stream configuration for region 'west'](global-virtual-region.png)

Drawing of the transformation of the subject of a message published on `foo.test` in region 'west' as it makes its way from a publishing to a consuming client application.

## Interacting with the global virtual stream

You can use `nats --context` to interact with the stream as would a client connecting to the different clusters.

For example let's connect to the 'west' cluster and publish a message on the subject `foo.test`:
```bash
nats --context sc-west req foo.test 'Hello world from the west region'
```
Using `nats req` rather than `nats pub` here in order to see the JetStream publish acknowledgement just like a client application would when using the JetStream `publish()` call and checking that the `PubAck` does not contain an error.

We can then check that the message has indeed propagated to all the regions, in this example using the `nats stream view` command (that creates an ephemeral consumer on the stream and then iterate over it to get and display the messages).
```bash
nats --context sc-west stream view foo
```

You can see that the message stored in the global virtual 'foo' stream is indeed there with the subject `foo.test` which we used earlier to publish the message. Let's check that the message has also made it to the other clusters:
```bash
nats --context sc-central stream view foo
```
and
```bash
nats --context sc-east stream view foo
```
You can also even do a `nats stream info` on the virtual stream (this will show you the info about your local 'read' stream), but note how `nats stream ls` doesn't show the global virtual stream, but rather all of its (non-virtual) underlying local streams.

## Simulating disasters
You can simulate whole regions going down by killing all of the `nats-server` processes for a region, there are some simple convenience scripts in the repository to kill or restart regions easily.

### Killing one region
For example: let's first kill the central region cluster
```bash
source killcentral
```
Then publish message from or 'east'
```bash
nats --context sc-east req foo.test 'Hello world from the east region'
```
Check that the message made it to 'west'
```bash
nats --context sc-west stream view foo
```
Then restart 'central'
```bash
source startcentral
```
It may take up to a couple of seconds for the recovery to complete then check that the message is now there in 'central'
```bash
nats --context sc-central stream view foo
```

### Killing two regions to go down at once and simulating a split brain
The two failure scenarios are similar and related: a split brain from the point of view of the region getting isolated is no different from both of the other two regions going down at the same time.

The difference being that in the case of split brain, the two other regions that can still see each other continue to operate normally (including processing new 'writes') and the isolated regions ends up in the same 'limited' mode of operation as in the case when two regions do down at the same time.

As soon as the network partition gets resolved or as the missing regions come back up the two parts of the brain will replicate missed messages between themselves and eventually become consistent again (though not necessarily in the same order).

In the case of two regions going down at the same time or of being the smaller part of the split brain the remaining region can still operate but in a 'limited' fashion, as not all functionality will be available since there will be an inability for the remaining nodes to elect a JetStream 'meta leader'.
- Publications to the stream _will still work_, the only way publications to stream in a regions would stop working is if 2 of the 3 servers in the region (or 2 out of 5) go down at the same time.
- Get operations (e.g. what the KV 'get' operation uses) _will still work_.
- Getting messages from already existing consumers (at the time the second regions goes down) on the stream _will still work_, and locally published messages will be seen in the 'read' stream right away.
- However, creating new consumers (or new Streams) _will not work_.

First kill both 'west' and 'east'
```bash
source killwest; source killeast
```
Publish a new message on 'central' (as if it was isolated region)
```bash
nats --context sc-central req foo.test 'Hello world from the central region'
```
Then bring down the 'central' region and 'east and 'west' back up
```bash
source killcentral; source startwest; source starteast
```
Wait up to a couple seconds and publish another message from one of those two regions
```bash
nats --context sc-east req foo.test 'Hello again from the east region'
```
Check you can create a new consumer and see that message from the other region
```bash
nats --context sc-west stream view foo
```
And finally resolve the split brain by restarting 'central'
```bash
source startcentral
```
After a few seconds you can see that all the messages where are now present in all the 'read' streams, though not necessarily in the same order by comparing the output of
```bash
nats --context sc-west stream view foo
```
With
```bash
nats --context sc-central stream view foo
```