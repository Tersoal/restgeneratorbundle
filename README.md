# Voryx REST Generator Bundle
[![SensioLabsInsight](https://insight.sensiolabs.com/projects/ac1842d9-4e36-45cc-8db1-b97e2e62540e/big.png)](https://insight.sensiolabs.com/projects/ac1842d9-4e36-45cc-8db1-b97e2e62540e)

## About

A CRUD like REST Generator

## Features

* Generators RESTful action from entity
* Simplifies setting up a RESTful Controller

## Installation
Require the "voryx/restgeneratorbundle" package in your composer.json and update your dependencies.

* Note: you need to add  this repository to composer.json

```php
"repositories": [
    {
        "type": "vcs",
        "url": "https://github.com/Tersoal/restgeneratorbundle"
    }
]
```

```bash
$ php composer.phar require voryx/restgeneratorbundle
```

Add the VoryxRestGeneratorBundle to your application's kernel along with other dependencies:

```php
public function registerBundles()
{
    $bundles = array(
        //...
          new Voryx\RESTGeneratorBundle\VoryxRESTGeneratorBundle(),
          new FOS\RestBundle\FOSRestBundle(),
          new JMS\SerializerBundle\JMSSerializerBundle($this),
          new Nelmio\CorsBundle\NelmioCorsBundle(),
          
          //If you would like to use Hateoas option
          new Bazinga\Bundle\HateoasBundle\BazingaHateoasBundle(),
          new WhiteOctober\PagerfantaBundle\WhiteOctoberPagerfantaBundle(),
        //...
    );
    //...
}
```

## Configuration

This bundle depends on a number of other Symfony bundles, so they need to be configured in order for the generator to work properly

```yaml
framework:
    csrf_protection: false #only use for public API

fos_rest:
    routing_loader:
        default_format: json
    param_fetcher_listener: true
    body_listener: true
    #disable_csrf_role: ROLE_USER
    body_converter:
        enabled: true
    view:
        view_response_listener: force

nelmio_cors:
    defaults:
        allow_credentials: false
        allow_origin: []
        allow_headers: []
        allow_methods: []
        expose_headers: []
        max_age: 0
    paths:
        '^/api/':
            allow_origin: ['*']
            allow_headers: ['*']
            allow_methods: ['POST', 'PUT', 'GET', 'DELETE']
            max_age: 3600

sensio_framework_extra:
    request: { converters: true }
    view:    { annotations: false }
    router:  { annotations: true }
```

## Generating the Controller


Generate the REST controller

```bash
$ php app/console voryx:generate:rest
```
    
This will guide you through the generator which will generate a RESTful controller for an entity.


## Example

Create a new entity called 'Post':

```bash
$ php app/console doctrine:generate:entity --entity=AppBundle:Post --format=annotation --fields="name:string(255) description:string(255)" --no-interaction
```

Update the database schema:

```bash
$ php app/console doctrine:schema:update --force
```

Generate the API controller:

```bash
$ php app/console voryx:generate:rest --entity="AppBundle:Post"
```

### Available options
* entity (mandatory)

The target entity of the new API controller.
 
* route-prefix

Prefix used on all routes for this entity.

* overwrite

Do not stop the generation if rest api controller already exist, thus overwriting all generated files

* resource

The object will return with the resource name

* document

Use NelmioApiDocBundle to document the controller

* hateoas

Use BazingaHateoasBundle and WhiteOctoberPagerfantaBundle to handle the response on cget action

* jms-group

JMSSerializerBundle group added to output View and Nelmio Doc. Add multiple times to give an array

### Hateoas requirement

To handle hateoas response on cget action, repository returned data must be paginated. By now only Pagerfanta is supported.
Thus, the recommendation is to create a parent class for all repositories and add this:

**CAUTION:** This documentation is incomplete, because the use of a library for add dynamic criteria. This will be completed very soon.

```php
    /**
     * Finds all the resources by criteria given, and return pagination.
     * Can do ordering, limit and offset.
     *
     * @param string[] $criteria Array which contains the criteria as key value
     * @param string[] $orderBy  Array which contains the sorting as key value
     * @param int      $limit    The limit
     * @param int      $page     The page
     *
     * @return array|Pagerfanta|Paginator
     */
    public function findPaginated(array $criteria = [], array $orderBy = [], $limit = 10, $page = 1)
    {
        $queryBuilder = $this->getQueryBuilder();
        $this->addCriteria($queryBuilder, $criteria);
        $this->orderBy($queryBuilder, $orderBy);

        if ($limit && $page) {
            if (class_exists('Pagerfanta\Pagerfanta')) {
                return $this->paginatePagerfanta($queryBuilder->getQuery(), $page, $limit);
            }

            return $this->paginate($queryBuilder->getQuery(), $page, $limit);
        }

        return $queryBuilder->getQuery()->getResult();
    }
    
    /**
     * Gets query builder.
     *
     * @return \Doctrine\ORM\QueryBuilder
     */
    protected function getQueryBuilder()
    {
        return $this->createQueryBuilder($this->getAlias());
    }

    /**
     * Gets paginated data.
     *
     * @param Query|QueryBuilder $dql
     * @param int                $currentPage
     * @param int                $pageSize
     *
     * @return Paginator
     */
    protected function paginate($dql, $currentPage = 1, $pageSize = 10)
    {
        $paginator = new Paginator($dql);

        $paginator
            ->getQuery()
            ->setFirstResult($pageSize * ($currentPage - 1)) // set the offset
            ->setMaxResults($pageSize); // set the limit

        return $paginator;
    }

    /**
     * Gets paginated data by Pagerfanta.
     *
     * @param Query|QueryBuilder $dql
     * @param int                $currentPage
     * @param int                $pageSize
     *
     * @return Pagerfanta
     */
    protected function paginatePagerfanta($dql, $currentPage = 1, $pageSize = 10)
    {
        $adapter = new DoctrineORMAdapter($dql);
        $pagerfanta = new Pagerfanta($adapter);

        $pagerfanta
            ->setMaxPerPage($pageSize)
            ->setCurrentPage($currentPage);

        return $pagerfanta;
    }
```

### Using the API
If you selected the default options you'll be able to start using the API like this:

Creating a new post (`POST`)

```bash
$ curl -i -H "Content-Type: application/json" -X POST -d '{"name" : "Test Post", "description" : "This is a test post"}' http://localhost/app_dev.php/api/posts
```

Updating (`PUT`)

```bash
$ curl -i -H "Content-Type: application/json" -X PUT -d '{"name" : "Test Post 1", "description" : "This is an updated test post"}' http://localhost/app_dev.php/api/posts/1
```

Get all posts (`GET`)

```bash
$ curl http://localhost/app_dev.php/api/posts
```

Get one post (`GET`)

```bash
$ curl http://localhost/app_dev.php/api/posts/1
```


Delete (`DELETE`)

```bash
$ curl -X DELETE  http://localhost/app_dev.php/api/posts/1
```


## Related Entities

If you want the form to be able to convert related entities into the correct entity id on POST, PUT or PATCH, use the voryx_entity form type

```php
#Form/PostType()

    ->add(
        'user', 'voryx_entity', array(
            'class' => 'Acme\Bundle\Entity\User'
        )
    )
```
