# Informative Description of the Proposal to Support the Exchange of Structured Information using FHIRcast

This is informative content provides a description of the proposal for sharing additional information in a FHIRcast integration. This proposed operations are additions to FHIRcast STU2 and are backwards compatible with implementations of the FHIRcast STU2.  The additions are necessary only for Hubs and clients which wish to support the exchange of structured information.

## Sharing of Structured Information in a FHIRcast Context

FHIRcast is a  standard for real-time context synchronization between healthcare applications. For example, a radiologist typically works in disparate client applications at the same time (e.g. a reporting application, a PACS viewer, an interactive AI application, and an EMR). Clinical users want each of these systems to display use a shared context to synchronize their information displays to the same study and patient (and additional contextual subjects). In addition to basic context synchronization, this proposal extends FHIRcast to support real-time structured data exchange between synchronized client applications.

#### Concepts and Terminology

In addition to the core FHIRcast constructs such as topics, subscriptions, and FHIRcast Hubs; the following terminology is introduced to support the sharing of structured information.

 Concept | Description
--- | ---
context | Resource on which the user is currently working such as documenting information associated with a patient's encounter with a care provider. In this case the context is a specific encounter and the patient who is the subject of the encounter.
anchor context | A context which is serving as a container for FHIRcast events that enable sharing of content. The content sharing events include the anchor context and content shared in these events. Typical anchor contexts are resources such as Patient, Encounter, ImagingStudy, and DiagnosticReport.
content | Resources created during a user's interaction with the active context. For example, if the current context is an imaging study the user may make a measurement resulting in an observation containing the measurement information. The shared content either directly or indirectly references the resource acting as the anchor context.

##### Transactional Updates
A key concept of the information sharing events is that information is shared in a transactional manner.

![**Transactional Updates**](img/TransactionalUpdates.png)

The above diagram shows a series of operations beginning with a [`[FHIR resource]-open`](#fhir-resource-open) request followed by three [`[FHIR resource]-update`](#fhir-resource-update) requests.  The content in an anchor context is built up by the successive [`[FHIR resource]-update`](#fhir-resource-update) requests which contain only changes to the current state.  These changes are propagated by the Hub to all subscribed clients using a [`[FHIR resource]-update`](#fhir-resource-update) events containing only the changes to be made.

In order to avoid lost updates and other out of sync conditions, the Hub serves as the transaction coordinator.  It fulfills this responsibility by creating a version of the content's state with each operation.  If an operation is requested by a client which provides an incorrect version, this request is rejected.  This approach is similar to the version concurrency approach used by [FHIR versions and managing resource contention](https://www.hl7.org/fhir/http.html#concurrency).  Additionally, many of the FHIRcast content sharing concepts have similarities to the [FHIR messaging mechanisms](https://www.hl7.org/fhir/messaging.html) and where possible the basic concepts are aligned.

FHIR resources are used to convey the structured information being exchanged in [`[FHIR resource]-update`](#fhir-resource-update) operations.  However, it is important to note that these resources may never be persisted in a FHIR server.  During the exchange of information, the content (and hence resources) may be very dynamic in nature with a user creating, modifying, and even removing information which is being exchanged.  For example, a measurement made in an imaging application may be altered many times before it is finalized and it may be entirely removed from the content in the anchor context.

### Responsibilities of a FHIRcast Hub and a Subscribed Client

The existing responsibilities of a FHIRcast Hub remain when supporting information exchange operations:
1. Accept subscriptions to new or already established topics
2. Maintain a list of subscribers and events for which they would like notification
3. Distribute events to subscribers as appropriate

Hubs must fulfill these addition responsibilities when supporting information exchange operations:
1. Assign and maintain an anchor context's `versionId` when processing a [`[FHIR resource]-open`](#fhir-resource-open) request
2. Validate [`[FHIR resource]-update`](#fhir-resource-update) and [`[FHIR resource]-select`](#fhir-resource-select) requests for the correct `versionId` and reject the request if the version does not match the current `versionId` by returning a 4xx/5xx HTTP Status Code rather than updating the content
3. Assign and maintain a new `versionId` for the anchor context's content and provide the new `versionId` in the event corresponding to the validated request 
4. Maintain a list of current FHIR resource content in the anchor context so that it may provide the anchor context's content in response to a [`[FHIR resource]-open`](#fhir-resource-open) request and when responding to a [`GET Context`](#get-context) request
5. When a [`[FHIR resource]-close`](#fhir-resource-close)  request is received, the Hub no longer retains the content for that anchor context

A Hub is not responsible for structurally validating FHIR resources.  While a Hub must be able to successfully parse FHIR resources in a manner sufficient to perform its required capabilities (e.g. find the `id` of a resource and the `versionId` of the anchor context's content), a Hub is not responsible for additional structural checking. 

A Hub is not responsible for any long-term persistence of shared information and should purge the content when a [`[FHIR resource]-close`](#fhir-resource-close) request is received.

Additionally, a Hub is not responsible to prevent applications participating in exchanging structured information from causing inconsistencies in the information exchanged.  For example, an inconsistency could arise if an application removes from the anchor context's content a resource which is referenced by another resource.  The Hub may check [`[FHIR resource]-update`](#fhir-resource-update) requests for such inconsistencies and reject the request with an appropriate error message; however, it is not required to do so.  Additionally, a Hub may check for inconsistencies which it deems to be critical but not perform exhaustive validation. For example, a Hub could validate that the content in a `DiagnosticReport` anchor context always includes at least one primary imaging study.

Clients wishing to exchange structure information must:
1. Adhere to a FHIRcast event naming convention as follows: [FHIR resource]-[open|update|select|close]
2. Use a [`[FHIR resource]-open`](#fhir-resource-open) request to open a new resource which becomes the anchor context
3. Make a [`[FHIR resource]-update`](#fhir-resource-update) request when appropriate. The [`[FHIR resource]-update`](#fhir-resource-update) request contains a `Bundle` resource which is a collection of resources that are atomically processed by the Hub with the anchor context's content being adjusted appropriately
4. Maintain the current `versionId` of the anchor context provided by the Hub so that a subsequent [`[FHIR resource]-update`](#fhir-resource-update) request may provide the current `versionId`
5. Appropriately process [FHIR resource]-[open|update|select|close] events; note that a client may choose to ignore the contents of a [FHIR resource]-[open|update|select|close] event but should still track the `versionId` for subsequent use
6. If a [`[FHIR resource]-update`](#fhir-resource-update) request fails with the Hub, the client may issue a [`GET Context`](#get-context) request to the Hub in order to retrieve the current content in the anchor context and its current `versionId`
7. When using websockets, clients will now receive the current content (if any exists) of the anchor context (if one has been established) in response to the Subscribe request, see [`return of current content`](#websocket-return-of-current-content).  Clients that don't support the exchange of structured information may ignore the content of the Subscribe response payload.

### Example of Content Sharing in an Anchor Context
The below example uses a `DiagnosticReport` as the anchor context.  However, the pattern of the example holds when other FHIR resource types are the anchor context.

#### Diagnostic Report Workflow Example
When reporting applications integrate with PACS and/or RIS applications, a radiologist's (or other clinician's) workflow is centered on the final deliverable, a diagnostic report. In radiology the imaging study (exam) is an integral resource with the report referencing one or more imaging studies. Structured data, many times represented by an `Observation` resource, may also be captured as part of a report.  In addition to basic context synchronization, a diagnostic report centered workflow builds upon the basic FHIRcast operations to support near real-time exchange of structured information between applications participating in a diagnostic report context.  Also, the `DiagnosticReport` resource contains certain attributes (such as report status), that are useful to the PACS/RIS applications and don't exist in other types of FHIR resources (e.g. an ImagingStudy).  Participating applications may include clients such as reporting applications, PACS, EHRs, workflow orchestrators, and interactive AI applications.

Exchanged resources need not have an independent existence. For the purposes of a working session in FHIRcast, they are all "contained" in one resource (the `DiagnosticReport` anchor context). For example, a radiologist may use the PACS viewer to create a measurement. The PACS application sends this measurement as an `Observation` to the other subscribing applications for consideration. If the radiologist determines the measurement is useful in another application (and accurate), it may then become an `Observation` to be included in the diagnostic report. Only when that diagnostic report becomes an official signed document would that `Observation` possibly be maintained with an independent existence. Until that time, FHIR domain resources serve as a convenient means to transfer data within a FHIRcast context.

Structured information may be added, changed, or removed quite frequently during the lifetime of a Diagnostic Report context. Exchanged information is transitory and it is not required that the information exchanged during the collaboration is persisted. However, as required by their use cases, each participating application may choose to persist information in their own structures which may or may not be expressed as a FHIR resource. Even if stored in the form of a FHIR resource, the resource may or may not be stored in a system which provides access to the information through a FHIR server and associated FHIR operations (i.e., it may be persisted only in storage specific to a given application).

#### FHIRcast Notification Request Event Fields
Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The session topic given in the subscription request.
`hub.event` | Required | string | The event that triggered this notification (see list of supported events below).
`context` | Required | array | An array of named FHIR objects corresponding to the user's context after the given event has occurred. The contents of this field will vary depending on the event. However, in a diagnostic report centered workflow, this context will be a `DiagnosticReport` FHIR resource.  Attributes may be valued and other FHIR resources associated with the report may be embedded in the resource.

#### Diagnostic Report Centered Workflow Events
Operation | Description
--- | --- 
[`DiagnosticReport-open`](#fhir-resource-open) | This notification is used to begin a new report. This should be the first event and establishes the anchor context.
[`DiagnosticReport-update`](#fhir-resource-update) | This notification is used to make changes (updates) to the current report. These changes usually include adding/removing imaging studies and/or observations to the current report.
[`DiagnosticReport-select`](#fhir-resource-select) | This notification is sent to tell subscribers to make one or more images or observations visible (in focus), such as a measurement (or other finding).
[`DiagnosticReport-close`](#fhir-resource-close) | This notification is used to close the current diagnostic report anchor context with the current state of the exchanged content stored by subscribed applications as appropriate and cleared from these applications and the Hub. 

#### Example Use Case
A frequent scenario which illustrates a diagnostic report centered workflow involves an EHR, an image reading application, a reporting application, and an advanced quantification application.  The EHR, image reading application, and reporting application are authenticated and subscribed to the same topic using a FHIRcast Hub with the EHR establishing a patient context.  Using a reporting application, a clinical user decides to create a report by choosing an imaging study as the primary subject of the report.  The reporting application creates a report and then opens a diagnostic report context by posting a [`DiagnosticReport-open`](#fhir-resource-open) request to the Hub. On receiving the [`DiagnosticReport-open`](#fhir-resource-open) event from the Hub, an EHR decides not to react to this event noticing that the patient context has not changed. The image reading application responds to the event by opening the imaging study referenced in the `DiagnosticReport` anchor context.

The clinical user takes a measurement using the imaging reading application which then provides the reporting application this measurement by making a [`DiagnosticReport-update`](#fhir-resource-update) request to the Hub. The reporting application receives the measurement through a [`DiagnosticReport-update`](#fhir-resource-update) event from the Hub and adds this information to the report. As the clinical user continues the reporting process they select a measurement or other structured information in the reporting application, the reporting application may note this selection by posting a [`DiagnosticReport-select`](#fhir-resource-select) request to the Hub. Upon receiving the [`DiagnosticReport-select`](#fhir-resource-select) event the image reading application may navigate to the image on which this measurement was acquired.

At some point the image reading application (automatically or through user interaction) may determine that an advanced quantification application should be used and launches this application including the appropriate FHIRcast topic.  The advanced quantification application then requests the current context including any already exchanged structured information by making a [`GET Context`](#get-context) request to the Hub which returns the current context in the response.

Finally the clinical user closes the report in the reporting application. The reporting application makes a [DiagnosticReport-close](#fhir-resource-close) request. Upon receipt of the [DiagnosticReport-close](#fhir-resource-close) event both the imaging reading application and advanced quantification application close all relevant image studies.
 
# Normative Standard
## Description of Operations to Support Content Sharing
The following sections cover the additions to the FHIRcast STU2 standard required to support content sharing.  These operations would likely be distributed across existing structures in the existing FHIRcast standard.

### Subscription Response
Upon receiving subscription or unsubscription requests, the Hub SHALL respond to a subscription request with an HTTP 202 "Accepted" response. This indicates that the request was received and will now be verified by the Hub. When using websockets, the HTTP body of the response SHALL consist of a JSON object containing an element name of `hub.channel.endpoint` and a value of the WSS URL. The websocket WSS URL SHALL be cryptographically random, unique, and unguessable. If using webhooks, the Hub SHOULD perform verification of intent as soon as possible.

If a Hub refuses the request or finds any errors in the subscription request, an appropriate HTTP error response code (4xx or 5xx) SHALL be returned. In the event of an error, the Hub SHOULD return a description of the error in the response body as plain text, to be used by the client developer to understand the error. This is not meant to be shown to the end user. Hubs MAY decide to reject some subscription requests based on their own policies.

#### `websocket` Return of Current Content
If using websockets and a Hub accepts the subscription request, the Hub MAY return the current context of the topic's anchor context. The current content is returned in a `contexts` array in the JSON response body; if there is no current content the response SHALL contain an empty array ("[ ]").

If using webhooks the current context is not present in the subscription mechanism. After successful subscription, the subscriber may obtain the current context by making a [Get Context](#get-context) request.

### `webhook` vs `websocket`

A Hub SHALL support websockets and MAY support webhooks subscriptions. A subscriber specifies the preferred `hub.channel.type` of either `webhook` or `websocket` during creation of its subscription. Websockets are particularly useful if a subscriber is unable to host an accessible callback URL.

> Implementer feedback is solicited around the optionality and possible deprecation of webhooks.

#### `webhook` Subscription Request Example
In this example, the app asks to be notified of the `patient-open` and `patient-close` events.
```
POST https://hub.example.com
Host: hub.example.com
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/x-www-form-urlencoded

hub.channel.type=webhook&hub.callback=https%3A%2F%2Fapp.example.com%2Fsession%2Fcallback%2Fv7tfwuk17a&hub.mode=subscribe&hub.topic=fdb2f928-5546-4f52-87a0-0648e9ded065&hub.secret=shhh-this-is-a-secret&hub.events=patient-open,patient-close
```

#### `webhook` Subscription Response Example 
```
HTTP/1.1 202 Accepted
```

#### `websocket` Initial Subscription Request Example
In this example, the app creates an initial subscription and asks to be notified of the `patient-open` and `patient-close` events.
```
POST https://hub.example.com
Host: hub.example.com
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/x-www-form-urlencoded

hub.channel.type=websocket&hub.mode=subscribe&hub.topic=fdb2f928-5546-4f52-87a0-0648e9ded065&hub.events=patient-open,patient-close
```

#### `websocket` Subscription Response Example
```
HTTP/1.1 202 Accepted

{   
 "hub.channel.endpoint": wss://hub.example.com/ee30d3b9-1558-464f-a299-cbad6f8135de
}
```

 #### `websocket` Subscription Response Example with Return of the Current Context
```
HTTP/1.1 202 Accepted

{   
 "hub.channel.endpoint": wss://hub.example.com/ee30d3b9-1558-464f-a299-cbad6f8135de
  "contexts": [
     ... see example contexts in Get Context below
  ]
}
```
### Get Context
HTTP Method: GET<br>
Endpoint: base-hub-URL/{topic}<br>
Returns:<br>
This method returns an object containing the current context of the topic session. The current context is made up of one or more "top-level" contextual resource types such as an ImagingStudy or a DiagnosticReport. The `contextType` field identifies the resource type of the anchor context. For example, a DiagnosticReport-open event will create a new context with `contextType: "DiagnosticReport"`.

The Hub MAY return the content of the anchor context as elements without a "key" value in the context array.
 
```
[
  {
    "contextType": "DiagnosticReport",
    "context": [
      {
        "key": "report",
        "resource": {
          "resourceType": "DiagnosticReport",
          "id": "40012366",
          "meta": {
             "versionId": "2"
          },
          "status": "unknown",
          "subject": {
            "reference": "Patient/4b6de2b1f1c343888c2ffb751ccfb349",
          },
          "imagingStudy": [
            {
              "reference": "ImagingStudy/8i7tbu6fby5ftfbku6fniuf"
            }
          ]
        }
      },
      {
        "key": "patient",
        "resource": {
          "resourceType": "Patient",
          "id": "4b6de2b1f1c343888c2ffb751ccfb349",
          "identifier": [
            {
              "system": "urn:oid:1.2.840.114350",
              "value": "185444"
            }
          ]
        }
      },
      {
        "key": "study",
        "resource": {
          "resourceType": "ImagingStudy",
          "description": "CHEST XRAY",
          "started": "2010-01-30T23:00:00.000Z",
          "status": "available",
          "id": "8i7tbu6fby5ftfbku6fniuf",
          "identifier": [
            {
              "type": {
                "coding": [
                  {
                    "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                    "code": "ACSN"
                  }
                ]
              },
              "value": "342123458"
            }
          ],
          "patient": {
            "reference": "Patient/4b6de2b1f1c343888c2ffb751ccfb349"
          }
        }
      },
      {
        "resourceType": "Observation",
        "id": "4b6de2b1f1c343888c2ffb751ccfb349",
        "identifier": [
          {
            "system": "dcm:121151",
            "value": "L1"
          }
        ],
        "status": "preliminary",
        "issued": "2001-07-23T06:02:11-04:00",
        "derivedFrom": {
          "reference": "ImagingStudy/8i7tbu6fby5ftfbku6fniuf"
        },
        "component": [
          {
            "code": {
              "coding": [
                {
                  "system": "https://loinc.org",
                  "code": "21889-1",
                  "display": "Distance"
                }
              ]
            },
            "valueQuantity": {
              "system": "http://unitsofmeasure.org",
              "value": "30.3134634578934",
              "code": "mm",
              "unit": "mm"
            }
          }
        ],
        "category": {
          "system": "http://terminology.hl7.org/CodeSystem/observation-category",
          "code": "imaging",
          "display": "Imaging"
        },
        "code": {
          "system": "http://hl7.org/fhir/ValueSet/observation-codes",
          "code": "32449-1",
          "display": "Physical findings of Lung"
        }
      }
    ]
  }
]
```
 
(QUESTION: should the following events go into the FHIRcast event catalog?)
### [FHIR resource]-open
A `[FHIR resource]-open` request is posted to the Hub when a new or existing anchor context is opened by an application and established as the anchor context of a topic. The `context` field MUST contain at least one `Patient` resource and the anchor context resource.

When the Hub distributes a [FHIR resource]-open event, it associates a `versionId` with the anchor context.  Subscribed applications MUST submit this `versionId` in subsequent [`[FHIR resource]-update`](#fhir-resource-update) requests.

When a [`[FHIR resource]-open`](#fhir-resource-open) event is received by an application, the application should respond as is appropriate for its clinical use.  For example, an image reading application may want to respond to an event posted by a reporting application by opening the imaging study(ies) specified in the context. A reporting application may want to respond to an event posted by an image reading application by creating and opening a new report (or existing report as an addendum).

**[ToDo]** Make a request change to the FHIRcast STU2 standard that the id of the request in the below example (0d4c9998) should be the same as the id of the event distributed by the Hub so that the requestor can unambiguously record the versionId provided by the Hub.  At present the STU2 specification says: "Following an accepted context change request, the Hub MAY re-use this value in the broadcasted event notifications" (see Request Context Change Request - Request Context Change Parameters). This should be changed to "the Hub SHALL re-use".

#### Context
Key | Optionality | FHIR operation to generate context | Description
--- | --- | --- | ---
`[FHIR resource]`| REQUIRED | `[FHIR resource]/{id}?_elements=identifier` | Type of the [FHIR resource] being opened
`patient` | REQUIRED | `Patient/{id}?_elements=identifier` | FHIR Patient resource describing the patient associated with the diagnostic report currently in context.

##### [FHIR resource]-open Example Request
The following example shows a report being opened that contains a single primary study.  Note that the diagnostic report's `imagingStudy` and `subject` attributes have references to the imaging study and patient which are also in the open request.
```
{
  "timestamp": "2020-09-07T14:58:45.988Z",
  "id": "0d4c9998",
  "event": {
    "hub.topic": "DrXRay",
    "hub.event": "DiagnosticReport-open",
    "context": [
      {
        "key": "Report",
        "resource": {
          "resourceType": "DiagnosticReport",
          "id": "40012366",
          "status": "unknown",
          "subject": {
            "reference": "Patient/ewUbXT9RWEbSj5wPEdgRaBw3"
          },
          "imagingStudy": [
            {
              "reference": "ImagingStudy/8i7tbu6fby5ftfbku6fniuf"
            }
          ]
        }
      },
      {
        "key": "patient",
        "resource": {
          "resourceType": "Patient",
          "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
          "identifier": [
            {
              "system": "urn:oid:1.2.840.114350",
              "value": "185444"
            }
          ]
        }
      },
      {
        "key": "study",
        "resource": {
          "resourceType": "ImagingStudy",
          "description": "CHEST XRAY",
          "started": "2010-01-30T23:00:00.000Z",
          "status": "available",
          "id": "8i7tbu6fby5ftfbku6fniuf",
          "identifier": [
            {
              "type": {
                "coding": [
                  {
                    "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                    "code": "ACSN"
                  }
                ]
              },
              "value": "342123458"
            },
            {
              "system": "urn:dicom:uid",
              "value": "urn:oid:2.16.124.113543.6003.1154777499.38476.11982.4847614254"
            }
          ],
          "subject": {
            "reference": "Patient/ewUbXT9RWEbSj5wPEdgRaBw3"
          }
        }
      }
    ]
  }
}
```

##### [FHIR resource]-open Event Example
The event distributed by the Hub includes a meta section in the anchor context object (in this case a DiagnosticReport) with a `versionId` which will be used by subscribers to make subsequent [`[FHIR resource]-update`](#fhir-resource-update) requests.
```
{
  "timestamp": "2020-09-07T14:58:45.988Z",
  "id": "0d4c9998",
  "event": {
    "hub.topic": "DrXRay",
    "hub.event": "DiagnosticReport-open",
    "context": [
      {
        "key": "Report",
        "resource": {
          "resourceType": "DiagnosticReport",
          "id": "40012366",
          "meta": {
            "versionId": "0"
          },
          "status": "unknown",
          "subject": {
            "reference": "Patient/ewUbXT9RWEbSj5wPEdgRaBw3"
          },
          "imagingStudy": [
            {
              "reference": "ImagingStudy/8i7tbu6fby5ftfbku6fniuf"
            }
          ]
        }
      },
      {
        "key": "patient",
        "resource": {
          "resourceType": "Patient",
          "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
          "identifier": [
            {
              "system": "urn:oid:1.2.840.114350",
              "value": "185444"
            }
          ]
        }
      },
      {
        "key": "study",
        "resource": {
          "resourceType": "ImagingStudy",
          "description": "CHEST XRAY",
          "started": "2010-01-30T23:00:00.000Z",
          "status": "available",
          "id": "8i7tbu6fby5ftfbku6fniuf",
          "identifier": [
            {
              "type": {
                "coding": [
                  {
                    "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                    "code": "ACSN"
                  }
                ]
              },
              "value": "342123458"
            },
            {
              "system": "urn:dicom:uid",
              "value": "urn:oid:2.16.124.113543.6003.1154777499.38476.11982.4847614254"
            }
          ],
          "subject": {
            "reference": "Patient/ewUbXT9RWEbSj5wPEdgRaBw3"
          }
        }
      }
    ]
  }
}
```

### [FHIR resource]-update
A `[FHIR resource]-update` request will be posted to the Hub when an application desires a change be made to the current state of exchanged information or to add or remove a reference to a contained FHIR resource in the context of the current anchor context. One or more updates MAY occur while the anchor context is open.

The updates could include (but are not limited to) any of the following:
* adding, updating, or removing observations
* adding or removing other FHIR resources contained in the anchor context
* updating attributes of the anchor resource's attributes

The context MUST contain a `Bundle` in an updates key which contains one or more resources as entries in the Bundle.

The exchange of information is made using a transactional approach using change sets in the `[FHIR resource]-update` event (i.e. not the complete current state); therefore it is essential that applications interested in the current state of exchanged information process all events and process the events in the order in which they were successfully received by the Hub.  Each `[FHIR resource]-update` event posted to the Hub SHALL be processed atomically by the Hub (i.e., all entries in the request's `Bundle` should be processed prior to the Hub accepting another request).

The Hub plays a critical role in helping applications stay synchronized with the current state of exchanged information.  On receiving a `[FHIR resource]-update` request the Hub SHALL examine the `versionId` of the anchor context, which is in the resource's `meta` attribute.   The Hub SHALL compare the `versionId` of the incoming request with the `versionId` the Hub previously assigned to the anchor context (i.e, in the case of an anchor context of a Diagnostic Report the `versionId` assigned when the previous `DiagnosticReport-open` or `DiagnosticReport-update` request was processed). If the incoming `versionId` and last assigned `versionId`  do not match, the request SHALL be rejected and the Hub SHALL return a 4xx/5xx HTTP Status Code.
 
If the `versionId` values match, the Hub proceeds with processing each of the FHIR resources in the Bundle and SHALL process all Bundle entries in an atomic manner.  After updating its copy of the current state of exchanged information, the Hub SHALL assign a new `versionId` to the anchor context and use this new `versionId` in the `[FHIR resource]-update` event it forwards to subscribed applications.  The distributed update event SHALL contain a Bundle resource with the same Bundle `id` which was contained in the request. 

When a  `[FHIR resource]-update` event is received by an application, the application should respond as is appropriate for its clinical use.    For example, an image reading application may choose to ignore an observation describing a patient's blood pressure.  Since transactional change sets are used during information exchange, no problems are caused by applications deciding to ignore exchanged information not relevant to their function.  However, they should read and retain the `versionId` of the anchor context provided in the event for later use.

#### Context
Key | Optionality | Description
--- | --- | ---
`[FHIR resource]`| REQUIRED | Type of the [FHIR resource]. The `id` and `versionId` of the diagnostic report SHALL be provided; however, additional attributes MAY be present.
`updates`| REQUIRED | Contains one and only one FHIR Bundle resource with a `type` of `transaction`. The Bundle MUST contain 1 to n entries with each `entry` containing a FHIR resource with structured information that according to the `method` of the entry's `request` attribute should be added (POST), updated (PUT), or removed (DELETE) from the current state of exchanged structured information.

#### Supported Update Request Methods
Request Method | Operation
--- | ---
`PUT`| Replace/update an existing resource
`POST`| Add a new resource
`DELETE`| Remove an existing resource

##### DiagnosticReport-update Request Example for Adding an Imaging Study and Derived Observation
The following example shows adding an imaging study to the existing diagnostic report context.  The `context` holds the `id` and `versionId` of the diagnostic report as required in all  `DiagnosticReport-update` events.  The Bundle holds the addition (POST) of an imaging study and adds (POST) an observation derived from this study. 

```
{
  "timestamp": "2019-09-10T14:58:45.988Z",
  "id": "0d4c7776",
  "event": {
    "hub.topic": "DrXRay",
    "hub.event": "DiagnosticReport-update",
    "context": [
      {
        "key": "report",
        "resource": {
          "resourceType": "DiagnosticReport",
          "id": "40012366",
          "meta": {
            "versionId": "1"
          }
        }
      },
      {
        "key": "updates",
        "resource": {
          "resourceType": "Bundle",
          "id": "8i7tbu6fby5fuuey7133eh",
          "type": "transaction",
          "entry": [
            {
              "request": {
                "method": "POST"
              },
              "resource": {
                "resourceType": "ImagingStudy",
                "description": "CHEST XRAY",
                "started": "2010-02-14T01:10:00.000Z",
                "id": "3478116342",
                "identifier": [
                  {
                    "type": {
                      "coding": [
                        {
                          "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                          "code": "ACSN"
                        }
                      ]
                    },
                    "value": "3478116342"
                  },
                  {
                    "system": "urn:dicom:uid",
                    "value": "urn:oid:2.16.124.113543.6003.1154777499.30276.83661.3632298176"
                  }
                ]
              }
            },
            {
              "request": {
                "method": "POST"
              },
              "resource": {
                "resourceType": "Observation",
                "id": "435098234",
                "partOf": {
                  "reference": "ImagingStudy/3478116342"
                },
                "status": "preliminary",
                "category": {
                  "system": "http://terminology.hl7.org/CodeSystem/observation-category",
                  "code": "imaging",
                  "display": "Imaging"
                },
                "code": {
                  "coding": [
                    {
                      "system": "http://www.radlex.org",
                      "code": "RID49690",
                      "display": "simple cyst"
                    }
                  ] 
                },
                "issued": "2020-09-07T15:02:03.651Z"
              }
            }
          ]
        }
      }
    ]
  }
}
```
The HUB SHALL distribute a corresponding event to all applications currently subscribed to the topic. The Hub SHALL replace the `versionId` in the request with a new `versionId` generated and retained by the Hub.

### [FHIR resource]-select
A `[FHIR resource]-select` request will be made to the Hub when an application desires to indicate that one or more FHIR resources contained in the anchor context's content are to be made visible, in focus, or otherwise "selected". It is assumed that an FHIR resource (e.g., observation) with the specified `id` is contained in the current anchor context's content, the Hub MAY or MAY NOT provide validation of its presence.
 
This event allows other participating applications to adjust their UIs as appropriate.  For example, a reporting system may indicate that the user has selected a particular observation associated with a measurement value. After receiving this event an image reading application which created the measurement may wish to change its user display such that the image from which the measurement was acquired is visible.

If a resource is noted as selected, any other resource which had been selected is no longer selected (i.e. an implicit unselect of the previously selected resource).  Additionally, an application may indicate that all selections have been cleared by posting a `DiagnosticReport-select` with an empty `select` array.

#### Context
Key | Optionality | Description
--- | --- | ---
`report`| REQUIRED | FHIR DiagnosticReport resource in context is included for risk mitigation. The `id` and `versionId` of the diagnostic report SHALL be provided; however, additional attributes MAY be present.
`select`| REQUIRED | Contains zero or more references to selected resources. If a reference to a resource is present in the `select` array, there is an implicit unselect of any previously selected resource. If no resource references are present in the `select` array, this is an indication that any previously selected resource is now unselected.

##### [FHIR resource]-select Example
The following example shows the selection of a single Observation resource in an anchor context of a diagnostic report. 

```
{
  "timestamp": "2019-09-10T14:58:45.988Z",
  "id": "0e7ac18",
  "event": {
    "hub.topic": "DrXRay",
    "hub.event": "DiagnosticReport-select",
    "context": [
      {
        "key": "report",
        "resource": {
          "resourceType": "DiagnosticReport",
          "id": "40012366",
          "meta": {
            "versionId": "1"
          }
        }
      },
      {
        "select": [
          {
            "resourceType": "Observation",
            "id": "a67tbi5891trw123u6f9134"
          }
        ]
      }
    ]
  }
}
```

### [FHIR resource]-close
The `[FHIR resource]-close` event is posted to the Hub when an application desires to close the active anchor context centered workflow.

##### [FHIR resource]-close Example
This example closes a DiagnosticReport anchor context and sets the `status` of the diagnostic report to `final`.

```
{
  "timestamp": "2020-09-07T15:04:43.133Z",
  "id": "4441881",
  "event": {
    "hub.topic": "DrXRay",
    "hub.event": "DiagnosticReport-close",
    "context": [
      {
        "key": "Report",
        "resource": {
          "resourceType": "DiagnosticReport",
          "id": "40012366",
          "status": "final",
          "issued": "2020-09-07T15:04:42.134Z"
        }
      }
    ]
  }
}
```

