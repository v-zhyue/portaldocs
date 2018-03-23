
<a name="working-with-data"></a>
# Working with data

    
<a name="working-with-data-overview"></a>
## Overview

The Azure Portal UX provides unique data access challenges. Many blades and parts may be displayed at the same time, each of which instantiates a new `ViewModel` instance. Each `ViewModel` often needs access to the same or related data. To optimize for these interesting data-access patterns, extensions follow specific patterns that consist of code organization and data management.

The following sections discuss these patterns and how to apply them to an extension.

* **Code organization**

    * Organizing the extension source code into [areas](#areas)
    
* **Data management**

    * Shared data access using [data contexts](#data-contexts)

    * Using [data caches](#data-caches) to load and cache data

    * Memory management with [data views](#data-views)

All of these data concepts are used to achieve the goals of loading and updating extension data, in addition to efficient memory management. For more information about how the pieces fit together and how the resulting construct relates to the conventional MVVM pattern, see the [Data Architecture](https://auxdocs.blob.core.windows.net/media/DataArchitecture.pptx) video that discusses extension blades and parts.

**NOTE**: In this discussion, `<dir>` is the `SamplesExtension\Extension\` directory and  `<dirParent>`  is the `SamplesExtension\` directory. Links to the Dogfood environment are working copies of the samples that were made available with the SDK.

<a name="working-with-data-overview-areas"></a>
### Areas

From the perspective of code organization, an area can be defined as a project-level folder. It is of great assistance when data operations are segmented within the extension. `Areas` are a scheme for organizing and partitioning the source code of the extension. This makes it easier for teams to develop an extension in parallel. Areas help categorize related blades and parts, and each area also maintains the data that is required by its blades and parts.

Every area within an extension gets a distinct `DataContext` singleton, and it loads/caches/updates the data that is associated with the models that support the area's blades and parts.

An area is defined in the extension by using the following steps.

  1. Create a folder in the `Client\` directory of the project that contains the extension. The name of that folder is the name of the area, in addition to being the name of the `DataContext`.

  1. In the root of that folder, create a file for the `DataContext` and name it `<AreaName>Area.ts`, where `<AreaName>` is the name of the folder that was just created, as specified in [#developing-a-datacontext-for-an-area](#developing-a-datacontext-for-an-area). For example, the `DataContext` for the `Security` area in the sample is located at `\Client\Security\SecurityArea.ts`.  

A typical extension resembles the following image.

![alt-text](../media/portalfx-data-context/area.png "Extensions can host multiple areas")

<a name="working-with-data-overview-data-contexts"></a>
### Data contexts

For each area in an extension, there is a singleton `DataContext` instance that supports access to shared data for the blades and parts implemented in that area. Access to shared data means data loading, caching, refreshing and updating. Whenever multiple blades and parts make use of common server data, the `DataContext` is an ideal place to locate data logic for an extension [area](portalfx-extensions-glossary-data.md).

When a blade or part `ViewModel` is instantiated, its constructor is sent a reference to the `DataContext` singleton instance for the associated extension area.   The `ViewModel` accesses the data required by that blade or part in its constructor, as in the code located at `<dir>/Client/V1/MasterDetail/MasterDetailEdit/ViewModels/MasterViewModels.ts`.  The code is also in the following example.

```typescript

constructor(container: MsPortalFx.ViewModels.ContainerContract, initialState: any, dataContext: MasterDetailArea.DataContext) {
    super();

    this.title(ClientResources.masterDetailEditMasterBladeTitle);
    this.subtitle(ClientResources.masterDetailEditMasterBladeSubtitle);

    this._view = dataContext.websitesQuery.createView(container);
    
```

The benefits of centralizing data access in a singleton `DataContext` include the following.

1. Caching and Sharing

   The `DataContext` singleton instance will exist for the entire amount of time that the extension is loaded in the browser. Consequently, when a blade is opened, and therefore a new blade `ViewModel` is instantiated, data required by the new blade may have previously been loaded and cached in the `DataContext`, because it was required by some previously opened blade or rendered part.
  
   This cached data is available immediately, which optimizes rendering performance and improves [perceived responsiveness](portalfx-extensions-glossary-data.md). Also, no new **AJAX** calls are necessary to load the data for the newly-opened blade, which reduces server load and Cost of Goods Sold (COGS).

1. Consistency
  
   It is very common for multiple blades and parts to render the same data in levels of different detail, or with different presentation. Also, there are situations where such blades or parts are displayed on the screen simultaneously, or separated in time only by a single user navigation. 

   In these cases, the user expects to see all the blades and parts that describe the exact same state of the data. Using a shared `DataContext`  effectively achieves this consistency by allowing all the blades to use the single copy of the data that it contains.
  
1. Fresh data

   Users expect to see information that always reflects the current state of their data in the cloud. Another benefit of loading and caching data in a single location is that the cached data is regularly updated to accurately reflect the state of server data.

   For more information on refreshing data, see [portalfx-data-refreshingdata.md](portalfx-data-refreshingdata.md).

<a name="working-with-data-overview-data-caches"></a>
### Data caches
 
  The `DataCache` classes are a full-featured and convenient way to load and cache data required by blade and part `ViewModels`. The `DataCache` class objects in the API are `QueryCache`, `EntityCache` and the `EditScopeCache`. 

* QueryCache

  Loads data of type `Array<T>` according to an extension-specified `TQuery` type. `QueryCache` is useful for  list-like views like Grid, List, Tree, or Chart. For more information about the `QueryCache`, see [portalfx-data-caching.md#querycache](portalfx-data-caching.md#querycache). 
    
* EntityCache

  The `EntityCache` caches a single item, typically from the `QueryCache` object, as specified in [portalfx-data-caching.md#entitycache](portalfx-data-caching.md#entitycache).

* EditScopeCache

  The `EditScopeCache` class is less commonly used. It loads and manages instances of `EditScope`, which is a change-tracked, editable model for use in Forms, as specified in [portalfx-legacy-editscopes.md](portalfx-legacy-editscopes.md).
  
 * `DataCache` class methods

  The `DataCache` objects use the CRUD methods: Create, Replace, Update, and Delete. Commands for blades and parts often use these methods to modify server data. These commands are implemented in methods on the `DataContext` class, where each method can issue **AJAX** calls to the server and reflect data changes in associated DataCaches. For more information about sharnig  data across the parent and child blades, see  [portalfx-data-masterdetailsbrowse.md](portalfx-data-masterdetailsbrowse.md).

<a name="working-with-data-overview-data-views"></a>
### Data views

`DataCaches` in the `DataContext` of an `area` will exist in memory for as long as an extension is loaded, and consequently may support blades and parts that come and go as the user navigates in the Azure Portal.  Memory management becomes important, because memory overuse by many extensions or blades simultaneously can impact responsiveness. Consequently, each `DataCache` instance manages a set of [cache entries](portalfx-extensions-glossary-data.md), and the `DataCache` class includes  mechanisms to automatically manage the number of cache entries present at any given time. 

The data in the `DataCache`, and therefore the `DataContext`, is impacted by the needs of the `DataView` for the `ViewModel`. When a `ViewModel` calls the `fetch()` method for its `DataView`, the method implicitly forms a ref-count to a `DataCache` entry. This ref-count keeps the entry in the `DataCache` as long as the blade or part `ViewModel` has not been disposed by the extension.

The `DataCache` manages its size automatically, without explicit code in the extension source, by using the `DataView` to track the ref-counts to cache entries. When every blade or part `ViewModel` that holds a ref-count to a specific cache entry is disposed indirectly by using `DataView`, the `DataCache` can elect to discard the cache entry. 

<a name="working-with-data-overview-developing-a-datacontext-for-an-area"></a>
### Developing a DataContext for an area

The `DataContext` class does not specify the use of any single FX base class or interface. In practice, the members of a `DataContext` class typically include [data caches](#data-caches) and [data views](#data-views).

Typically, the `DataContext` associated with a particular area is instantiated from the `initialize()` method of `<dir>\Client\Program.ts`, which is the entry point of the extension, as in the following example.

```typescript

this.viewModelFactories.V1$MasterDetail().setDataContextFactory<typeof MasterDetailV1>(
    "./V1/MasterDetail/MasterDetailArea",
    (contextModule) => new contextModule.DataContext());

```

There is a single `DataContext` class per area. That class is named `<AreaName>Area.ts`, as in the code located at  `<dir>\Client\V1\MasterDetail\MasterDetailArea.ts` and in the following example.

```typescript

/**
* Context for data samples.
*/
export class DataContext {
   /**
    * This QueryCache will hold all the website data we get from the website controller.
    */
   public websitesQuery: QueryCache<WebsiteModel, WebsiteQueryParams>;

   /**
    * Provides a cache that will enable retrieving a single website.
    */
   public websiteEntities: EntityCache<WebsiteModel, number>;

   /**
    * Provides a cache for persisting edits against a website.
    */
   public editScopeCache: EditScopeCache<WebsiteModel, number>;

```



    
<a name="working-with-data-the-datacache-class"></a>
## The DataCache class

Multiple parts or services in an extension can rely on the same set of data. For queries, this may be a list of results, whereas for a details blade, it may be a single entity. In either case, it is critical to ensure that all parts that share a specific set of data perform the following actions.

1. Use a single HTTP request to access that data
1. Read from a single cache of data in memory
1. Release that data from memory when it is no longer required

These are all features that the `DataCache` class provides. The Azure Portal `MsPortalFx.Data` class objects are `QueryCache`, `EntityCache` and the `EditScopeCache`. The `DataCache` classes all share the same API. 

<!--TODO: Determine whether it is more accurate to say the following sentence.
The `DataCache` objects all share the same class within the API.   -->
  
They are a full-featured way of loading and caching data used by blade and part `ViewModels`.  The `QueryCache` object queries for a collection of data, whereas the `EntityCache` object loads an individual entity. 
 
<a name="working-with-data-the-datacache-class-using-the-datacache-class"></a>
### Using the DataCache class

Use the following steps to create a blade or part that uses the `DataCache` class. 

**NOTE**: In this discussion, `<dir>` is the `SamplesExtension\Extension\` directory and `<dirParent>` is the `SamplesExtension\` directory. Links to the Dogfood environment are working copies of the samples that were made available with the SDK.

<!-- TODO:  Determine whether there is a better sample that illustrates the points in the content, specifically the load and refresh points. -->

1. In a `DataContext`, the extension creates and configures `DataCache` instances. Configuring the instance specifies how to load data when it is missing from the cache and how to implicitly refresh cached data, to keep it consistent with the server state. The following `WebsiteQuery` example includes a constructor for a Website extension that creates a data cache. The code is also located at `<dir>Client\V1\Data\MasterDetailBrowse\MasterDetailBrowseData.ts`.

```typescript

this.websiteEntities = new MsPortalFx.Data.EntityCache<SamplesExtension.DataModels.WebsiteModel, number>({
    entityTypeName: SamplesExtension.DataModels.WebsiteModelType,
    sourceUri: MsPortalFx.Data.uriFormatter(Util.appendSessionId(DataShared.websiteByIdUri), true),
    findCachedEntity: {
        queryCache: this.websitesQuery,
        entityMatchesId: (website, id) => {
            return website.id() === id;
        }
    }
});

``` 

2. Each blade and part `ViewModel` creates a `DataView` in its constructor, so that it can load and refresh data for the blade or part.

```typescript

this._websiteEntityView = dataContext.websiteEntities.createView(container);

```

3. When the blade or part `ViewModel` receives its parameters in the `onInputsSet` method, the `ViewModel` calls the  `dataView.fetch()` method to load data.

```typescript

/**
 * Invoked when the blade's inputs change
 */   
public onInputsSet(inputs: Def.BrowseMasterListViewModel.InputsContract): MsPortalFx.Base.Promise {
    return this._websitesQueryView.fetch({ runningStatus: this.runningStatus.value() });
}

```
  
<a name="working-with-data-the-datacache-class-the-querycache"></a>
### The QueryCache

The `QueryCache` object is used to query for a collection of data or cache a list of items. It is useful for loading data for list-like views like Grid, List, Tree, or Chart. It takes a generic parameter for the type of object stored in its cache, and a type for the object that defines the query. It loads data of type `Array<T>` according to an extension-specified `TQuery` type.

The `QueryCache` parameters are as follows.

<!-- TODO: Determine whether the sourceURI and entityTypeName are required paramters.  Per the example given, the other parameters are optional. -->

**DataModels.WebsiteModel**: Optional. The model type for the website. This is usually auto-generated (see TypeMetadata section below).

**WebsiteQuery**: Optional. The type that defines the parameters for the query. This often will include sort order, filter parameters, paging data, etc.

**entityTypeName**: Provides the name of the metadata type. This is usually auto-generated (see TypeMetadata section below).

**sourceUri**: Provides the endpoint which will accept the HTTP request.

**poll**: Optional. A boolean field which determines whether entries in this cache should be refreshed on a timer.  You can use it in conjunction with property pollInterval can be used to override the default poll interval and pollPreservesClientOrdering can be used to preserve the client's existing record order when polling as opposed to using the order from the server response.

**supplyData**: Optional. Allows the extension to override the logic used to perform an AJAX request. Allows for making the AJAX call, and post-processing the data before it is placed into the cache.

In the example, when the `QueryCache` that contains the websites is created, two elements are specified.

1. **entityTypeName**: The name of the metadata type. The QueryCache needs to know the shape of the data contained in it, in order to handle the data appropriately. That information is specified with the entity type.

1. **sourceUri**: The endpoint that will accept the HTTP request. This function returns the URI to populate the cache. It is sent a set of parameters that are sent to a `fetch` call. In this case, `runningStatus` is the only parameter. Based on the presence of the `runningStatus` parameter, the URI is modified to query for the correct data.

<!-- TODO:  determine whether "presence" can be changed to "value". Did they mean true if present and false if absent, with false as the default value?  This sentence needs more information. -->

When all of these items are complete, the `QueryCache` is configured. It can be populated while Views that use the cache are being created, and the `fetch()` method is called. 

The sample located at  `<dir>Client\V1\Data\MasterDetailBrowse\MasterDetailBrowseData.ts` creates a `QueryCache` which polls the `sourceUri` endpoint on a timed interval. This code is also included in the following example.

```ts
export interface WebsiteQuery {
    runningStatus: boolean;
}

public websitesQuery = new MsPortalFx.Data.QueryCache<DataModels.WebsiteModel, WebsiteQuery>({
    entityTypeName: DataModels.WebsiteModelType,
    sourceUri: MsPortalFx.Base.Resources.getAppRelativeUri("/api/Websites?runningStatus={0}"),
    poll: true
});
```

<a name="working-with-data-the-datacache-class-entitycache"></a>
### EntityCache
 
The `EntityCache` object can be used to cache a single item.  It is useful for loading data into property views and single-record views. 

<!-- Determine whether a template class can be specified as an object in the content.  Otherwise, find a more definitive term. -->

It accepts data of type `T` according to some extension-specified `TId` type. This specifies the type for the object that defines the query, as in the `WebsiteQuery` example located at `<dir>Client/V1/MasterDetail/MasterDetailArea.ts`. This code is also included in the following example.

```typescript

this.websiteEntities = new EntityCache<WebsiteModel, number>({
    entityTypeName: SamplesExtension.DataModels.WebsiteModelType,

    // uriFormatter() is a function that helps you fill in the parameters passed by the fetch()
    // call into the URI used to query the backend. In this case websites are identified by a number
    // which uriFormatter() will fill into the id spot of this URI. Again this particular endpoint
    // requires the sessionId parameter as well but yours probably doesn't.
    sourceUri: FxData.uriFormatter(Util.appendSessionId(MsPortalFx.Base.Resources.getAppRelativeUri("/api/Websites/{id}")), true),

    // this property is how the EntityCache looks up a website from the QueryCache. This way we share the same
    // data object across multiple views and make sure updates are reflected across all blades at the same time
    findCachedEntity: {
        queryCache: this.websitesQuery,
        entityMatchesId: (website, id) => {
            return website.id() === id;
        }
    }
});

```

<!--TODO: Determine whether these parameters can be described as "
a generic parameter for the type of object stored in its cache, "  -->

When an EntityCache is instantiated, three elements are specified.

1. **entityTypeName**: The name of the metadata type. The EntityCache needs to know the shape of the data contained in it, in order to reason over the data.

1. **sourceUri**: The endpoint that will accept the HTTP request. This function returns the URI that the cache uses to get the data. It is sent a set of parameters that are sent to a `fetch` call. In this instance, the  `MsPortalFx.Data.uriFormatter()` helper method is used. It fills one or more parameters into the URI provided to it. In this instance there is only  one parameter, the `id` parameter, which is entered into the part of the URI containing `{id}`.

1. **findCachedEntity**: Optional. Allows the lookup of an entity from the `QueryCache`, instead of retrieving the data a second time from the server, which creates a second copy of the data on the client. This element also serves as a method, whose two properties are: 1) the `QueryCache` to use and 2) a function that, given an item from the QueryCache, will specify whether this is the object that was requested by the parameters to the `fetch()` call.
    
<a name="working-with-data-the-datacache-class-editscopecache"></a>
### EditScopeCache

The `EditScopeCache` class is less commonly used. It loads and manages instances of `EditScope`, which is a change-tracked, editable model for use in Forms, as specified in [portalfx-legacy-editscopes.md](portalfx-legacy-editscopes.md).  


    
<a name="working-with-data-using-dataviews"></a>
## Using DataViews

Data in a cache is presented to the `ViewModel` by using a `DataView`. 
The `DataCache` objects contain a `createView` method that is used to create the `DataView`. The data in the cache is processed by the `DataView` that it associates with a specific `ViewModel`. There can be many `DataViews` for a cache but there is a one-to-one correspondence between the `DataView` and the `ViewModel`. Creating a `DataView` does not result in a data load operation from the server.  When a `ViewModel` calls the `fetch()` method for its `DataView`, the method implicitly counts each time a reference is made to each `DataCache` entry. This ref-count keeps the entry in the `DataCache` as long as the blade or part `ViewModel` has not been disposed by the extension.


**NOTE**: In this discussion, `<dir>` is the `SamplesExtension\Extension\` directory and  `<dirParent>`  is the `SamplesExtension\` directory. Links to the Dogfood environment are working copies of the samples that were made available with the SDK.

<!--TODO: Remove the following placeholder sentence when it is explained in more detail. -->

**NOTE**: Because this discussion includes **AJAX**, **TypeScript**, and template classes, it does not strictly specify an object-property-method model.

The example located at `<dir>Client\V1\MasterDetail\MasterDetailBrowse\ViewModels\MasterViewModels.ts` demonstrates a `SaveItemCommand` class that uses the binding between a part and a command. This code is also included in the following `WebsiteQuery` example.

```ts
this._websitesQueryView = dataContext.masterDetailBrowseSample.websitesQuery.createView(container);
```

In the previous code, the `container` object acts as a lifetime object that  informs the cache when a given view is currently being displayed on the screen. This allows the Shell to make several adjustments for performance, including the following.

* Adjust polling interval when the part is not on the screen

* Automatically dispose of data when the blade that contains the part is closed

The server is only queried when the `fetch` operation of the view is invoked, as in the code located at 
`<dir>Client\V1\MasterDetail\MasterDetailBrowse\ViewModels\MasterViewModels.ts` and  in the following example.

```ts
public onInputsSet(inputs: any): MsPortalFx.Base.Promise {
    return this._websitesQueryView.fetch({ runningStatus: inputs.filterRunningStatus.value });
}
```

<a name="working-with-data-using-dataviews-observable-map-filter"></a>
### Observable map &amp; filter

In many cases, the developer may want to shape the extension data to fit the view to which it will be bound. Some cases where this is useful are as follows.

* Matching the contract of a control, like data points of a chart

* Adding a computed property to a model object

* Filtering data that is presented on the client

The recommended approach is to use the `map` and `filter` methods from the library that is located at [https://github.com/stevesanderson/knockout-projections](https://github.com/stevesanderson/knockout-projections).

The `runningStatus` is a filter that is applied to the query to allow several views to be created over a single cache. Each view can present a different subset of data from the cache, as in the code located at 
`<dir>Client\V1\MasterDetail\MasterDetailBrowse\ViewModels\MasterViewModels.ts` and in the following example.

```ts
public onInputsSet(inputs: any): MsPortalFx.Base.Promise {
    return this._websitesQueryView.fetch({ runningStatus: inputs.filterRunningStatus.value });
}
```

For more information about shaping and filtering your data, see [portalfx-data-projections.md](portalfx-data-projections.md).

    
<a name="working-with-data-master-details-browse"></a>
## Master details browse

This scenario describes how to share data across the parent blade and the child blades. In this scenario, the extension retrieves information from the server and displays this data across multiple blades. There are other scenarios in which the one-to-many relationship, or the summary-detail relationship, uses grids to display information from `QueryCache`-`EntityCache` cache objects
in parent-child blades.

In the example code, the information to display in the parent blade is a list of websites. It displays this information in a grid. When the user activates a website, a child blade is opened, and displays detailed information about the activated website. The child blade  is also dependent on the parent blade for specific resources.

In this discussion, `<dir>` is the `SamplesExtension\Extension\` directory and  `<dirParent>`  is the `SamplesExtension\` directory. Links to the Dogfood environment are working copies of the samples that were made available with the SDK. The code for this example is located at:
`<dir>\Client\V1\MasterDetail\MasterDetailArea.ts`
`<dir>\Client\V1\MasterDetail\MasterDetailBrowse\MasterDetailBrowse.pdl`
`<dir>\Client\V1\Data\MasterDetailBrowse\MasterDetailBrowseData.ts`
`<dir>\Client\V1\asterDetail\MasterDetailBrowse\ViewModels\DetailViewModels.ts`
`<dir>\Client\V1\MasterDetail\MasterDetailBrowse\ViewModels\MasterViewModels.ts`

The `QueryCache` and the `EntityCache` are the two caches that are used in this scenario. The `QueryCache` caches a list of items as specified in [portalfx-data-caching.md#querycache](portalfx-data-caching.md#querycache). The `EntityCache` caches a single item, as specified in [portalfx-data-caching.md#entitycache](portalfx-data-caching.md#entitycache).

The server data is cached using one of the two caches, and then uses that cache to display the websites across the two blades.  The data for both blades is located in the cache, therefore the server is not queried again when it is time to open the second blade. When the data in the cache is updated, that update is displayed across all blades at the same time. Consequently, the Portal always presents a consistent view of the data.

How the extension code uses the sample code is specified in [#the-masterdetail-area-and-datacontext](#the-masterdetail-area-and-datacontext). 
The two primary pieces of code are the following.
* [Implementing the master view](#implementing-the-master-view)
* [Implementing the detail view](#implementing-the-detail-view)


<a name="working-with-data-master-details-browse-the-masterdetail-area-and-datacontext"></a>
### The MasterDetail Area and DataContext

The Portal uses an `Area` to hold the cache and other data objects that are shared across multiple blades. The code for the area is located in its own folder, as specified in [portalfx-data-overview.md#areas](portalfx-data-overview.md#areas). In this example, the area folder is named `MasterDetail` and it is located in the `Client` folder of the extension. 

 Inside the folder, there is a TypeSscript file whose name is a combination of the name of the area and the word 'Area'. The file is named  `MasterDetailArea.ts` and is located at `<dir>Client/V1/MasterDetail/MasterDetailArea.ts`. This code is also included in the following example.

```typescript

this.websitesQuery = new QueryCache<WebsiteModel, WebsiteQueryParams>({
    entityTypeName: SamplesExtension.DataModels.WebsiteModelType,

    // when fetch() is called on the cache the params will be passed to this function and it 
    // should return the right URI for getting the data
    sourceUri: (params: WebsiteQueryParams): string => {
        let uri = MsPortalFx.Base.Resources.getAppRelativeUri("/api/Websites");

        // if runningStatus is null we should get all websites
        // if a value was provided we should get only websites with that running status
        if (params.runningStatus !== null) {
            uri += "?$filter=Running eq " + params.runningStatus;
        }

        // this particular controller expects a sessionId as well but this is not the common case.
        // Unless your controller also requires a sessionId this can be omitted
        return Util.appendSessionId(uri);
    }
});

```

This file contains the `DataContext` class, which is the class that will be sent to all the `ViewModels` associated with the area.  The `DataContext` also contains an `EditScopeCache` which is used in the master detail edit scenario that is located at [portalfx-forms-construction.md](portalfx-forms-construction.md). This code is also included in the following example.

 <!--TODO:  Locate the gitHub copies of the samples. -->

<!-- determine whether an "as specified in portalfx-forms-editscope*" is relevant here.  -->

If this is a new area for the extension, edit the  `Program.ts` file to create the `DataContext` when the extension is loaded. The SDK edition of the `Program.ts` file is located at `<dir>\Client\Program.ts`. For the extension that is being built, find the `initializeDataContexts` method and then use the `setDataContextFactory` method to set the `DataContext`.

<a name="working-with-data-master-details-browse-implementing-the-master-view"></a>
### Implementing the master view

The master view is used to display the data in the caches. The advantage of using views is that they allow the `QueryCache` to handle the caching, refreshing, and evicting of data.

1. Make sure that the PDL that defines the blades specifies the  `Area` so that the `ViewModels` receive the `DataContext`. In the `<Definition>` tag at the top of the PDL file, include an `Area` attribute whose value corresponds to the name of the area that is being built, as in the example located at `<dir>Client/V1/MasterDetail/MasterDetailBrowse/MasterDetailBrowse.pdl`. This code is also included in the following example.

   ```xml

<Definition xmlns="http://schemas.microsoft.com/aux/2013/pdl"
Area="V1/MasterDetail">

``` 

    The `ViewModel` for the list of websites is located in `<dir>\Client\V1\MasterDetail\MasterDetailBrowse\ViewModels\MasterViewModels.ts`. 

1. Create a view on the `QueryCache`, as in the following example.

     ```typescript

this._websitesQueryView = dataContext.websitesQuery.createView(container);

``` 

   The view is the `fetch()` method that is called to populate the `QueryCache`, and allows the items that are returned by the fetch call to be viewed. 

   **NOTE**:  There may be multiple views over the same `QueryCache`. This  occurs when multiple blades are displayed on the screen at the same time, all of which are using data from the same cache. 

   There are two controls on this blade, both of which use the view that was just created: a grid and the `OptionGroup` control.  

   * The grid displays the data in the `QueryCache`, as specified in [portalfx-data-dataviews.md](portalfx-data-dataviews.md) and in [#fetching-data-for-the-grid](#fetching-data-for-the-grid).

   * The `OptionGroup` control allows the user to select whether to display websites that are in a running state, websites in a stopped state or display both types of sites. The display created by the `OptionGroup` control is specified in [#the-optiongroup-control](#the-optiongroup-control).

<a name="working-with-data-master-details-browse-implementing-the-master-view-fetching-data-for-the-grid"></a>
#### Fetching data for the grid

The observable `items` array of the view is sent to the grid constructor as the `items` parameter, as in the following example.

```typescript

this.grid = new Grid.ViewModel<WebsiteModel, number>(this._lifetime, this._websitesQueryView.items, extensions, extensionsOptions);

```

The `fetch()` command has not yet been issued on the QueryCache. When the command is issued, the view's `items` array will be observably updated, which populates the grid with the results. This occurs by calling the  `fetch()` method on the blade's `onInputsSet()`, which returns the promise shown in the following example.

```typescript

/**
 * Invoked when the blade's inputs change
 */   
public onInputsSet(inputs: Def.BrowseMasterListViewModel.InputsContract): MsPortalFx.Base.Promise {
    return this._websitesQueryView.fetch({ runningStatus: this.runningStatus.value() });
}

```

This will populate the `QueryCache` with items from the server and display them in the grid.

<a name="working-with-data-master-details-browse-implementing-the-master-view-the-optiongroup-control"></a>
#### The OptionGroup control

The  control is initialized, and the extension then subscribes to its value property, as in the following example.

```typescript

this.runningStatus.value.subscribe(this._lifetime, (newValue) => {
    this.grid.loading(true);
    this._websitesQueryView.fetch({ runningStatus: newValue })
        .finally(() => {
            this.grid.loading(false);
        });
});

```

In the subscription, the extension performs the following actions.

1. Put the grid in a loading mode. It will remain in this mode until all of the data is retrieved.
1. Request the new data by calling the `fetch()` method on the data view with new parameters.
1. When the `fetch()` method completes, take the grid out of loading mode.

There is no need to get the results of the fetch and replace the items in the grid because the grid's `items` array has been pointed to the `items` array of the view. The view will update its `items` array as soon as the fetch is complete.

The rest of the code demonstrates that the grid has been configured to activate any of the websites when they are clicked. The `id` of the website that is activated is sent to the details child blade as an input.  For more information about the details child blade, see [#implementing-the-detail-view](#implementing-the-detail-view).

<a name="working-with-data-master-details-browse-implementing-the-detail-view"></a>
### Implementing the detail view

The detail view uses the `EntityCache` that was associated with the  `QueryCache` from the `DataContext` to display the details of a website. 

1. The child blade creates a view on the `EntityCache`, as in the following code.

```typescript

this._websiteEntityView = dataContext.websiteEntities.createView(container);

```

1.  In the `onInputsSet` method, the `fetch` method is called, with the ID of the website to display as a parameter, as in the following example. 

```typescript

/**
* Invoked when the blade's inputs change.
*/
public onInputsSet(inputs: Def.BrowseDetailViewModel.InputsContract): MsPortalFx.Base.Promise {
    return this._websiteEntityView.fetch(inputs.currentItemId);
}

```

1. When the fetch is completed, the data is available in the view's `item` property. This blade uses the `text` data-binding in its HTML template to show the name, id and running status of the website. 

    gitdown": "include-file", "file": "../templates/portalfx-data-lifetime.md"}

    gitdown": "include-file", "file": "../templates/portalfx-data-loadingdata.md"}



    gitdown": "include-file", "file": "../templates/portalfx-data-projections.md"}
  
    gitdown": "include-file", "file": "../templates/portalfx-data-refreshingdata.md"}

    gitdown": "include-file", "file": "../templates/portalfx-data-virtualizedgriddata.md"}

    gitdown": "include-file", "file": "../templates/portalfx-data-typemetadata.md"}

<a name="advanced-data-topics"></a>
# Advanced data topics

    gitdown": "include-file", "file": "../templates/portalfx-data-atomization.md"}

   ## Best Practices 

<a name="advanced-data-topics-use-querycache-and-entitycache"></a>
### Use QueryCache and EntityCache

When performing data access from your view models, it may be tempting to make data calls directly from the `onInputsSet` function. By using the `QueryCache` and `EntityCache` objects from the `DataCache` class, you can control access to data through a single component. A single ref-counted cache can hold data across your entire extension.  This has the following benefits.

* Reduced memory consumption
* Lazy loading of data
* Less calls out to the network
* Consistent UX for views over the same data.

**NOTE**: Developers should use the `DataCache` objects `QueryCache` and `EntityCache` for data access. These classes provide advanced caching and ref-counting. Internally, these make use of Data.Loader and Data.DataSet (which will be made FX-internal in the future).

To learn more, visit [portalfx-data-caching.md#configuring-the-data-cache](portalfx-data-caching.md#configuring-the-data-cache).



<a name="advanced-data-topics-avoid-unnecessary-data-reloading"></a>
### Avoid unnecessary data reloading

As users navigate through the Ibiza UX, they will frequently revisit often-used resources within a short period of time.
They might visit their favorite Website Blade, browse to see their Subscription details, and then return to configure/monitor their
favorite Website. In such scenarios, ideally, the user would not have to wait through loading indicators while Website data reloads.

To optimize for this scenario, use the `extendEntryLifetimes` option that is available on the `QueryCache` object and the `EntityCache` object.

```ts

public websitesQuery = new MsPortalFx.Data.QueryCache<SamplesExtension.DataModels.WebsiteModel, any>({
    entityTypeName: SamplesExtension.DataModels.WebsiteModelType,
    sourceUri: MsPortalFx.Data.uriFormatter(Shared.websitesControllerUri),
    supplyData: (method, uri, headers, data) => {
        // ...
    },
    extendEntryLifetimes: true
});

```

The cache objects contain numerous cache entries, each of which are ref-counted based on not-disposed instances of QueryView/EntityView. When a user closes a blade, typically a cache entry in the corresponding cache object will be removed, because all QueryView/EntityView instances will have been disposed. In the scenario where the user revisits the Website blade, the corresponding cache entry will have to be reloaded via an **AJAX** call, and the user will be subjected to loading indicators on
the blade and its parts.

With `extendEntryLifetimes`, unreferenced cache entries will be retained for some amount of time, so when a corresponding blade is reopened, data for the blade and its parts will already be loaded and cached.  Here, calls to `this._view.fetch()` from a blade or part `ViewModel` will return a resolved Promise, and the user will not see long-running loading indicators.

**NOTE**:  The time that unreferenced cache entries are retained in QueryCache/EntityCache is controlled centrally by the FX
 and the timeout will be tuned based on telemetry to maximize cache efficiency across extensions.)

For your scenario to make use of `extendEntryLifetimes`, it is **very important** that you take steps to keep your client-side
QueryCache/EntityCache data caches **consistent with server data**.
See [Reflecting server data changes on the client](portalfx-data-configuringdatacache.md) for details.


   ## Frequently asked questions

<a name="advanced-data-topics-"></a>
### 

* * * 
   
   ## Glossary

 This section contains a glossary of terms and acronyms that are used in this document. For common computing terms, see [https://techterms.com/](https://techterms.com/). For common acronyms, see [https://www.acronymfinder.com](https://www.acronymfinder.com).

| Term                 | Meaning |
| ---                  | --- |
| area                 | A project-level folder that is used to organize and partition the source code by categorizing related blades and parts by their data requirements.  |
| ARM | Azure Resource Manager | 
| cache entries | | 
| COGS |   Cost of Goods Sold |
| CORS | Cross-origin resource sharing. A mechanism that defines a way in which a browser and server can interact to determine whether or not it is safe to allow the cross-origin requests to be served.  For example, restricted resources, like  fonts,  may be requested from domains outside the domain from other resources are served. It may use additional HTTP headers to allow users to gain permission to access selected resources from a server on a different origin. | 
| CRUD | Create, Replace, Update, Delete |
| JSON Web Token |  JSON-based token that asserts information between the server and the client.  For example, a JWT could assert that the client user has the claim "logged in as admin".  | 
| JWT token | JSON Web Token |
| lifetime | The amount of time an object exists in memory between instantiation and destruction.  It may be automatically destroyed by when a parent object is deallocated, or its existence may be managed by a lifetime manager object or a child lifetime manager object. |
| lifetime object | An object that informs the `DataCache` when a specific `DataView` is currently being displayed on the screen. This allows the Shell to make performance adjustments. | 
| polling | The process that determines which client-side data should be automatically refreshed. |
| perceived responsiveness | Responsiveness that is immediately apparent to an average user. |
| reactor | |