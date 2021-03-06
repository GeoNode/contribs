# Use datastore shards in GeoNode

Many organizations have hundreds of layers uploaded to GeoNode. In such a case
using the default GeoNode configuration, with just one PostGIS database and one
GeoServer PostGIS store for all of the layers has several limitations, such as:

    * Layer upload and creation time become very large
        Layer upload and creation time tends to become very large when the PostGIS
        database starts containing many layers. We have seen cases where even 4/5
        minutes were needed for uploading a small sized shapefile).
        This issue is caused by the actual implementation of the PostgreSQL JDBC driver and
        has been reported in details in GeoServer bug GEOS-7533:
        https://osgeo-org.atlassian.net/browse/GEOS-7533
    * Large backups
        When data are not edited in GeoNode, it is easier to backup smaller
        PostgreSQL databases rather than a big one, where only a few tables have
        been changed since the last backup. When using a single database for all
        of the uploads this is not possible

These problems can be tackled using the Datastore Shards GeoNode contrib module,
which automatically creates new shards when some defined conditions changes.

## How to use the `datastore_shards` module

As a first thing, add the module in the INSTALLED_APPS section of the settings
file:

```Python
    INSTALLED_APPS = (
        ...,
        'geonode.contrib.datastore_shards',
        ...
    )
```

Here is a typical extract of the settings that must be used for GeoNode to use
the datatabase shards module:

```Python
    # SHARD DATABASES SETTINGS
    # SHARD_STRATEGY may be yearly, monthly, layercount
    SHARD_STRATEGY = 'layercount'
    SHARD_LAYER_COUNT = 100
    SHARD_PREFIX = 'wm_'
    SHARD_SUFFIX = ''
    DATASTORE_URL = 'postgis://user:password@localhost:5432/data'
```

Override the `geonode.geoserver.helpers` **datastore_db** method as follows:

```Python
    @property
    def datastore_db(self):
        """
        Returns the server's datastore dict or None.
        """
        if self.DATASTORE and settings.DATABASES.get(self.DATASTORE, None):
            datastore_dict = settings.DATABASES.get(self.DATASTORE, dict())
            if hasattr(settings, 'SHARD_STRATEGY'):
                if settings.SHARD_STRATEGY:
                    from geonode.contrib.datastore_shards.utils import get_shard_database_name
                    datastore_dict['NAME'] = get_shard_database_name()
            return datastore_dict
        else:
            return dict()
```

Now syncronize the module models with the database::

    python manage.py migrate datastore_shards


Now you are set and datastore shards will be used as soon as GeoNode is restarted.
The datastore_shards application will automatically create a new shard whenever it
is needed.

.. note:: The PostrgreSQL ROLE which is used (user in DATASTORE_URL) must have CREATEDB option assigned in order to be able to create the PostgreSQL shards.

## Database shards settings

### SHARD_STRATEGY

This setting can currently be set to 'yearly', 'monthly', 'layercount':

    * yearly
        a database shard is created and used each year.
        PostgreSQL database and GeoServer
        store name is in the form: prefix_YYYY_suffix
    * monthly
        a database shard is created and used each month.
        PostgreSQL database and GeoServer
        store name is in the form: prefix_YYYYMM_suffix
    * layercount
        a database shard is created when previous shard reaches a certain number
        of layers (which is set by the SHARD_LAYER_COUNT setting).
        PostgreSQL database and GeoServer store name is in the form:
        prefix_01234_suffix. 01234 is a progressive number starting from 0.

### SHARD_PREFIX and SHARD_SUFFIX

When these settings are used, a prefix and a suffix is appended to the name of the
shard.

For example, if SHARD_PREFIX = 'foo' and SHARD_SUFFIX = 'bar', when using
a SHARD_STRATEGY set to 'yearly', the shard for 2017 will be named
foo_2017_bar.

### SHARD_LAYER_COUNT

This setting is used when SHARD_STRATEGY is set to "layercount", and it represents
the maximum number a shard can contain before next shard is created and used.

Here is how it looks GeoServer when using a SHARD_STRATEGY set to "layercount"
and SHARD_LAYER_COUNT set to 3:

.. figure:: img/shards_001.png

As you can easily see a GeoServer PostGIS store is created every
time the store contains three layers. Each store links to a different PostGIS
database.
