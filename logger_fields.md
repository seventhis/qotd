# Logger fields

A logger in this application is something that will produce a log entry.  There are two types of loggers; independent and dependent.  Independent loggers will write to the log at a regular frequency (with a variable mean delay).  Dependent loggers are associated with an endpoint call.  That is, every time the endpoint is called the logger will add a new message to the log.  Dependent logs therefore will vary with overall volume, while independnt loggers are consistent regardless of load volume.  Example of independent loggers are "heatbeat" logs which reguarly write a simple status out to the logs.

## Field types

**Pick** `pick`

The field will be replaced with a value from one of the randomly chosen (flat distribution) options.  

Example:
```JSON
"log": {
    "id": "log1",
    "name": "memory checksum",
    "template": "WARNING - computation option {{OPTION}} not valid.",
    "fields": {
        "OPTION": {
            "type": "pick",
            "options": [ "op1", "op2", "op3" ]
        }
    },
    "repeat": {
        "mean": 4000,
        "stdev": 1000,
        "min": 2000,
        "max": 8000
    }
}
```

The resulting log entry might look like:

```
WARNING - computation option op1 not valid.
```

**Pick Weighted** `pickWeighted`

Similar to the **pick** field type, a weighted pick allows the selection to be weighted.  Each option is assigned a weight.  The algorithm used to select the value for any given instance of the generated log is to create a slots for each option, with weight being the number of slots.  Then only one of the slots is selected at random.  The more slots associated with an option, the more likley it will be selected.

For example suppose the weighted options are:

```JSON
"options": [
    { "value": "GET", "weight": 8 },
    { "value": "PUT", "weight": 1 },
    { "value": "POST", "weight": 1 },
    { "value": "DELETE", "weight": 2 }
]
```

The total weight is 12.  This means that 8 times out of 12 the value `GET` will be selected.  One time out of 12 the `PUT` or `POST` value will be selected.

Example New Logger:

```JSON
{
    "name": "Start new repeating log warning about memory checksum every 2 seconds",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "memory checksum",
        "template": "CALL to external service returned an HTTP status {{ERROR}}. ",
        "fields": {
            "ERROR": {
                "type": "pickWeighted",
                "options": [
                    { "value": "400", "weight": 5 },
                    { "value": "401", "weight": 2 },
                    { "value": "404", "weight": 2 },
                    { "value": "500", "weight": 1 }
                ]
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**Normal Int** `normalInt`

Generates a integer value based off a normal curve.  The mean, std deviation, min and max are specified.

Example:

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "outside temp",
        "template": "The outside temperature is a nice {{temp}}F. ",
        "fields": {
            "temp": {
                "type": "normalInt",
                "mean": 70,
                "stdev": 10,
                "min": 60,
                "max": 90
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**Normal Float** `normalFloat`

Generates a numeric value from a normal distribution with the given mean and standard deviation.  The optional parameter `toFixed` is applied to the resulting value to fix the number of decimal places in the output.

Example:

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "outside temp",
        "template": "The outside temperature is a nice {{temp}}F. ",
        "fields": {
            "temp": {
                "type": "normalFloat",
                "mean": 70,
                "stdev": 10,
                "min": 60,
                "max": 90,
                "toFixed": 2
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**Random Int** `randomInt`

Generates a random integer from a flat distribution.  Parameters `min` and `max` set the range (inclusive).

Example:

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "customer id number",
        "template": "Failed accessing customer ID: {{custId}} ",
        "fields": {
            "custId": {
                "type": "randomInt",
                "min": 1000000,
                "max": 9999999
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```


**Random Float** `randomFloat`

Generates a random number with decimal places.  The params `min` and `max` set the range (inclusive).  The optional parameter `toFixed` will set the number of decimal places.


Example:

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "customer id number",
        "template": "Failed accessing customer ID: {{custId}} ",
        "fields": {
            "custId": {
                "type": "randomFloat",
                "min": 1000000,
                "max": 9999999
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**IP** `ip`

Generates a random IP address (i.e. 169.14.199.4).

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "threat warning",
        "template": "Unexpected service access from {{IP}} ",
        "fields": {
            "IP": {
                "type": "ip"
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**Email Address** `email`

Generates a generic email address.

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "new user",
        "template": "Created new user account for {{newEmail}}",
        "fields": {
            "newEmail": {
                "type": "email"
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**User Name** `userName`

Generates an example user name (i.e. name following typical name guidelines like no spaces, or control characters, etc.)

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "new user",
        "template": "User {{user}} requested access to a forbidden directory.",
        "fields": {
            "user": {
                "type": "userName"
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**URL** `url`

Generates a generic URL.

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "access url warning",
        "template": "Unable to access arbitrary URL: {{aurl}}.",
        "fields": {
            "aurl": {
                "type": "url"
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**MAC Address** `mac`

Generates a sample MAC address.

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "access MAC warning",
        "template": "Duplicate MAC value warning {{dupMac}}.",
        "fields": {
            "dupMac": {
                "type": "mac"
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**Domain Name** `domainName`

Generates a sample domain name.

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "Unavailable domain name",
        "template": "Who would have thought the domain name {{dn}} would be taken already.",
        "fields": {
            "dn": {
                "type": "domainName"
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**Word** `word`

Generates a nonsensical word (pig latin).  

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "Baby name",
        "template": "Only a billionare would name thier child {{name}}.",
        "fields": {
            "name": {
                "type": "word"
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**Sentence** `sentence`

Generates a sentence worth of nonsensical words.

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "strange comment",
        "template": "User submitted comment {{comment}}.",
        "fields": {
            "comment": {
                "type": "sentence"
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**Recent Date** `dateRecent`

Generates a recent date (something within days).  The format of the date time can be specified with a string that corresponds to the [Moment.js formats](https://momentjs.com/docs/#/displaying/format/).


```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "order fulfilled",
        "template": "Order fulfilled on {{dt}}.",
        "fields": {
            "dt": {
                "type": "dateRecent",
                "format": "dddd, MMMM Do YYYY, h:mm:ss a"
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**Formatted Current Timestamp** `formattedTimestamp`

Generates the current timestamp, and allows for formatting with [Moment.js formats](https://momentjs.com/docs/#/displaying/format/).

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "xaction completed",
        "template": "Transaction completed on {{now}}.",
        "fields": {
            "now": {
                "type": "dateRecent",
                "format": "dddd, MMMM Do YYYY, h:mm:ss a"
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**UTC Timestamp** `utcTimestamp`

Generated the current time in UTC format.

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "esperanto now",
        "template": "Transaction completed on {{now}}.",
        "fields": {
            "now": {
                "type": "utcTimestamp"
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**Filename** `fileName`

Generates a sample filename.

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "file missong",
        "template": "Unable to access file: {{fn}} in current directory.",
        "fields": {
            "now": {
                "type": "fileName"
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```

**Filetype** `fileType`

Generates a known file type extension (i.e. png, pdf, gif, pptx)

```JSON
{
    "name": "Sample log entry",
    "type": "add_logger",
    "service": "rating",
    "log": {
        "id": "log1",
        "name": "file ext not accepted",
        "template": "File types with {{ext}} are not accepted here.",
        "fields": {
            "now": {
                "type": "fileType"
            }
        },
        "repeat": {
            "mean": 3000,
            "stdev": 1000,
            "min": 1000,
            "max": 5000
        }
    }
}
```
