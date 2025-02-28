---
description: >-
  This use case illustrates a demo configuration of an Extra Horizon (ExH)
  environment to support a polysomnography application.
---

# Polysomnography (PSG)

## Background

Polysomnography, also called a sleep study, is a comprehensive test used to diagnose sleep disorders. Polysomnography records your brain waves, the oxygen level in your blood, heart rate, and breathing, as well as eye and leg movements during the study. Polysomnography may be done at a sleep disorders unit within a hospital, a sleep center, or at home. A polysomnography records raw, multichannel time series data from channels such as EEG, EMG, ECG, PulseOx,… at a sampling rate of ±256Hz

![Example of a polysomnography record](<../.gitbook/assets/image (1) (1).png>)

## Objective <a href="objective" id="objective"></a>

The objective of this demo configuration is to ingest, process, store and annotate a data file generated by a medical device developed by the customer and make that data available to retrieve via the API and visualize it. The dataset is an [EDF file](https://en.wikipedia.org/wiki/European\_Data\_Format#:\~:text=European%20Data%20Format%20\(EDF\)%20is,independent%20of%20the%20acquisition%20system.) containing multiple hours of data.

![](<../.gitbook/assets/EDF file sequence.png>)

## ExtraHorizon introduction

The following [image](https://miro.com/app/board/o9J\_lC-X3Us=/?moveToWidget=3074457359254632780\&cot=14) shows a conceptual overview of Extra Horizon. On the left, you’ll find client interfacing applications such as a web front-end and mobile app. These clients connect to the customer's API.

![microservices](<../.gitbook/assets/Overview infrastructure.png>)

### Access management

Access management in Extra Horizon services relies on two modules: the [Authentication module ](../for-developers/services/authentication.md)and the [User module](../for-developers/services/user-service.md) for user management.

#### Security

Security is an important feature of every web-based application. Authorization is a requirement for all of our services. Authorization with ExtraHorizon services is done through the `auth service`, which will grant a token that can be used to validate requests to other services.

#### Users

The ExH User module manages standard user interaction like registering new users, activating prescriptions and resetting passwords. It also provides special features such as using _roles_ to manage user privileges and _groups_ to connect/aggregate any number of patients and staff members.

![Relationship between groups and users](<../.gitbook/assets/Polysomnography scheme 1 (1).png>)

As is illustrated above, the `user service` allows to control a user's privileges by medium of _roles_. Users can also be connected to a _group_, wherein privilege levels are controlled through _group roles_. Group roles determine a user's permissions within a group, provided they are enlisted to that group as a staff member.

### Data Management

Next to storing data in structured documents, the Data module allows configuring the structure and behavior of these documents using **data schemas**. With this feature, the behavior of the data can be programmed.

### Automation <a href="automation" id="automation"></a>

The [Task module](../for-developers/services/task-service.md) provides a way to execute code on demand by scheduling tasks. Tasks do not contain code themselves, but instead contain the information necessary to invoke code that is stored elsewhere, such as an AWS Lambda function. Tasks can either be queued to be executed as soon as possible, or scheduled for execution at a later moment.

### Communication <a href="communication" id="communication"></a>

The [Notification module](../for-developers/services/notification-service/) makes it easy to send notifications to users and checking if they've been received. The [Mail module](../for-developers/services/mail-service.md) allows more formal communication with users and works based on mail templates. E-mails and notifications can be sent in multiple languages by using the [Localization module](../for-developers/services/localization-service.md).

## Example configuration <a href="demo-process" id="demo-process"></a>

In this example, we illustrate a typical data-related use case: annotating biological signals. For this, an EDF file containing multiple biological signals and some metadata is provided.

### Data pathway <a href="the-data-pathway" id="the-data-pathway"></a>

The figure illustrates the data pathway from an EDF file until the data it contains is used for some purpose.

![pathway](<../.gitbook/assets/Polysomnography scheme 2.png>)

This pathway can be summarized as follows:

1. A new EDF file is stored on the files service (1a). As the file token is received with the response from the files service, this token is stored in a collection on the data service in the form of a JSON document (1b).
2. When a new document is uploaded to the data service, it automatically triggers a task in the task service.
3. The task service gets the file token as it is triggered by the data service, and the token is used to extract the corresponding file from the files service.
4. The task, a python script, opens the file and segments all the signals into 1-minute chunks. Each synchronized segment is then uploaded to a second collection on the data service in the form of a JSON document.
5. The JSON documents are ready to be used for further applications. For instance, the signals can be annotated.

### Storing files <a href="the-files-service" id="the-files-service"></a>

The file service allows uploading any kind of data, structured or unstructured. After the upload, a token will be returned to access at a later time.

#### Post a file <a href="post-a-file" id="post-a-file"></a>

To upload a new file to the files service, a post request is needed:

```python
filename = 'example.edf' 
files = {'file': (filename, open(filename, 'rb'), 'multipart/form-data')} 
data = {'tags[]': 'psg'} url = 'https://files.fibricheck.com/v1/' 
response = requests.post(url=url, files=files, data=data, auth=auth) 
print(response.text)

```

Example response:

```python
{
    "creatorId": "58ef448e4cedfd0005f8073e",
    "creationTimestamp": "2019-07-31T13:30:37.364Z",
    "updateTimestamp": "2019-07-31T13:30:37.364Z",
    "name": "example.edf",
    "mimetype": "multipart/form-data",
    "size": 4065374,
    "tokens": [
        {
            "accessLevel": "full",
            "token": "5d4197fd654ac34c881140bd-92252639-50eb-432a-afa5-9d7c15234249"
        }
    ],
    "tags": [
        "psg"
    ]
}
```

#### Get a file <a href="get-a-file" id="get-a-file"></a>

To retrieve a file from the files service, a get request is used

```python
url = 'https://files.fibricheck.com/v1/{token}/file/'
response = requests.get(url=url, auth=auth)
```

### The data service <a href="the-data-service" id="the-data-service"></a>

The data service is meant for storing structured data.

#### Schemas <a href="schemas" id="schemas"></a>

The data service can contain multiple collections to hold data in different structures. Each collection is characterized by a schema that specifies the data structure it accepts, but also the different statuses data can be into, how data transition between statuses, etc.

**Token schema**

The following shows the schema for the token collection. As can be seen, the data can contain two properties, `token` and `info`. In addition, A task is triggered at the creation of the document.

```python
{
    "name": "PSG tokens",
    "statuses": {
        "start": {}
    },
    "creationTransition": {
        "toStatus": "start",
        "type": "manual",
        "actions": [
            {
                "type": "task",
                "functionName": "psgChunkSignals"
            }
        ]
    },
    "transitions": [],
    "properties": {
        "token": {
            "type": "string"
        },
        "info": {
            "type": "object",
            "additionalProperties": {
                "type": "string"
            }
        }
    }
}
```

**Signals schema**

To store the 1-minute signals, another collection is used. Here is its schema.

```python
{
    "name": "PSG signals",
    "statuses": {
        "start": {}
    },
    "creationTransition": {
        "toStatus": "start",
        "type": "manual"
    },
    "transitions": [],
    "properties": {
        "signals": {
            "type": "object",
            "properties": {},
            "additionalProperties": {
                "type": "object",
                "properties": {
                    "device": {
                        "type": "object",
                        "properties": {
                            "manufacturer": {
                                "type": "string"
                            },
                            "model": {
                                "type": "string"
                            },
                            "os": {
                                "type": "string"
                            },
                            "type": {
                                "type": "string"
                            }
                        }
                    },
                    "data": {
                        "type": "array",
                        "items": {
                            "type": "number"
                        },
                        "queryable": false
                    },
                    "time": {
                        "type": "array",
                        "items": {
                            "type": "number"
                        },
                        "queryable": false
                    },
                    "annotations": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "annotatorId": {
                                    "type": "string"
                                },
                                "flags": {
                                    "type": "object",
                                    "additionalProperties": {
                                        "type": "object",
                                        "properties": {
                                            "start": {
                                                "type": "array",
                                                "items": {
                                                    "type": "number"
                                                }
                                            },
                                            "end": {
                                                "type": "array",
                                                "items": {
                                                    "type": "number"
                                                }
                                            },
                                            "proportion": {
                                                "type": "number"
                                            },
                                            "nRR": {
                                                "type": "number"
                                            }
                                        }
                                    }
                                },
                                "info": {
                                    "type": "object",
                                    "properties": {
                                        "completed": {
                                            "type": "boolean"
                                        },
                                        "valid": {
                                            "type": "boolean"
                                        },
                                        "bpm": {
                                            "type": "number"
                                        }
                                    }
                                },
                                "tags": {
                                    "type": "array",
                                    "items": {
                                        "type": "string"
                                    }
                                },
                                "peaks": {
                                    "type": "object",
                                    "additionalProperties": {
                                        "type": "array",
                                        "items": {
                                            "type": "number"
                                        }
                                    }
                                }
                            }
                        }
                    },
                    "info": {
                        "type": "object",
                        "properties": {
                            "nAnnotations": {
                                "type": "number"
                            },
                            "nCompletedAnnotations": {
                                "type": "number"
                            },
                            "timeOrigin": {
                                "type": "string",
                                "enum": [
                                    "time",
                                    "frequency"
                                ]
                            },
                            "nValidAnnotations": {
                                "type": "number"
                            }
                        }
                    },
                    "tags": {
                        "type": "array",
                        "items": {
                            "type": "string"
                        }
                    },
                    "frequency": {
                        "type": "number"
                    }
                }
            }
        },
        "info": {
            "type": "object",
            "additionalProperties": {
                "type": "string"
            }
        },
        "measurementTimestamp": {
            "type": "string",
            "format": "date-time"
        },
        "tags": {
            "type": "array",
            "items": {
                "type": "string"
            }
        },
        "claim": {
            "type": "object",
            "properties": {
                "userId": {
                    "type": "string"
                },
                "timestamp": {
                    "type": "number"
                }
            }
        }
    }
}
```

#### Statuses & transitions <a href="statuses-and-transitions" id="statuses-and-transitions"></a>

Different statuses can be defined in the schema. The documents can go from one status to another by means of transitions. A transition can be manual or automatic. A manual transition is triggered by sending a request, while an automatic transition is triggered by the data service itself when some specified set of conditions is fulfilled (field present, check on values, etc.).

#### Post a document <a href="post-a-document" id="post-a-document"></a>

To write a new document in a collection of the data service, a post request is used:

```python
data = { 'signals': { 'EKG': { 'data': [0,1,2,3]}}} 
url = 'https://data.fibricheck.com/v1/{schemaId}/documents/' 
response = requests.post(url=url, json=data, auth=auth)
```

#### Read a document <a href="read-a-document" id="read-a-document"></a>

To read a document from a specified collection of the data service:

```python
url = 'https://data.fibricheck.com/v1/{schemaId}/documents/?id=0123456789abcdef' 
response = requests.get(url=url, auth=auth)
```

## Example use case - Annotating

Once an EDF file is stored into the File module and consequently the signals have been uploaded into the Data module, the data is easily accessible for further applications. One example is to annotate the signals. The following figure shows an annotator tool developed by FibriCheck. The data are retrieved from the data service, annotated and annotations are stored back to the data service. A demo is available here

![Fibricheck Annotator Tool](https://pages.fibricheck.com/demo/psg/assets/img/annotator\_tool.da0dae2d.png)
