# OrgBook API (v3) Documentation

Documentation for OrgBook API usage.

## Table of contents
- [Common scenarios](#common-scenarios)
  - [Name search with autocomplete](#name-search-with-autocomplete)
  - [Basic organization search](#basic-organization-search)
  - [Faceted organization search](#faceted-organization-search)
  <!-- - [Organization credential verification](#organization-credential-verification)
  - [Credential issuer search](#credential-issuer-search)
  - [Credential type search](#credential-type-search) -->

## Common scenarios

This section goes through some of the most common use cases for the OrgBook API and offers some implementation guides for each of the scenarios.

_If you would like to see how some of these features are implemented in a real application, feel free to check out the  [Angular demo](/demo/README.md)._

### Name search with autocomplete

This is likely to be the most popular use case of the OrgBook API, giving you the ability to create a name lookup feature in your application for legally registered organizations, with autocomplete functionality.

The `/search/autocomplete` path has been created specifically for this.

`/search/autocomplete` takes a query parameter, `q`: a query string that will match the most closely related organization names in OrgBook. There are optional query parameters (described below) that can be provided, however default values are used when these query parameters are not included.

Example (using `'abc'` as the query string):

#### Request

```
/search/autocomplete?q=abc&inactive=false&revoked=false
```

The `inactive` query parameter denotes whether names of discontinued entities should be returned in the results. The `revoked` query parameter denotes whether entities with invalid credentials should be returned in the results. By default, only currently operating organizations with valid credentials are queried for.

#### Response

The API response will look something like this type definition:

_**Note:** the type names are arbitrary, the type definition is more important here._

```
interface AggregateAutocompleteResponse {
  total: number;
  first_index: number;
  last_index: number;
  results: AggregateAutocomplete[];
}
```
What you will be most interested in is the `results` field which is a list of closely matching entity names in OrgBook BC, and has the following type definition:

```
interface AggregateAutocomplete {
  type: string;
  value: string;
  score: number;
  topic_source_id: string;
  topic_type: string;
  credential_id: string;
}
```

Results are mostly returned in batches of up to 10 in descending search `score` order. More specific query strings will result in fewer results returned with higher search scores. Non-matching query strings will return empty results.
The main fields you will be interested are:
* `type`: The data field that autocomplete search is based on (example: 'name').
* `value`: The value of the data field that autocomplete search is based on (example: 'ABC LTD.').

#### Example

> Checkout this [example](https://stackblitz.com/edit/js-uum64f?file=index.js) on Stackblitz for a simple implementation using jQuery and plain HTML.

### Basic organization search

You will likely want more comprehensive information about an organization such as the type of organization, its business number, when it was registered, whether the organization is still active, even information about related organizations.

The `/search/topic` path is available for basic organization searches and returns data in the form of one or more verifiable credentials about the 'topic' of interest.

`/search/topic` takes a query parameter, `name`: a query string that will match the most closely related organization names in OrgBook. There are also optional query parameters (described below) that can be passed, and default values will be used when these query parameters are not included.

#### Request

Example (using `'abc'` as the query string):

```
/search/topic?name=abc&inactive=false&latest=true&revoked=false
```

The `inactive` query parameter denotes whether names of discontinued entities should be returned in the results. The `revoked` query parameter denotes whether entities with invalid credentials should be returned in the results. Since this path returns verifiable credentials, the `latest` query parameter denotes whether only the most recent credential(s) should be returned. By default, only currently operating organizations with the latest valid credentials are queried for.

#### Response

The API response will have the following interface definition:

```
interface TopicResponse {
  total: number;
  page_size: number;
  page: number;
  first_index: number;
  last_index: number;
  next: string;
  previous: string;
  results: CredentialTopicSearch[];
}
```

Results are returned in pages of up to 10 similarly matching entities to the name query (the number of results in the current response are indicated in the `page_size` field). The `total` field will indicate whether there are more results available than what's in the current page, and if so, `next` and `previous` URLs will be included in the response for accessing the next or previous pages of results, respectively. More specific query strings will result in fewer results and pages returned. Non-matching query strings will return empty pages.

The `results` field is a list of credentials for closely or exactly matching entities in OrgBook, and has the following type definition:

```
interface CredentialTopicSearch {
  id: number;
  create_timestamp: Date;
  update_timestamp: Date;
  effective_date: Date;
  inactive: boolean;
  latest: boolean;
  revoked: boolean;
  revoked_date: Date;
  credential_id: string;
  credential_set: CredentialSet;
  credential_type: CredentialType;
  attributes: CredentialAttribute[];
  names: CredentialName[];
  topic: CredentialTopicExt;
  related_topics: CredentialNamedTopic[];
}
```

The `inactive`, `latest`, `revoked` and `revoked_date` fields are self explanatory. `effective_date` indicates when the credential was activated.

_**Note:** `effective_date` is not the same as incorporation/registration date. It simply denotes the last time a credential was created for this entity in OrgBook._

Organization data is found in the `topic` field, which has the following type definition:

```
interface CredentialTopicExt {
  id: number;
  create_timestamp: Date;
  update_timestamp: Date;
  source_id: string;
  type: string;
  names: CredentialName[];
  local_name: CredentialName;
  remote_name: CredentialName;
  addresses: CredentialAddress[];
  attributes: TopicAttribute[];
}
```

You'll notice that it contains many of the same fields as the top-level response, with some additions, such as `source_id` (the unique registration number generated for the organization), `addresses` and `local_name`.

The `names` field contains the operating business names and has the following type definition (with `type` typically being 'entity_name'):

```
interface CredentialName {
  id: number;
  text: string;
  language: string;
  credential_id: string;
  type: string;
}
```

The `attributes` field is a list that likely contains a lot of information you are going to be interested is about the entity such as `registration_date`, `entity_name_effective`, `entity_status`, `entity_status_effective`, `entity_type`, `home_jurisdiction`, and `reason_description`. It has the following type definition:

```
interface TopicAttribute {
  id: number;
  type: string;
  format: string;
  value: string;
  credential_id: string;
  credential_type_id: string;
}
```

For example the following attribute list tells us that the organization is an active Sole Proprietorship within British Columbia:

```
"attributes": [
    {
        "id": ...,
        "type": "entity_status",
        "format": "category",
        "value": "ACT",
        "credential_id": ...,
        "credential_type_id": 1
    },
    {
        "id": ...,
        "type": "entity_type",
        "format": "category",
        "value": "SP",
        "credential_id": ...,
        "credential_type_id": 1
    },
    {
        "id": ...,
        "type": "home_jurisdiction",
        "format": "jurisdiction",
        "value": "BC",
        "credential_id": ...,
        "credential_type_id": 1
    }
]
```

#### Example

> Checkout this [example](https://stackblitz.com/edit/js-je6ckp?file=index.js) on Stackblitz for a simple implementation using jQuery and plain HTML.

### Faceted organization search

Faceted search augments basic search functionality by providing a mechanism to narrow down search results through the application of various filters, similar to what you would see on an e-commerce site like an online clothing store. _See [this](https://en.wikipedia.org/wiki/Faceted_search) Wikipedia article about faceted search._

The `/search/topic/facets` path has been created specifically for this and works exaclty the same as `search/topic` with the only difference being that the response is augmented with facets.

#### Request

Example (using `'abc'` as the query string):

```
/search/topic/facets?name=abc&inactive=false&latest=true&revoked=false
```

#### Response

The API response will have the following type definition, where `objects` is the same as the response body of `/search/topic` (see above):

```
interface TopicFacetsResponse {
  facets: TopicFacets;
  objects: TopicResponse;
}
```

The `facets` field has the following type definition, of which, `fields` are lists of objects that describe topics, grouped together by query parameters that can be added to a topic search.

```
interface TopicFacets {
  fields: TopicFacetFields;
  dates: TopicFacetDates;
  queries: TopicFacetQueries;
}
```

For example, the `category` facet field contains a number of objects with properties such as `entity_type`, `entity_status`, etc:

```
{
  category: [ ... ];
  credential_type_id: [ ... ];
  issuer_id: [ ... ];
}
```

Field objects have the following type definition:

```
export interface TopicFacetField {
    value: string;
    count: number;
    text: string;
}
```

The `value` field denotes the specific term to add to the `/search/topic` or `/search/topic/facets` paths to narrow a topic search down. The `count` field denotes how many topics are categorized by that term for the current topic search.

For example, suppose the following facets were returned from a topic search:

```
{
  category: [
    {
        "value": "entity_status::ACT",
        "count": 480
    },
    {
        "value": "entity_type::SP",
        "count": 302
    },
    {
        "value": "entity_type::BC",
        "count": 114
    },
    {
        "value": "entity_type::GP",
        "count": 49
    }
  ];
}
```

The current search results would contain all entity types that match or closely match an organization of iterest. Appending the field name as a query parameter and the value as the query parameter value to the topic search (for example, `?category=entity_type::GP`) would narrow down search results only to organizations that are General Partnerships.

<!-- ### Organization credential verification
### Credential issuer search
### Credential type search -->