# Ekala Enhancement Proposals

This repository is meant to facilitate proposals to the Ekala ecosystem, both in
technology and in processes. This is heavily inspired by [Python Enhancement proposals](https://peps.python.org/pep-0001/),
in which there is a much tighter iteration loop on submitting proposals than the
existing NixOS RFC process.

## EEP Types


There are three kinds of EEP:

- Standards Track EEP - Describes a new feature or implementation for the Ekala ecosystem. Generally this is a change to one or more technology in Ekala project.
- Informational EEP - Describes an existing issue, or provides general guidelines or information to the Ekala project, but does not propose a new feature. Informational EEPs do not necessarily represent an Ekala project consensus or recommendation, so users and implementers are free to ignore Informational EEPs or follow their advice. The desire to identify issues in a formal and complete manner.
- Process EEP - Describes a process surrounding Ekala, or proposes a change to (or an event in) a process. Process EEPs are like Standards Track EEPs but apply to areas other than projects in the official Ekala ecosystem. They may propose an implementation; they often require majority contributor consensus; unlike Informational EEPs, they are more than recommendations, and users are typically not free to ignore them. Examples include procedures, guidelines, changes to the decision-making process, and changes to the tools or environment used in Ekala ecosystem. Any meta-EEP is also considered a Process EEP.

## EEP Template

Headers marked with “*” are optional and are described below. All other headers are required.

```
  EEP: <eep number>
  Title: <eep title>
  Author: <list of authors' real names and optionally, email addrs>
* Sponsor: <real name of sponsor>
* EEP-Delegate: <EEP delegate's real name>
  Discussions-To: <URL of current canonical discussion thread>
  Status: <Draft | Active | Accepted | Provisional | Deferred | Rejected |
           Withdrawn | Final | Superseded>
  Type: <Standards Track | Informational | Process>
* Topic: <Governance | Packaging | Release | Typing>
* Requires: <eep numbers>
  Created: <date created on, in YYYY-MM-DD format>
  Post-History: <dates, in dd-mmm-yyyy format,
                 inline-linked to EEP discussion threads>
* Replaces: <eep number>
* Superseded-By: <eep number>
* Resolution: <url>
```
