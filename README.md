# Orleans.Providers.EntityFramework
An Entity Framework Core implementation of Orleans Grain Storage.


## Usage

Nuget: https://www.nuget.org/packages/Orleans.Providers.EntityFramework/

or

```dotnet add package Orleans.Providers.EntityFramework --version 0.9.0	```

or 

```Install-Package Orleans.Providers.EntityFramework```


And configure the storage provider using SiloHostBuilder:

```
ISiloHostBuilder builder = new SiloHostBuilder();

builder.AddEfGrainStorage<FrogsDbContext>("ef");  

```

This requires your DbContext to be registered as well

```
services
    .AddDbContextPool<FatDbContext>(
        (sp, options) => {});
```
The GrainStorage will resolve and releases contexts per operation so you won't have many context in use.
Hence it's better to use the context pool provided in the entity framework package or use your own.

## Configuration

By default the provider will search for key properties on your data models that match your grain interfaces, 
but you can change the default behavior like so:

```
services.Configure<GrainStorageConventionOptions>(options =>
{
    options.DefaultGrainKeyPropertyName = "Id";
    options.DefaultPersistanceCheckPropertyName = "Id";
    options.DefaultGrainKeyExtPropertyName = "KeyExt";
});
```

**DefaultPersistanceCheckPropertyName** is used to check if a model needs to be inserted into the database or updated. 
The value of said property will be checked against the default value of the type.

The following sample model would work out of the box for a grain that implements IGrainWithGuidCompoundKey and requires no configuration:

```
public class Box {
  public Guid Id { get; set; }
  public string KeyExt { get; set; }
}
```

_If you use conventions (as described, configuring GrainStorageConventionOptions) you're context should contain DbSets for your models._

```
public DbSet<Box> Boxes { get; set; }
```

### Querying models using custom expressions
To configure a special model you can do:

```
public class SpecialBox {
  public long WeirdId { get; set; }
  public string Type { get; set; }
  public long ClusterIndexId { get; set; }
}

services
    .ConfigureGrainStorageOptions<FatDbContext, SpecialBoxGrain, SpecialBox>(
        options => options
            .UseQueryExpression(grainRef =>
            {
                long key = grainRef.GetPrimaryKeyLong(out string keyExt);
                return (box => box.WeirdId == key && box.Type == keyExt);
            })
    )
```

The **UseQueryExpression** method instructs the sotrage to use the provided expression to query the database.

### Loading additional data on read state
You can load additional data while reading the state. Using the SpecialBox model:

```
services
    .ConfigureGrainStorageOptions<FatDbContext, SpecialBoxGrain, SpecialBox>(
        options => options
            .UseQuery(context => context.SpecialBoxes.AsNoTracking()
                .Include(box => box.Gems)
                .ThenInclude(gems => gems.Map))
    )
```

### Using custom persistance check

When using Guids as primary keys you're most likely to add a cluster index that is auto incremented. 
That field can be used to check if the state is already inserted into the database or not:

```
services
    .ConfigureGrainStorageOptions<FatDbContext, SpecialBoxGrain, SpecialBox>(
        options => options
            .ConfigureIsPersisted(box => box.ClusterIndexId > 0)
    )
```

If you use different cluster indices than your primary keys you can configure the dafaults to write less configuration code:

```
services.Configure<GrainStorageConventionOptions>(options =>
{
    options.DefaultPersistanceCheckPropertyName = "ClusterIndexId";
});
```


## Known Issues

- As types has to be configured in dbcontext, arbitrary types can't use this provider. 
This specially causes issues with Orleans VersionStoreGrain internal grain, hence this GrainStorage can't 
be used as default grain storage. I'll handle that special case if I get the time needed.

- Etags support