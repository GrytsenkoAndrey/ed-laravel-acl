# ed-laravel-acl

## RestFul Spec

Some things that triggers me really bad is to find that several API’s that proclaim being RestFul, they are not, in the essence they don’t follow the “overall standards” (I put the double quotes since it’s up to you at the end, this is not like a ACID compliance of RDBMS). So for instance, they use verbs in the paths like: POST /api/user/10/reset-data 😠

In our scenario, we won’t proceed that way. I’m a fan of the KISS principle (Keep It Simple, Stupid), so I’ll show you some practical routes for our example of constructing our elegant ACL.

Create an unit inside the course 10

POST /course/10/unit

Get all the units related from the Course 20

GET /course/20/unit

Delete the unit 7

DELETE /unit/7

Updates the unit 7

PATCH /unit/7

So think is this use case. We have 3 roles, the admin can use all of them, because he’s the boss obviously, and can do whatever he wants, the second roles it’s the teacher responsible of only managing the units, but can read everything related to the course, and at the end, the student, he can only read.

How do you restrict that? where do you put the magic if…

```
if ($user->getRole() === 'admin' && $url … )
```

It’s not an easy question, since you can do it in several ways, and by the way, I know you can do this with Laravel and all the fancy magic behind, but that’s not the purpose of this post, if you want to know how to do it “a la laravel” go to the docs from them.

## First step: Canonical resources

On my end, I created the term of “Canonical resource” to define a path that would represent a resource that needs to be controlled it’s access.

So the canonical for the previous paths would be:

```
Course unit resource /course/{course_id}/unit
Unit resource /unit
```

In this way, I can use these templates to match any path and categorize into any of my audited resources.

## Second Step: Permissions

With all our resource identified, we need to tell our system what access to we have to them, so to keep it simple, I have created this spec:
c: Create
r: Read
u: Update
d: Delete

## Third: Mix baby

Since we have already the definition of our roles, and the acces that hey have to the existing resources, we just need to map that into an array, example:

```
// From some settings.php that you have
[
  'permissions' => [
     RolesEnum::ADMIN => [
      '/course' => 'crud', // Create, read, update, delete courses
      '/course/{course_id}/unit' => 'cr', // Create and read
      '/unit' => 'rud', // Read, update and delete
     ],
     RolesEnum::TEACHER => [
      '/course' => 'r', // Read the course data
      '/course/{course_id}/unit' => 'r', // Read all the course units data
      '/unit' => 'ru',   ], // Read and update any unit
    ],
];
```

Then, check if a user/role have access to certain resources is pretty straightforward

```
class AclManager {

  /**
  * @param CacheInterface $cache
  * @param array $permissions
  */
 public function __construct(array $permissions = []) {
    $this->permissions = $permissions;
 }

  /**
  * @param string $role
  * @param string $method
  * @param string $path
  * @return bool
  */
 public function hasAccessToPath(string $role, string $method, string $path): bool {
    $canonical = $this->pathToCanonical($path);
    $access = $this->getAccessFromMethod($method);
  
    if ($this->permissions === [] ||
      !isset($this->permissions[$role]) ||
      !isset($this->permissions[$role][$canonical]))
    {
       return false;
    }
  
    $allowedAccess = $this->permissions[$role][$canonical];
  
    return (bool)preg_match("/[$allowedAccess]/", $access);
 }

 /**
  * @param string $method
  * @throws InvalidArgumentException
  * @return string
  */
 public function getAccessFromMethod(string $method): string
 {
    if ($method === 'POST')
    {
       return 'c';
    }
  
    if ($method === 'GET')
    {
       return 'r';
    }
  
    if ($method === 'PATCH' || $method === 'PUT')
    {
       return 'u';
    }
  
    if ($method === 'DELETE')
    {
       return 'd';
    }
  
    throw new \InvalidArgumentException('Not Mapped method definition as ACL access');
 }

 /**
  * @param string $path
  * @param string $basePath
  * @return string
  */
 public function pathToCanonical(string $path, string $basePath = '/api/v1/'): string {
  $i = 1;
  $parts = array_values(array_filter(explode('/', str_replace($basePath, '/', $path))));
  $canonical = [];
  $resource = null;

  foreach ($parts as $part)
  {
   if ($resource === null)
   {
    $resource = $part;
   }

   if ($i % 2 === 0)
   {
    $part = '{' . strtolower($resource) . '_id}';
    $resource = null;
   }

   $canonical[] = $part;

   $i++;
  }

  if (count($canonical) % 2 === 0)
  {
   unset($canonical[count($canonical) -1]);
  }

  return '/' . implode('/', $canonical);
 }
}
```

Then, you can add it to your preferred container, in my case Php-di

```
 AclManager::class => function($container)
 {
  $permissions = $container->get('config')->get('acl.permissions');
  return new AclManager($permissions);
 },
```


## Final Step: Create an ACL Middleware

I think this concept it’s already widely used and understood, so I will show you how to use from it. Just consider that this ACL approach needs to know the role attached to the processed request in order to work.

```
class AclMiddleware implements MiddlewareInterface, ContainerAwareInterface, CanStopExecutionInterface
{
 use ContainerAwareTrait;
 const BASE_URL = '/api/v1/';

 /**
  * @param RequestInterface $request
  * @param ResponseInterface $response
  * @return array
  * @throws AuthenticationException
  * @throws PermissionDeniedException
  * @throws ContainerExceptionInterface
  * @throws NotFoundExceptionInterface
  */
 public function __invoke(RequestInterface $request, ResponseInterface $response) {
  // We obtain an instance of our ACL Manager from the DI container
  $aclManager = $this->container->get(AclManager::class);
  
  $method = $request->getMethod();
  $path = $request->getUri()->getPath();
  
  // At this point, the user should be already authenticated
  $authUser = $request->getAttribute('authenticated_user');

  if (!$authUser) {
      throw new AuthenticationException;
  }
  
  if (!$aclManager->hasAccessToPath($authUser->getRole(), $method, $path)) {
      throw new PermissionDeniedException;
  }

  return [$request, $response];
 }
}
```

So, every time your API run the middleware stack and get into the ACL one, it will check the current role, the http method and the request path in order to allow or deny the access, without any extra effort from you!.
So now, you can run and tell grandpa that you build an ACL by your own.

## Future work / Improvements

- Move the permissions spec into a caching layer to speed up the compute of access, might be useful to inject a Caching dependency into the Acl Manager class.
- Define the list of resources independently and then, create some sort of “module” to group them, then it would be easy to attach “modules” to roles, instead of defining each permission by role.
- Battle test the functions (haha) this is a side project remember : D

Keep in mind this is a simple example/approach about how to tackle this, certainly it’s a starting point, but you know that every project have it’s own requirements and business rules, but with this simple approach I have being able to tackle almost any role/permission issue.

