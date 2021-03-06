# Debug
Spiral provides an embedded debug component with a set of classes useful in developing and debugging your application. Debug does not replace existing debugging tools like XDebug. It was created to simplify your application development and profiling in remote enviroments.

## Loggers
The Spiral Logger class is built on top of the PSR3 standard and is compatible with it. You can simply replace the default application logger with Monolog classes and etc.
Spiral loggers aggregate log messages using a channel name. In most cases (when the LoggerTrait was used) the channel name will be the same as the parent class.

If you wish to access the logger in your code (for example in a controller action) you can simply declare the dependency:

```php
public function index(Logger $logger)
{
    $logger->alert('abc!');
    
    //Loggers support message interpolation
    $logger->warning('message {name}!', ['name' => 'value']);
}
```

> This code will create an instance of Logger associated with the global channel "@global". Spiral 

### Log Handlers
Spiral provides a simplistic way to create handlers to store or send specific log messages. You can add your handler to Logger using the setHandler method:

```php
public function index(Logger $logger)
{
    $logger->setHandler(Logger::ERROR, new MyHandler());
}
```

Your handler must only define __invoke method with the following arguments:

```php
/**
 * Handle log message.
 *
 * @param int    $level
 * @param string $message
 * @param array  $context
 */
public function __invoke($level, $message, array $context = []);
```

If you wish your handler to log every log message (every level) you can assing it to `Logger::ALL`:

```php
public function index(Logger $logger)
{
    $logger->setHandler(Logger::ALL, function ($level, $messsage, array $context) {
        dump($level);
        dump($message);
    });
}
```

> Message will come pre-interpolated into handler method.

Every logger instance requested via dependency injection will receive an instance of the Debug component. This instance provides the ability to pre-configure some 
log handlers using the debug configuraion:

```php
'loggers'       => [
    Logger::GLOBAL_CHANNEL  => [
        Logger::ALL   => [
            'class'    => FileHandler::class,
            'filename' => directory('runtime') . '/logging/global.log'
        ]
    ],
    Debugger::class       => [
        Logger::ERROR => [
            'class'    => FileHandler::class,
            'filename' => directory('runtime') . '/logging/errors.log'
        ],
        Logger::ALL   => [
            'class'    => FileHandler::class,
            'filename' => directory('runtime') . '/logging/debug.log'
        ]
    ],
    HttpDispatcher::class => [
        Logger::WARNING => [
            'class'    => FileHandler::class,
            'filename' => directory('runtime') . '/logging/httpErrors.log'
        ]
    ]
],
```

As you can see, you are able to assign handlers to specific channels and log levels. However this  has some limitations - all used handlers must implement `HandlerInterface` in order to be properly created and configured:

```php
interface HandlerInterface
{
    /**
     * HandlerInterface should only accept options from debug. Use depends method for additional
     * classes.
     *
     * @param array $options
     */
    public function __construct(array $options);

    /**
     * Handle log message.
     *
     * @param int    $level
     * @param string $message
     * @param array  $context
     */
    public function __invoke($level, $message, array $context = []);
}
```

The default file handler which you can see in the configuration provides the following set of options you can alter or leave as the default:

```php
/**
 * @var array
 */
protected $options = [
    'filename'      => '',                             //This option must be declared
    'filesize'      => 2097152,                        //Max log filesize is 2MB
    'mode'          => FilesInterface::RUNTIME,        //Mode
    'rotatePostfix' => '.old',                         //When log size exeeds max filesize it's will be rotated by addind this
                                                       //postfix to filename    
    'format'        => '{date}: [{level}] {message}',  //Messaged in log file will look like that
    'dateFormat'    => 'H:i:s d.m.Y'                   //Date format will be interpolated into line format
];
```

### Logger Trait
To simplify access to the logger, Spiral provides a `LoggerTrait` which is compatible with `LoggerAwareInterface` but provides a few additional features. Let's view an example using a controller:

```php
class HomeController extends Controller
{
    use LoggerTrait;

    public function index()
    {
        $this->logger()->alert('test');
    }
}
```

As you can see, the LoggerTrait has a helper method for us, `logger()`, which will create an instance of the `LoggerInterface` for us on demand. The only argument will be provided to such logger is `name` which will be equal to class name (Controllers\HomeController).

> By default spiral binds `LoggerInterface` to `Logger`.

LoggerTrait has a few abilities you might consider:
* by default Logger instance created by logger() method will be assigned to class statically, meaning every instance of our controller will be writing to same log
* there is additional variable in `setLogger` method - static (false by default), which provides you ability to set logger for instance or specific class

### Global log
The Spiral `Logger` class provides one extra ability which can be very useful when working with the `Profiler` module - a global log. The global log can be turned on and off in the debug configuration, when it's turned on every raised log message will be aggregated in the`Debug` component and can be analyzed later.

Let's look at the global log configuration section in our `debug.php` file:

```php
    'globalLogging' => [
        'enabled' => true,
        'maxSize' => 1000
    ],
```

> The database component can log every query. The `Profiler` module will not only display but format and highlight these queries.

## Benchmarks
Spiral provides a simplistic trait which can help to record and profile different parts of your application. To use benchmarks we only need to connect BenchmarkTrait to your component or service (every Controller already has a `BenchmarkTrait`).

```php
class TestService extends Service
{
    use BenchmarkTrait;

    public function method()
    {
        //Opening benchmark
        $benchmark = $this->benchmark('doing something');

        sleep(2);

        //Closing, benchmark will return elapsed seconds when closed
        dump($this->benchmark($benchmark));
    }
}
```

> Attention, benchmarking will work only if `Spiral\Debug\BenchmarkerInterface` is defined in your application. The Profiler module will define this class automatically when being initialized.

The Benchmarking functionality may not look useful, however the most critical parts of Spiral (queries, adapter creations, storage manipulations) are covered with them, as result you can use the Profiler module and its panel to view a timeline of your application:

```php
public function index(Database $db)
{
    foreach ($db->getTables() as $table) {
        foreach ($table->schema()->getColumns() as $column) {
            dump($column);
        }
    }
}
```

![Profiler benchmarks](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler.png)

## Dumps
One additional feature can help you to view content of objects or variables without launching XCache, for this feature we will need instance of `Dumper`:

```php
public function index(Dumper $dumper, MemcacheStore $memcacheStore)
{
    $dumper->dump($memcacheStore);
}
```

The following code will dump the structure of $memcacheStore into output and will look like this:
![Dump](https://raw.githubusercontent.com/spiral/guide/master/resources/dumps.png)

You can also use the short "dump" function in Spiral environments (you'll probably notice it's in my examples):

```php
public function index(MemcacheStore $memcacheStore)
{
    dump($memcacheStore);
}
```

The Spiral `Dumper` fully supports the  `__debugInfo` [method](http://php.net/manual/en/language.oop5.magic.php) which allows you to dump only important information. In addition,  you can hide certain fields of your objects (for example, the container property since its dump can be pretty big) by adding an "@invisible" annotation.

```php
class TestService extends Service
{
    /**
     * Dumping this property will give us nothing, so we can hide it.
     *
     * @invisible
     * @var ContainerInterface
     */
    protected $container = null;
}
```

The Dumper and dump method, in addition to outputing content into the active buffer can send the dump info into different outputs:

```php
public function index(MemcacheStore $memcacheStore)
{
    //Output buffer
    dump($memcacheStore, Dumper::OUTPUT_ECHO);
    
    //Dump information in Spiral\Debug\Debugger log using print_r
    dump($memcacheStore, Dumper::OUTPUT_LOG);

    //Dump information in Spiral\Debug\Debugger log with nice formatting
    dump($memcacheStore, Dumper::OUTPUT_LOG_NICE);

    //Return dump as string
    $dump = dump($memcacheStore, Dumper::OUTPUT_RETURN);

}
```

## Profiler Module
You can achieve the best development performance by enabling the `Profiler` middleware in your primary middleware chain located in `application/config/http.php`. The profiler will display loaded classes, the application timeline using captured benchmarks and global log messages.
