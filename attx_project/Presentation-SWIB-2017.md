# SWIB 2017

http://swib.org/swib17/ Deadline extended till May 28.

## (Accepted) Proposal

### Practical Data Provenance in Distributed Environment or:
#### Implementing Linked Data Broker Using Microservices Architecture

Maintaining some sort of data provenance, i.e. the data about the actions that have led the target data to its current state, is an integral feature of a system acting as a data broker. After all, brokering can be seen as an activity that transforms external data sources, which one might not have any control over, into new data source. This transformation can involve complex processing steps, which all contribute to the provenance data. Keeping track of who did what, why and when, is therefore necessary in order be able to ascribe responsibility of, e.g. data quality, to the right (human or software) entity.

We have been developing a data broker solution based on semantic web technologies that is flexible and extendible both in terms of incoming and outgoing data, as well as the cloud based infrastructural resources employed to operate the broker instance. Our solution consists of components implementing different types of services such as workflow and graph management, processing, distribution and provenance.

We present the result of the ATTX project, which provides a set of software components that can be used to build scalable data brokers that work on linked data. We will cover issues and implementation related to modelling, acquisition, exposing and using provenance information produced by services that comprise the ATTX data broker instance.
