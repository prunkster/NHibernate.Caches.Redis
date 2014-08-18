NHibernate.Caches.Redis
=======================

This is a [Redis](http://redis.io/) based [ICacheProvider](http://www.nhforge.org/doc/nh/en/#configuration-optional-cacheprovider) 
for [NHibernate](http://nhforge.org/) written in C# using [ServiceStack.Redis](https://github.com/ServiceStack/ServiceStack.Redis).

Installation
------------

1. You can install using NuGet: `PM> Install-Package NHibernate.Caches.Redis`
2. Or build/install from source: `msbuild .\build\build.proj` and then look
   inside the `bin` directory.

Usage
-----

Configure NHibernate to use the custom cache provider:

```xml
<property name="cache.use_second_level_cache">true</property>
<property name="cache.use_query_cache">true</property>
<property name="cache.provider_class">NHibernate.Caches.Redis.RedisCacheProvider, 
    NHibernate.Caches.Redis</property>
```

Set the `ConnectionMultiplexer` on the `RedisCacheProvider`
*before* creating your `ISessionFactory`:

```csharp
// Or use your IoC container to wire this up.
var connectionMultiplexer = ConnectionMultiplexer.Connect("localhost:6379");
RedisCacheProvider.SetConnectionMultiplexer(connectionMultiplexer);

using (var sessionFactory = ...)
{
    // ...
}

// When your application exits:
connectionMultiplexer.Dispose();
```

Check out the `NHibernate.Caches.Redis.Sample` project to learn more.

Configuration
-------------

Inside of the `app/web.config`, a custom configuration section can be added to
configure each cache region. For example, this feature allows you to control the
expiration for a specific class that you cache.

```xml
<configSections>
  <section name="nhibernateRedisCache" type="NHibernate.Caches.Redis.RedisCacheProviderSection, NHibernate.Caches.Redis" />
</configSections>

<nhibernateRedisCache>
  <caches>
    <cache region="BlogPost" expiration="900" />
  </caches>
</nhibernateRedisCache>
```

Exception Handling
------------------

You may require that we gracefully continue to the database as if we "missed"
the cache if an exception occurs. By default, this is what happens.  

However, there is also the need of advanced exception handling scenarios. For 
example, imagine if you are using NHibernate in a web project and your Redis
server is unavailable. You may not want NHibernate to continue to timeout for
*every* NHibernate operation. You could do something similar to this:

```csharp
public class RequestRecoveryRedisCache : RedisCache
{
    public const string SkipNHibernateCacheKey = "__SkipNHibernateCache__";

    public RequestRecoveryRedisCache(string regionName, ConnectionMultiplexer connectionMultiplexer, RedisCacheProviderOptions options)
        : base(regionName, connectionMultiplexer, options)
    {

    }

    public override object Get(object key)
    {
        if (HasFailedForThisHttpRequest()) return null;
        return base.Get(key);
    }

    public override void Put(object key, object value)
    {
        if (HasFailedForThisHttpRequest()) return;
        base.Put(key, value);
    }

    public override void Remove(object key)
    {
        if (HasFailedForThisHttpRequest()) return;
        base.Remove(key);
    }

    public override void Clear()
    {
        if (HasFailedForThisHttpRequest()) return;
        base.Clear();
    }

    public override void Destroy()
    {
        if (HasFailedForThisHttpRequest()) return;
        base.Destroy();
    }

    public override void Lock(object key)
    {
        if (HasFailedForThisHttpRequest()) return;
        base.Lock(key);
    }

    public override void Unlock(object key)
    {
        if (HasFailedForThisHttpRequest()) return;
        base.Unlock(key);
    }

    private bool HasFailedForThisHttpRequest()
    {
        return HttpContext.Current.Items.Contains(SkipNHibernateCacheKey);
    }
}

public class RequestRecoveryRedisCacheProvider : RedisCacheProvider
{
    protected override RedisCache BuildCache(string regionName, IDictionary<string, string> properties, RedisCacheElement configElement, ConnectionMultiplexer connectionMultiplexer, RedisCacheProviderOptions options)
    {
        options.OnException = (e) =>
        {
            HttpContext.Current.Items[RequestRecoveryRedisCache.SkipNHibernateCacheKey] = true;
        };

        return new RequestRecoveryRedisCache(regionName, properties, configElement, connectionMultiplexer, options);

    }
}

```
Then, use `RequestRecoveryRedisCacheProvider` in your `web.config` settings.

Changelog
---------

**1.3.0**
- Add the `OnException` method for sub-classing the cache client and handling 
  exceptions.

**1.2.1**
- Update ServiceStack.Redis to 3.9.55.

**1.2.0**
- Allow the provider to gracefully continue when Redis is unavailable.
- Fix infinite loop when data in Redis was cleared.

**1.1.0**
- Added configuration section for customizing the cache regions.
- Added sample project.

**1.0.0**
- Initial release.

---

Contributors
------------

@MattiasJakobsson and @Tazer for helping switch over to `StackExchange.Redis`.

Happy caching!
