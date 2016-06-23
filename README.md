
The Cumul.io API makes it possible to interact with Cumul.io in a programmatic way, for example to create datasets and dashboards, push real-time data, develop your own custom integrations to data sources or retrieve sliced & diced, aggregated data from your datasets.

#Essentials

You can connect to Cumul.io via the WebSocket API or the REST API: these APIs support basic operations like creating, updating, retrieving, deleting, associating or dissociating metadata Entities. You can also receive push notifications and request real-time "data subscriptions": you request specific sliced & diced data and you'll receive notifications immediately in case any of the underlying datasets in your query is updated.

We also offer native SDKs (software development kits), currently for Node.js, PHP and .NET, that make it easier to use this API and deal with authentication or reconnection automatically.

Each API request should contain your API key and token, which are used to authorize the requested action or restrict result sets to only records you are allowed to retrieve. You can create a new API key and token via the Integration management page. All API calls are made over a secured connection via SSL (Secure WebSocket) to https://api.cumul.io on port 443.

##Example: connecting to Cumul.io

This running example uses the Node.js SDK. To get started, install the cumulio package from npm by running:
> npm install cumulio


Within your script, require this package and then connect to Cumul.io with your application API key and secret token. You can get a token for your application by dropping us an e-mail at support@cumul.io.
>// Include the Cumul.io SDK
>var Cumulio = require('cumulio');

>// Set API key and token
>var client = new Cumulio({
>  api_key: 'Your API key',
>  api_token: 'Your API token'
>});

##Example: pushing data points

This example uses the Create action on the Data resource to push new data points into an existing dataset. This will also immediately bring all dashboards based on this dataset up-to-date.

To load the dataset initially, you might want to upload a CSV file with the first set of rows manually, to create the column structure. You can retrieve the dataset ID from the URL.

>// Push 2 data points to a (pre-existing) dataset
client.create(
  'data',
  {
    securable_id: 'Your dataset id',
    data: [
      ['plaice', 2014, 2.1234, 751],
      ['plaice', 2015, 1.8765, 573]
    ]
  })
  .then(function() {
    console.log('Success!');
  })
  .catch(function(error) {
    console.error('API error:', error);
  })
  .finally(function() {
    client.close();
  });


You should keep open the connection as long as you're using the API, but don't forget to close it eventually at the end of your application (or your app won't stop running!).

Note that the SDK uses a promise-based API: if you're unsure you can learn more about this here.

##Example: creating datasets

In the previous example, we assumed a dataset (along with its column structure) had already been created. You can also create datasets and their metadata programmatically however. In this case, we first build a new Securable (this is a dataset or dashboard).

>client.create(
  'securable',
  {
    name: {
      en: 'Burrito statistics',
      nl: 'Burrito-statistieken'
    },
    description: {
      en: 'The number of burritos consumed per type of burrito.',
      nl: 'Het aantal geconsumeerde burrito's per type.'
    },
    type: 'dataset'
  })
  .then(function(dataset) {
    console.log('My new dataset id:', dataset.id);
  });
  
  
The name and description are localized strings: use them to provide translations of dataset or dashboard titles, columns and other metadata. The currently known languages are nl (Dutch), fr (French), en (English) and de (German). You can retrieve the most recent set languages from the Locale entity. Now we can add a first column to this dataset:

>client.create(
  'column',
  {
    name: {
      en: 'Type of burrito'
    },
    type: 'hierarchy',
    order: 0,
    color: '#45DE25'
  },
  [
    {
      role: 'Securable',
      id: 'New dataset id'
    }
  ]
).then( ... );


The second optional part immediately Associates (links) the column to the previously created dataset. You can specify any number of initial associations here. Alternatively, you could explicitly associate both new entities with:

> client.associate('column', 'My column id', {role: 'Securable', id: 'My dataset ID'});
or
client.associate('securable', 'My securable id', {role: 'Columns', id: 'My column ID'});

##Example: retrieving data

Just like the charts on your dashboards request data in real-time, you can interrogate your data within the platform as well. For example, to retrieve the total number of burritos savoured per type of burrito:

>client.query(
  {
    dimensions: [{
      column_id: 'Type of burrito id',
      dataset_id: 'My dataset id'
    }],
    measures: [{
      column_id: 'Number burritos savoured id',
      dataset_id: 'My dataset id'
    }]
  })
  .then(function(data) {
    console.log('Result set:', data.data);               // Prints: [['spicy', 1256],
                                                         //          ['sweet',  913]]
  });
  
Or a more elaborate example: let's record the weight per burrito as another numeric measure. We want the weights to be classified in 10 equal sized bins, for example [50, 100[ grams, [100, 150[ grams. Now we want to retrieve the top 5 weight classes by number of burritos savoured, but only for burritos that are of the type 'spicy':

>client.query(
  {
    dimensions: [
      {column_id: 'Burrito weight id', dataset_id: 'My dataset id',
         discretization: {type: 'linear', size: 10}}
    ],
    measures: [
      {column_id: 'Number burritos savoured id', dataset_id: 'My dataset id'}
    ],
    where: [
      {expression: '? = ?', parameters: [
        {column_id: 'Type of burrito id', dataset_id: 'My dataset id},
        'spicy'
      ]}
    ],
    order: [
      {column_id: 'Number burritos savoured id', dataset_id: 'My dataset id', order: 'desc']
    ],
    limit: {by: 5}
  })
  .then(function(data) {
    console.log('Result set:', data.data);               // Prints: [['[ 100 - 150 [', 125],
                                                         //          ['[ 150 - 200 [', 121]]
  });
  
# Analytics-as-a-Service

Cumul.io and Cumul.io dashboards can also be seamlessly embedded in third-party applications and products. The authorization and data flows are shown schematically in the diagram below:

Schematic overview of Analytics-as-a-Service
This type of embedding has 3 great advantages over regular dashboard sharing:

End-users of your application stay within your application all the time, and don't notice that they are interacting with a third-party service for data visualization & analytics.

Because of the token-based security, you can use any authentication mechanism (SAML, Kerberos, OpenID, OAuth2, password-based authentication, ...) you want. At the time a dashboard a viewed, you can use custom business logic to decide who can have access to which data.

If you use multitenanted datasets (eg. a relational database for all your clients), you can use data filters to ensure data is always pre-filtered server-side. Your end-clients will always only be able to query and view their own data.

##Example: embedding Cumul.io

Prerequisites: you already connected the necessary datasets via the UI, and have created an API key and token (see Essentials). You can find a full working example (including webserver) in our Github repository.

On each page request of an end-user of your platform, we will first create a temporary authorization token.

>var promise = client.create('authorization', {
  type: 'temporary',

  // Exhaustive list of datasets and dashboards to which to grant access
  securables: [
    '4db23218-1bd5-44f9-bd2a-7e6051d69166',
    'f335be80-b571-40a1-9eff-f642a135b826',
    '1d5db81a-3f88-4c17-bb4c-d796b2093dac
  ],

  // List of data filters to apply, so clients can only access their own data
  filters: [
    {
      clause: 'where',
      origin: 'global',
      securable_id: '4db23218-1bd5-44f9-bd2a-7e6051d69166',   // Demo data - United Widgets - Sales
      column_id: '3e2b2a5d-9221-4a70-bf26-dfb85be868b8',      // Product
      expression: '? = ?',
      value: 'Damflex'
    }
  ],

  // Expire the token after 5 minutes
  expiry: new Date(new Date().getTime() + 300*1000),

  // Force the screen mode to desktop (regardless of the iframe size) & language to Dutch
  screenmode: 'desktop',
  locale_id: 'nl'
});

In this case, we create a temporary authorization token that will cease working after 5 minutes. The token will have access to 1 dashboard and to the 2 datasets used in this dashboard. We only want this user to have access to the rows relating to the 'Damflex' product, so we set up a data filter. Data for this user will be filtered securely server-side.

Because we have knowledge of this user's preferences within our 3rd-party application, we set the default language to Dutch and force the dashboard to be shown in desktop mode.

Now, we use the freshly created token to embed the dashboard within our application.

promise.then(function(authorization) {
  // Generate the embedding url
  return client.iframe(dashboard, authorization);
})
.then(function(url) {
  // In reality, you might want to use a templating engine for this :-)
  response.end(
    ...
    '<iframe src="' + url + '"></iframe>' +
    ...
  );
});
For more embedding options, check the Authorization entity.
Actions

The API consists of 7 actions, which are applied to an Entity.

Create

Create a new instance of the requested resource type, for example: create a new User within an Organization, create a new access group, create new data rows, ...

client.create(entity, properties, [associations]);

entity          Lowercase entity name, eg. 'user' or 'grouprole'
properties      Object of entity properties, eg. {username: 'my username'}
associations    (optional) Initial associations to be set immediately after creation.
                Array of objects, eg. [{role: '...', id: '...'}]
Get

Retrieve one or more entities, possibily with (multiple levels of) nested associations, for example: retrieve a list of Users within your Organization along with the Datasets they own and all Columns of those Datasets. To retrieve data itself, refer to the Query action.

client.get(entity, query);

entity          Lowercase entity name, eg. 'user' or 'grouprole'
query           Metadata query object (not documented yet :-( )
Query

Retrieve sliced & diced, aggregated data from your datasets.

client.query(query);

query           Data query object (see examples above, full documentation is upcoming)
Update

Update the properties of a single specific entity, eg. update a column name or user password.

client.update(entity, id, properties);

entity          Lowercase entity name, eg. 'user' or 'grouprole'
id              Unique identifier (UUID) of the entity to update
properties      Object of updated entity properties, eg. {name: {en: 'new column name'}}
Delete

Delete a single specific entity.

client.delete(entity, id);

entity          Lowercase entity name, eg. 'user' or 'grouprole'
id              Unique identifier (UUID) of the entity to delete
Associate

Associate 2 entities to each other, eg. link a Column to a Dataset.

client.associate(entity, id, role);

entity          Lowercase entity name, eg. 'column'
id              Unique identifier (UUID) of the entity (eg. Column) to associate
role            Object describing role, eg. {role: 'Securable', id: 'Dataset id'}
Associations can be set in any direction. Specific user rights might be required for an association to succeed. Many associations are automatically set on creation as well, for example for Views or Modifications.

Dissociate

client.dissociate(entity, id, role);

entity          Lowercase entity name, eg. 'column'
id              Unique identifier (UUID) of the entity (eg. Column) to dissociate
role            Object describing role to dissociate, eg. {role: 'Securable', id: 'Dataset id'}
Remove an association between 2 entities. In case of 1-to-1 associations, "overwriting" an association will remove the previous association automatically.

