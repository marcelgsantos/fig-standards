# Meta Documento Mensagem HTTP

## 1. Resumo

O objetivo desta proposta é fornecer um conjunto de interfaces comuns para
mensagens HTTP como descrito em [RFC 7230](http://tools.ietf.org/html/rfc7230) e
[RFC 7231](http://tools.ietf.org/html/rfc7231), e URIs como descrito em
[RFC 3986](http://tools.ietf.org/html/rfc3986) (no contexto de uma mensagem HTTP).

- RFC 7230: http://www.ietf.org/rfc/rfc7230.txt
- RFC 7231: http://www.ietf.org/rfc/rfc7231.txt
- RFC 3986: http://www.ietf.org/rfc/rfc3986.txt

Todas as mensagens HTTP consistem na versão do protocolo HTTP sendo usado, cabeçalhos
e corpo da mensagem. Uma _Requisição_ baseia-se na mensagem para incluir o método
HTTP utilizado para fazer a requisição e a URI para o qual a requisição é feita.
Uma _Resposta_ inclui o código de status HTTP e a _reason phrase_.

No PHP, as mensagens HTTP são utilizadas em dois contextos:

- Para enviar uma requisição HTTP através da extensão `ext/curl`, da camada nativa
  de streams do PHP, etc. e processar a resposta HTTP recebida. Em outras palavras,
  mensagens HTTP são utilizadas quando utiliza-se o PHP como um _cliente HTTP_.
- Para processar uma requisição HTTP recebida pelo servidor e retornar uma
  resposta HTTP para o cliente que fez a solicitação. O PHP pode utilizar
  mensagens HTTP quando utilizado como uma _aplicação do lado do servidor_ para
  atender às solicitações HTTP.

Esta proposta apresenta uma API para descrever completamente todas as partes das
diferentes mensagens HTTP dentro do PHP.

## 2. Mensagens HTTP no PHP

O PHP não possui suporte nativo para mensagens HTTP.

### Suporte HTTP no lado do cliente

O PHP suporta o envio de requisições HTTP através de vários mecanismos:

- [Streams PHP](http://php.net/streams)
- A [extensão cURL](http://php.net/curl)
- [ext/http](http://php.net/http) (v2 também tenta contemplar o suporte no lado do servidor)

Streams PHP são a maneira mais conveniente e onipresente de enviar requisições HTTP,
mas representam uma série de limitações com relação ao suporte apropriado de
configurações SSL e fornece uma interface confusa sobre configurar coisas como
cabecalhos. O cURL disponibiliza um conjunto completo e expandido de funcionalidades,
mas, uma vez que não é uma extensão padrão, muitas vezes não está presente. A
extensão http sofre do mesmo problema que cURL, bem como o fato de que ela tem
tradicionalmente muito menos exemplos de uso.

A maioria das bibliotecas de clientes HTTP tendem a abstrair a implementação para
garantir que elas possam trabalhar em quaisquer ambientes que sejam executadas e
através de quaisquer camadas acima.

### Suporte HTTP no lado do servidor

O PHP utiliza Server APIs (SAPI) para interpretar requisições HTTP recebidas,
empacotar a entrada e passar a manipulação para scripts. O projeto original
do SAPI é espelhado no [Common Gateway Interface](http://www.w3.org/CGI/), que
empacotariam dados de requisição e os enviariam para variáveis de ambiente antes
de delegar para um script; o script então extrairia as variáveis de ambiente
a fim de processar a requisição e retornar a resposta.

O projeto do SAPI do PHP abstrai fontes comuns de entrada como cookies, argumentos
de query string e conteúdo de requisições POST codificados como url através de
superglobals (`$_COOKIE`, `$_GET` e `$_POST`, respectivamente) disponibilizando
uma camada de conveniência para desenvolvedores web.

No lado da resposta da equação, o PHP foi originalmente desenvolvido como uma
linguagem de template e permite misturar HTML e PHP; quaisquer porções de HTML
de um arquivo são imediatamente liberados para o buffer de saída. Aplicações
modernas e frameworks evitam esta prática pois pode levar a problemas em
relação ao envio da linha de status e/ou cabeçalhos de resposta; eles tendem
a reunir todos os cabeçalhos e conteúdo e enviá-los de uma só vez quando todo
o outro processamento da aplicação estiver completo. Deve se ter um cuidado
especial para assegurar que notificações de erros e outras ações que enviam conteúdo
para o buffer de saída não liberem o buffer de saída.

## 3. Why Bother?

HTTP messages are used in a wide number of PHP projects -- both clients and
servers. In each case, we observe one or more of the following patterns or
situations:

1. Projects use PHP's superglobals directly.
2. Projects will create implementations from scratch.
3. Projects may require a specific HTTP client/server library that provides
   HTTP message implementations.
4. Projects may create adapters for common HTTP message implementations.

As examples:

1. Just about any application that began development before the rise of
   frameworks, which includes a number of very popular CMS, forum, and shopping
   cart systems, have historically used superglobals.
2. Frameworks such as Symfony and Zend Framework each define HTTP components
   that form the basis of their MVC layers; even small, single-purpose
   libraries such as oauth2-server-php provide and require their own HTTP
   request/response implementations. Guzzle, Buzz, and other HTTP client
   implementations each create their own HTTP message implementations as well.
3. Projects such as Silex, Stack, and Drupal 8 have hard dependencies on
   Symfony's HTTP kernel. Any SDK built on Guzzle has a hard requirement on
   Guzzle's HTTP message implementations.
4. Projects such as Geocoder create redundant [adapters for common
   libraries](https://github.com/geocoder-php/Geocoder/tree/6a729c6869f55ad55ae641c74ac9ce7731635e6e/src/Geocoder/HttpAdapter).

Direct usage of superglobals has a number of concerns. First, these are
mutable, which makes it possible for libraries and code to alter the values,
and thus alter state for the application. Additionally, superglobals make unit
and integration testing difficult and brittle, leading to code quality
degradation.

In the current ecosystem of frameworks that implement HTTP message abstractions,
the net result is that projects are not capable of interoperability or
cross-pollination. In order to consume code targeting one framework from
another, the first order of business is building a bridge layer between the
HTTP message implementations. On the client-side, if a particular library does
not have an adapter you can utilize, you need to bridge the request/response
pairs if you wish to use an adapter from another library.

Finally, when it comes to server-side responses, PHP gets in its own way: any
content emitted before a call to `header()` will result in that call becoming a
no-op; depending on error reporting settings, this can often mean headers
and/or response status are not correctly sent. One way to work around this is
to use PHP's output buffering features, but nesting of output buffers can
become problematic and difficult to debug. Frameworks and applications thus
tend to create response abstractions for aggregating headers and content that
can be emitted at once - and these abstractions are often incompatible.

Thus, the goal of this proposal is to abstract both client- and server-side
request and response interfaces in order to promote interoperability between
projects. If projects implement these interfaces, a reasonable level of
compatibility may be assumed when adopting code from different libraries.

It should be noted that the goal of this proposal is not to obsolete the
current interfaces utilized by existing PHP libraries. This proposal is aimed
at interoperability between PHP packages for the purpose of describing HTTP
messages.

## 4. Scope

### 4.1 Goals

* Provide the interfaces needed for describing HTTP messages.
* Focus on practical applications and usability.
* Define the interfaces to model all elements of the HTTP message and URI
  specifications.
* Ensure that the API does not impose arbitrary limits on HTTP messages. For
  example, some HTTP message bodies can be too large to store in memory, so we
  must account for this.
* Provide useful abstractions both for handling incoming requests for
  server-side applications and for sending outgoing requests in HTTP clients.

### 4.2 Non-Goals

* This proposal does not expect all HTTP client libraries or server-side
  frameworks to change their interfaces to conform. It is strictly meant for
  interoperability.
* While everyone's perception of what is and is not an implementation detail
  varies, this proposal should not impose implementation details. As
  RFCs 7230, 7231, and 3986 do not force any particular implementation,
  there will be a certain amount of invention needed to describe HTTP message
  interfaces in PHP.

## 5. Design Decisions

### Message design

The `MessageInterface` provides accessors for the elements common to all HTTP
messages, whether they are for requests or responses. These elements include:

- HTTP protocol version (e.g., "1.0", "1.1")
- HTTP headers
- HTTP message body

More specific interfaces are used to describe requests and responses, and more
specifically the context of each (client- vs. server-side). These divisions are
partly inspired by existing PHP usage, but also by other languages such as
Ruby's [Rack](https://rack.github.io),
Python's [WSGI](https://www.python.org/dev/peps/pep-0333/),
Go's [http package](http://golang.org/pkg/net/http/),
Node's [http module](http://nodejs.org/api/http.html), etc.

### Why are there header methods on messages rather than in a header bag?

The message itself is a container for the headers (as well as the other message
properties). How these are represented internally is an implementation detail,
but uniform access to headers is a responsibility of the message.

### Why are URIs represented as objects?

URIs are values, with identity defined by the value, and thus should be modeled
as value objects.

Additionally, URIs contain a variety of segments which may be accessed many
times in a given request -- and which would require parsing the URI in order to
determine (e.g., via `parse_url()`). Modeling URIs as value objects allows
parsing once only, and simplifies access to individual segments. It also
provides convenience in client applications by allowing users to create new
instances of a base URI instance with only the segments that change (e.g.,
updating the path only).

### Why does the request interface have methods for dealing with the request-target AND compose a URI?

RFC 7230 details the request line as containing a "request-target". Of the four
forms of request-target, only one is a URI compliant with RFC 3986; the most
common form used is origin-form, which represents the URI without the
scheme or authority information. Moreover, since all forms are valid for
purposes of requests, the proposal must accommodate each.

`RequestInterface` thus has methods relating to the request-target. By default,
it will use the composed URI to present an origin-form request-target, and, in
the absence of a URI instance, return the string "/".  Another method,
`withRequestTarget()`, allows specifying an instance with a specific
request-target, allowing users to create requests that use one of the other
valid request-target forms.

The URI is kept as a discrete member of the request for a variety of reasons.
For both clients and servers, knowledge of the absolute URI is typically
required. In the case of clients, the URI, and specifically the scheme and
authority details, is needed in order to make the actual TCP connection. For
server-side applications, the full URI is often required in order to validate
the request or to route to an appropriate handler.

### Why value objects?

The proposal models messages and URIs as [value objects](http://en.wikipedia.org/wiki/Value_object).

Messages are values where the identity is the aggregate of all parts of the
message; a change to any aspect of the message is essentially a new message.
This is the very definition of a value object. The practice by which changes
result in a new instance is termed [immutability](http://en.wikipedia.org/wiki/Immutable_object),
and is a feature designed to ensure the integrity of a given value.

The proposal also recognizes that most clients and server-side
applications will need to be able to easily update message aspects, and, as
such, provides interface methods that will create new message instances with
the updates. These are generally prefixed with the verbiage `with` or
`without`.

Value objects provides several benefits when modeling HTTP messages:

- Changes in URI state cannot alter the request composing the URI instance.
- Changes in headers cannot alter the message composing them.

In essence, modeling HTTP messages as value objects ensures the integrity of
the message state, and prevents the need for bi-directional dependencies, which
can often go out-of-sync or lead to debugging or performance issues.

For HTTP clients, they allow consumers to build a base request with data such
as the base URI and required headers, without needing to build a brand new
request or reset request state for each message the client sends:

```php
$uri = new Uri('http://api.example.com');
$baseRequest = new Request($uri, null, [
    'Authorization' => 'Bearer ' . $token,
    'Accept'        => 'application/json',
]);;

$request = $baseRequest->withUri($uri->withPath('/user'))->withMethod('GET');
$response = $client->send($request);

// get user id from $response

$body = new StringStream(json_encode(['tasks' => [
    'Code',
    'Coffee',
]]));;
$request = $baseRequest
    ->withUri($uri->withPath('/tasks/user/' . $userId))
    ->withMethod('POST')
    ->withHeader('Content-Type', 'application/json')
    ->withBody($body);
$response = $client->send($request)

// No need to overwrite headers or body!
$request = $baseRequest->withUri($uri->withPath('/tasks'))->withMethod('GET');
$response = $client->send($request);
```

On the server-side, developers will need to:

- Deserialize the request message body.
- Decrypt HTTP cookies.
- Write to the response.

These operations can be accomplished with value objects as well, with a number
of benefits:

- The original request state can be stored for retrieval by any consumer.
- A default response state can be created with default headers and/or message body. 

Most popular PHP frameworks have fully mutable HTTP messages today. The main
changes necessary in consuming true value objects are:

- Instead of calling setter methods or setting public properties, mutator
  methods will be called, and the result assigned.
- Developers must notify the application on a change in state.

As an example, in Zend Framework 2, instead of the following:

```php
function (MvcEvent $e)
{
    $response = $e->getResponse();
    $response->setHeaderLine('x-foo', 'bar');
}
```

one would now write:

```php
function (MvcEvent $e)
{
    $response = $e->getResponse();
    $e->setResponse(
        $response->withHeader('x-foo', 'bar')
    );
}
```

The above combines assignment and notification in a single call.

This practice has a side benefit of making explicit any changes to application
state being made.

### New instances vs returning $this

One observation made on the various `with*()` methods is that they can likely
safely `return $this;` if the argument presented will not result in a change in
the value. One rationale for doing so is performance (as this will not result in
a cloning operation).

The various interfaces have been written with verbiage indicating that
immutability MUST be preserved, but only indicate that "an instance" must be
returned containing the new state. Since instances that represent the same value
are considered equal, returning `$this` is functionally equivalent, and thus
allowed.

### Using streams instead of X

`MessageInterface` uses a body value that must implement `StreamableInterface`. This
design decision was made so that developers can send and receive (and/or receive
and send) HTTP messages that contain more data than can practically be stored in
memory while still allowing the convenience of interacting with message bodies
as a string. While PHP provides a stream abstraction by way of stream wrappers,
stream resources can be cumbersome to work with: stream resources can only be
cast to a string using `stream_get_contents()` or manually reading the remainder
of a string. Adding custom behavior to a stream as it is consumed or populated
requires registering a stream filter; however, stream filters can only be added
to a stream after the filter is registered with PHP (i.e., there is no stream
filter autoloading mechanism).

The use of a well- defined stream interface allows for the potential of
flexible stream decorators that can be added to a request or response
pre-flight to enable things like encryption, compression, ensuring that the
number of bytes downloaded reflects the number of bytes reported in the
`Content-Length` of a response, etc. Decorating streams is a well-established
[pattern in the Java](http://docs.oracle.com/javase/7/docs/api/java/io/package-tree.html)
and [Node](http://nodejs.org/api/stream.html#stream_class_stream_transform_1)
communities that allows for very flexible streams.

The majority of the `StreamableInterface` API is based on
[Python's io module](http://docs.python.org/3.1/library/io.html), which provides
a practical and consumable API. Instead of implementing stream
capabilities using something like a `WritableStreamInterface` and
`ReadableStreamInterface`, the capabilities of a stream are provided by methods
like `isReadable()`, `isWritable()`, etc. This approach is used by Python,
[C#, C++](http://msdn.microsoft.com/en-us/library/system.io.stream.aspx),
[Ruby](http://www.ruby-doc.org/core-2.0.0/IO.html),
[Node](http://nodejs.org/api/stream.html), and likely others.

#### What if I just want to return a file?

In some cases, you may want to return a file from the filesystem. The typical
way to do this in PHP is one of the following:

```php
readfile($filename);

stream_copy_to_stream(fopen($filename, 'r'), fopen('php://output', 'w'));
```

Note that the above omits sending appropriate `Content-Type` and
`Content-Length` headers; the developer would need to emit these prior to
calling the above code.

The equivalent using HTTP messages would be to use a `StreamableInterface`
implementation that accepts a filename and/or stream resource, and to provide
this to the response instance. A complete example, including setting appropriate
headers:

```php
// where Stream is a concrete StreamableInterface:
$stream   = new Stream($filename);
$finfo    = new finfo(FILEINFO_MIME);
$response = $response
    ->withHeader('Content-Type', $finfo->file($filename))
    ->withHeader('Content-Length', (string) filesize($filename))
    ->withBody($stream);
```

Emitting this response will send the file to the client.

#### What if I want to directly emit output?

Directly emitting output (e.g. via `echo`, `printf`, or writing to the
`php://output` stream) is generally only advisable as a performance optimization
or when emitting large data sets. If it needs to be done and you still wish
to work in an HTTP message paradigm, one approach would be to use a
callback-based `StreamableInterface` implementation, per [this
example](https://github.com/phly/psr7examples#direct-output). Wrap any code
emitting output directly in a callback, pass that to an appropriate
`StreamableInterface` implementation, and provide it to the message body:

```php
$output = new CallbackStream(function () use ($request) {
    printf("The requested URI was: %s<br>\n", $request->getUri());
    return '';
});
return (new Response())
    ->withHeader('Content-Type', 'text/html')
    ->withBody($output);
```

#### What if I want to use an iterator for content?

Ruby's Rack implementation uses an iterator-based approach for server-side
response message bodies. This can be emulated using an HTTP message paradigm via
an iterator-backed `StreamableInterface` approach, as [detailed in the
psr7examples repository](https://github.com/phly/psr7examples#iterators-and-generators).

### Why are streams mutable?

The `StreamableInterface` API includes methods such as `write()` which can
change the message content -- which directly contradicts having immutable
messages.

The problem that arises is due to the fact that the interface is intended to
wrap a PHP stream or similar. A write operation therefore will proxy to writing
to the stream. Even if we made `StreamableInterface` immutable, once the stream
has been updated, any instance that wraps that stream will also be updated --
making immutability impossible to enforce.

Our recommendation is that implementations use read-only streams for
server-side requests and client-side responses.

### Rationale for ServerRequestInterface

The `RequestInterface` and `ResponseInterface` have essentially 1:1
correlations with the request and response messages described in
[RFC 7230](http://www.ietf.org/rfc/rfc7230.txt) They provide interfaces for
implementing value objects that correspond to the specific HTTP message types
they model.

For server-side applications there are other considerations for
incoming requests:

- Access to server parameters (potentially derived from the request, but also
  potentially the result of server configuration, and generally represented
  via the `$_SERVER` superglobal; these are part of the PHP Server API (SAPI)).
- Access to the query string arguments (usually encapsulated in PHP via the
  `$_GET` superglobal).
- Access to the parsed body (i.e., data deserialized from the incoming request
  body; in PHP, this is typically the result of POST requests using
  `application/x-www-form-urlencoded` content types, and encapsulated in the
  `$_POST` superglobal, but for non-POST, non-form-encoded data, could be
  an array or an object).
- Access to uploaded files (encapsulated in PHP via the `$_FILES` superglobal).
- Access to cookie values (encapsulated in PHP via the `$_COOKIE` superglobal).
- Access to attributes derived from the request (usually, but not limited to,
  those matched against the URL path).

Uniform access to these parameters increases the viability of interoperability
between frameworks and libraries, as they can now assume that if a request
implements `ServerRequestInterface`, they can get at these values. It also
solves problems within the PHP language itself:

- Until 5.6.0, `php://input` was read-once; as such, instantiating multiple
  request instances from multiple frameworks/libraries could lead to
  inconsistent state, as the first to access `php://input` would be the only
  one to receive the data.
- Unit testing against superglobals (e.g., `$_GET`, `$_FILES`, etc.) is
  difficult and typically brittle. Encapsulating them inside the
  `ServerRequestInterface` implementation eases testing considerations.

### Why "parsed body" in the ServerRequestInterface?

Arguments were made to use the terminology "BodyParams", and require the value
to be an array, with the following rationale:

- Consistency with other server-side parameter access.
- `$_POST` is an array, and the 80% use case would target that superglobal.
- A single type makes for a strong contract, simplifying usage.

The main argument is that if the body parameters are an array, developers have
predictable access to values:

```php
$foo = isset($request->getBodyParams()['foo'])
    ? $request->getBodyParams()['foo']
    : null;
```

The argument for using "parsed body" was made by examining the domain. A message
body can contain literally anything. While traditional web applications use
forms and submit data using POST, this is a use case that is quickly being
challenged in current web development trends, which are often API centric, and
thus use alternate request methods (notably PUT and PATCH), as well as
non-form-encoded content (generally JSON or XML) that _can_ be coerced to arrays
in many cases, but in many cases also _cannot_ or _should not_.

If forcing the property representing the parsed body to be only an array,
developers then need a shared convention about where to put the results of
parsing the body. These might include:

- A special key under the body parameters, such as `__parsed__`.
- A special named attribute, such as `__body__`.

The end result is that a developer now has to look in multiple locations:

```php
$data = $request->getBodyParams();
if (isset($data['__parsed__']) && is_object($data['__parsed__'])) {
    $data = $data['__parsed__'];
}

// or:
$data = $request->getBodyParams();
if ($request->hasAttribute('__body__')) {
    $data = $request->getAttribute('__body__');
}
```

The solution presented is to use the terminology "ParsedBody", which implies
that the values are the results of parsing the message body. This also means
that the return value _will_ be ambiguous; however, because this is an attribute
of the domain, this is also expected. As such, usage will become:

```php
$data = $request->getParsedBody();
if (! $data instanceof \stdClass) {
    // raise an exception!
}
// otherwise, we have what we expected
```

This approach removes the limitations of forcing an array, at the expense of
ambiguity of return value. Considering that the other suggested solutions —
pushing the parsed data into a special body parameter key or into an attribute —
also suffer from ambiguity, the proposed solution is simpler as it does not
require additions to the interface specification. Ultimately, the ambiguity
enables the flexibility required when representing the results of parsing the
body.

### Why is no functionality included for retrieving the "base path"?

Many frameworks provide the ability to get the "base path," usually considered
the path up to and including the front controller. As an example, if the
application is served at `http://example.com/b2b/index.php`, and the current URI
used to request it is `http://example.com/b2b/index.php/customer/register`, the
functionality to retrieve the base path would return `/b2b/index.php`. This value
can then be used by routers to strip that path segment prior to attempting a
match.

This value is often also then used for URI generation within applications;
parameters will be passed to the router, which will generate the path, and
prefix it with the base path in order to return a fully-qualified URI. Other
tools — typically view helpers, template filters, or template functions — are
used to resolve a path relative to the base path in order to generate a URI for
linking to resources such as static assets.

On examination of several different implementations, we noticed the following:

- The logic for determining the base path varies widely between implementations.
  As an example, compare the [logic in ZF2](https://github.com/zendframework/zf2/blob/release-2.3.7/library/Zend/Http/PhpEnvironment/Request.php#L477-L575)
  to the [logic in Symfony 2](https://github.com/symfony/symfony/blob/2.7/src/Symfony/Component/HttpFoundation/Request.php#L1858-L1877).
- Most implementations appear to allow manual injection of a base path to the
  router and/or any facilities used for URI generation.
- The primary use cases — routing and URI generation — typically are the only
  consumers of the functionality; developers usually do not need to be aware
  of the base path concept as other objects take care of that detail for them.
  As examples:
  - A router will strip off the base path for you during routing; you do not
    need to pass the modified path to the router.
  - View helpers, template filters, etc. typically are injected with a base path
    prior to invocation. Sometimes this is manually done, though more often it
    is the result of framework wiring.
- All sources necessary for calculating the base path *are already in the
  `RequestInterface` instance*, via server parameters and the URI instance.

Our stance is that base path detection is framework and/or application
specific, and the results of detection can be easily injected into objects that
need it, and/or calculated as needed using utility functions and/or classes from
the `RequestInterface` instance itself.

### Why does getUploadedFiles() return objects instead of arrays?

`getUploadedFiles()` returns a tree of `Psr\Http\Message\UploadedFileInterface`
instances. This is done primarily to simplify specification: instead of
requiring paragraphs of implementation specification for an array, we specify an
interface.

Additionally, the data in an `UploadedFileInterface` is normalized to work in
both SAPI and non-SAPI environments. This allows creation of processes to parse
the message body manually and assign contents to streams without first writing
to the filesystem, while still allowing proper handling of file uploads in SAPI
environments.

### What about "special" header values?

A number of header values contain unique representation requirements which can
pose problems both for consumption as well as generation; in particular, cookies
and the `Accept` header.

This proposal does not provide any special treatment of any header types. The
base `MessageInterface` provides methods for header retrieval and setting, and
all header values are, in the end, string values.

Developers are encouraged to write commodity libraries for interacting with
these header values, either for the purposes of parsing or generation. Users may
then consume these libraries when needing to interact with those values.
Examples of this practice already exist in libraries such as
[willdurand/Negotiation](https://github.com/willdurand/Negotiation) and
[aura/accept](https://github.com/pmjones/Aura.Accept). So long as the object
has functionality for casting the value to a string, these objects can be
used to populate the headers of an HTTP message.

## 6. People

### 6.1 Editor(s)

* Matthew Weier O'Phinney

### 6.2 Sponsors

* Paul M. Jones
* Beau Simensen (coordinator)

### 6.3 Contributors

* Michael Dowling
* Larry Garfield
* Evert Pot
* Tobias Schultze
* Bernhard Schussek
* Anton Serdyuk
* Phil Sturgeon
* Chris Wilkinson
