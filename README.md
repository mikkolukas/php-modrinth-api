# Aternos/php-modrinth-api
An API client for the Modrinth API written in PHP. This client is a combination of
code generated by OpenAPI Generator and some wrappers around it to improve the
usability.

The generated code can be found in `lib/Api` and `lib/Model`. It is recommended
to use the Wrappers in `lib/Client` instead of the generated code.

## Installation
Install the package via composer:
```bash
composer require aternos/modrinth-api
```

## Usage

The main entry point for the API is the `ModrinthAPIClient` class.
```php
<?php
use Aternos\ModrinthApi\Client\ModrinthAPIClient;

// create an API client. This is the main entry point for the API
$modrinthClient = new ModrinthAPIClient();

// set a user agent (recommended)
$modrinthClient->setUserAgent('aternos/php-modrinth-api-example');

// set an api token (optional)
$modrinthClient->setApiToken("api-token");
```
The API Token is only required for non-public requests but if it is provided, it will be used for all requests.

## Paginated Lists
Some methods return a paginated list which contains a list of results on the current page and methods to
navigate to the next and previous page. The paginated list implements `Iterator`, `ArrayAccess` and `Countable` so
you can use it like an array. It also has a `getResults()` method which returns the underlying array of results.

### Searching for Projects
```php
$projects = $modrinthClient->searchProjects();

foreach ($project as $project) {
    // like most other methods, this method returns a wrapper
    // you can use the getData() method to get the project data
    echo $project->getData()->getTitle() . PHP_EOL;
}

$projects = $projects->getNextPage();

foreach ($projects as $project) {
    echo $project->getData()->getTitle() . PHP_EOL;
}
```

### Search for Projects with Options
You can apply filters and change the sort order when searching for projects.
All options are optional and can be combined.
```php
use \Aternos\ModrinthApi\Client\Options\ProjectSearchOptions;
use \Aternos\ModrinthApi\Client\Options\SearchIndex;

$options = new ProjectSearchOptions();
$options->setQuery("mclogs");
$options->setSearchIndex(SearchIndex::UPDATED);
$projects = $modrinthClient->getProjects($options);
```

#### Facets
One way to filter search results are facets. Facets specify which loader, category, game version or license to filter for.
You need to provide exactly one AND group which can consist of multiple OR groups: 
```php
use \Aternos\ModrinthApi\Client\Options\Facets\Facet;
use \Aternos\ModrinthApi\Client\Options\Facets\FacetType;
use \Aternos\ModrinthApi\Client\Options\Facets\FacetORGroup;
use \Aternos\ModrinthApi\Client\Options\Facets\FacetANDGroup;

$facetOrGroup = new FacetORGroup();
$facetOrGroup->addFacet(new Facet(FacetType::VERSIONS, "1.20.1"))
             ->addFacet(new Facet(FacetType::VERSIONS, "1.20"));

// or add multiple facets at once:
$facetOrGroup->addFacets(
    new Facet(FacetType::VERSIONS, "1.19.4"),
    new Facet(FacetType::VERSIONS, "1.19.3")
);

// alternatively if all facets have the same type:
$facetOrGroup->addFacets(FacetType::VERSIONS, "1.19.2", "1.19.1", "1.19");

$options->setFacets($facetOrGroup);

// if you want to combine multiple OR groups, you can use an AND group:
$facetAndGroup = new FacetANDGroup();
$facetAndGroup->addORGroup($facetOrGroup)
              ->addORGroup(new FacetORGroup(new Facet(FacetType::LICENSE, "MIT")));
              
// or
$facetAndGroup = $facetOrGroup->toANDGroup()
    ->addORGroup(new FacetORGroup(new Facet(FacetType::LICENSE, "MIT")));

$options->setFacets($facetAndGroup);

$projects = $modrinthClient->getProjects($options);
```

By default facets are checked using the equals operator. There are several other operators available:
```php
use \Aternos\ModrinthApi\Client\Options\Facets\FacetOperator;

$facet = new Facet(FacetType::DOWNLOADS, "1.20.1", FacetOperator::GREATER_THAN);
```

## Getting Additional Project Data
The Project wrapper provides methods to fetch additional data about the project.
```php
// get a specific project
$project = $modrinthClient->getProject("mclogs");

// get versions of the project
$versions = $project->getVersions();

// get a specific version
$version = $project->getVersion("2.6.2");

// get the members of the project
$members = $project->getMembers();
```

## Fetching Projects
You can also fetch individual projects by their id or slug:
```php
$project = $modrinthClient->getProject("mclogs");
```

or multiple projects by their ids:
```php
$projects = $modrinthClient->getProjects(["6DdCzpTL", "VPo0otUH"]);
```
It seems like fetching by slugs is also supported here but this is not documented.

### Check if a slug/id is used
```php
$projectId = $modrinthClient->checkProjectValidity("mclogs");

if ($projectId) {
    echo "project exists, id:" . $projectId;
} else {
    echo "project does not exist";
}
```

### Project Dependencies
Get dependencies of a project
```php
$dependencies = $project->getDependencies();

// on modrinth you can depend on a project or directly on a version
$projects = $dependencies->getProjects();
$versions = $dependencies->getVersions();
```


## Versions
```php
// get versions of a project by name
$versions = $modrinthClient->getProjectVersions("mclogs");

// get the versions from a project
$versions = $project->getVersions();

// get a specific version by its id
$version = $modrinthClient->getVersion("xzRGr4AC");

// get multiple versions by their ids
$versions = $modrinthClient->getVersions(["xzRGr4AC", "EwNN8uNA"]);
```

## Hashes
The modrinth API also allows you to find a version by its SHA1 or SHA512 hash:
```php
use \Aternos\ModrinthApi\Client\HashAlgorithm;

$hash = "5952253d61e199e82eb852c5824c3981b29b209d";

// returns the modrinth version for the given hash
$version = $modrinthClient->getVersionByHash($hash, HashAlgorithm::SHA1);

// fetch multiple versions at once
$versions = $modrinthClient->getVersionsByHashes([$hash], HashAlgorithm::SHA1);

// returns the latest version for a specific loader and version
$version = $modrinthClient->getLatestVersionByHash($hash, ["spigot"], ["1.19.4"], HashAlgorithm::SHA1);

// fetch multiple versions at once
$versions = $modrinthClient->getLatestVersionsByHashes([$hash], ["spigot"], ["1.19.4"], HashAlgorithm::SHA1);
```

## Users
```php
// get a user
$user = $modrinthClient->getUser("matthias");

// get projects of a user
$projects = $user->getProjects();

// get notifications (requires authentication)
$notifications = $user->getNotifications();

// get followed projects (requires authentication)
$projects = $user->getFollowedProjects();

// get payout history (requires authentication)
$history = $user->getPayoutHistory();
```

## Teams
```php
// get the members of a team
$members = $modrinthClient->getTeamMembers("ThaUQrOs");

// get the members of multiple teams at once
$teams = $modrinthClient->getTeams(["ThaUQrOs"]);
```

## Tags
You can fetch all available categories, loaders, game versions and licenses from the API.
Our library provides methods to search projects by these directly:
```php
$categories = $modrinthClient->getCategories();
$categories[0]->searchProjects();
$loaders = $modrinthClient->getLoaders();
$loaders[0]->searchProjects();
$gameVersions = $modrinthClient->getGameVersions();
$gameVersions[0]->searchProjects();
$licenses = $modrinthClient->getLicenses();
$licenses[0]->searchProjects();
```

You can also fetch donation platforms and report types:
```php
$reportTypes = $modrinthClient->getReportTypes();
$donationPlatforms = $modrinthClient->getDonationPlatforms();
```


## Updating the generated code
The generated code can be updated by installing the [openapi generator](https://openapi-generator.tech/docs/installation) running the following command:
```bash
openapi-generator-cli generate -c config.yaml
```