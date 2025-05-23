# ESGF-NG Data Challenge 01: Publication with Index Synchronization

## Round 2 Results

### Overview
As decided in the face-to-face meeting on November 2024, the first round of Data Challenge One includes using the ESGF core architecture to publish (POST) events from the esg-publisher to a cloud-deployed STAC Transaction API which sends events to a managed Kafka queue. The event contains information about the authenticity of the publisher along with STAC compliant CMIP6 dataset(s). A Kafka Consumer listening on a topic within the Kafka queue will ingest the datasets to a Globus search index. 

For this round of publication, each STAC Item will be run through CMIP6 pydantic models for schema-level validation.

### What is needed to run Data Challenge 1 Round 2?
* Same setup as [Round 1](https://github.com/ESGF/esgf-roadmap/blob/main/data-challenges/results/01-Publication.md)
* [Pydantic models](https://github.com/ESGF/esgf-playground-utils/blob/main/esgf_playground_utils/models/item.py) for schema validation

### How is Data Challenge 1 Round 2 run?
* Round 2 is run the same way as round 1 with the exception of adding a `try/except` [statement](https://github.com/esgf2-us/stac-transaction-api/blob/data-challenges-01-round-02/src/client.py#L121-L125) to catch any schema validation issues.
  * If there is an issue, the STAC API will log the error and return an `HTTP 400` to the client 
* Introducing known validation errors in some sample data, we generated a few examples of the return 400 ValidationError

```
Type error example
{
    "errors": [
        {
            "type": "list_type",
            "loc": ["properties", "activity_id"],
            "msg": "Input should be a valid list",
            "input": "C4MIP",
            "url": "https://errors.pydantic.dev/2.10/v/list_type",
        }
    ],
    "event_id": "b5ec1638-f043-4f45-943f-44ddf2125481",
    "item_id": "CMIP6.C4MIP.E3SM-Project.E3SM-1-1.hist-bgc.r1i1p1f1.fx.sftlf.gr.v20201015",
    "request_id": "e7e32108-9552-4488-bcdb-07a725e55908",
    "status_code": 400,
    "type": "validation_error",
}

Missing required value example
{
    "errors": [
        {
            "type": "string_type",
            "loc": ["properties", "variant_label"],
            "msg": "Input should be a valid string",
            "input": None,
            "url": "https://errors.pydantic.dev/2.10/v/string_type",
        }
    ],
    "event_id": "3c84ada7-a402-43fe-9b3c-d36463a3078a",
    "item_id": "CMIP6.C4MIP.E3SM-Project.E3SM-1-1.hist-bgc.r1i1p1f1.fx.sftlf.gr.v20201015",
    "request_id": "76dde31f-d848-4753-9a72-05fda87a150d",
    "status_code": 400,
    "type": "validation_error",
}
```

### Results of Data Challenge 1 Round 2
After running through the above steps, the validation script detected the following:
* All items existed in each index. In other words, the same number of items existed in the West as in the East
  * There was one duplicate, so each index reported 999 items after synchronization
* There were differences in property values for each item. Please see an example below.
  * `dictionary_item_added` means the property exists in the com but not in the ref
    * CEDA has `updated` and `created` properties that the West does not have
  * `values_changed` indicates properties that are not the same across indices
    * This error can essentially be ignored. CEDA is investigating the lower-case discrenpancy for the project value (CMIP6). The `links` property is generated by the discovery service and is expected to be different across indices.
```
2025-03-06 13:45:41 ref = https://data-challenge-02.api.stac.esgf-west.org/
2025-03-06 13:45:41 com = https://api.stac.esgf.ceda.ac.uk/collections
2025-03-06 13:45:42 differences found!
id=CMIP6.CMIP.E3SM-Project.E3SM-2-0.historical.r5i1p1f1.Amon.pr.gr.v20220901
dictionary_item_added: !!python/object/new:deepdiff.helper.SetOrdered
  state:
  - root['properties']['created']
  - root['properties']['updated']
  root['links'][0]['href']:
    com_value: https://api.stac.esgf.ceda.ac.uk/collections/cmip6/items/CMIP6.CMIP.E3SM-Project.E3SM-2-0.historical.r5i1p1f1.Amon.pr.gr.v20220901
    ref_value: https://data-challenge-01.api.stac.esgf-west.org/collections/CMIP6/items/CMIP6.CMIP.E3SM-Project.E3SM-2-0.historical.r5i1p1f1.Amon.pr.gr.v20220901
  root['links'][1]['href']:
    com_value: https://api.stac.esgf.ceda.ac.uk/collections/cmip6
    ref_value: https://data-challenge-01.api.stac.esgf-west.org/collections/CMIP6
  root['links'][2]['href']:
    com_value: https://api.stac.esgf.ceda.ac.uk/collections/cmip6
    ref_value: https://data-challenge-01.api.stac.esgf-west.org/collections/CMIP6
  root['links'][3]['href']:
    com_value: https://api.stac.esgf.ceda.ac.uk/
    ref_value: https://data-challenge-01.api.stac.esgf-west.org/
```

### To-do Items
