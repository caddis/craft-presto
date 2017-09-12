# NOTICE: This plugin has been deprecated and is no longer in active development. Please download the [latest version](https://github.com/lewiscommunications/craft-presto) managed by [Lewis Communications](https://github.com/lewiscommunications).

![Presto](https://www.caddis.co/internal/repo/presto.svg)

Presto is a static file extension for the native [Craft cache](https://craftcms.com/docs/templating/cache). It works alongside standard Twig `{% cache %}` tag pairs and includes cache-busting features. Just like standard caching, Presto is automatic. Simply install, update your layouts, and then the cache will bust automatically as you create, update, or delete content.

## Setup Guide

### Step 1 -  Turn off element query caching

Turn off [element query caching](https://craftcms.com/docs/config-settings#cacheElementQueries) in your general config file. This will stop the `DeleteStaleTemplateCaches` task from running in the admin. Since Presto busts the entire cache when a new element is saved, element query caching is not necessary.

```php
'cacheElementQueries' => false
```

### Step 2 - Add cache tags

Presto lets Craft do the heavy lifting of calculating the elements within the template. As a result, all you need to do in your templates is pass the cache key returned from `craft.presto.cache` to the native cache tag pair. Presto will return a cache key that includes the host, group (if one is set), and path.

Note that the *entirety* of your template logic *must* be wrapped by the `cache` tags. In addition, it is recommended that you add the `globally` tag so that Craft does not overload the cache (i.e. query string requests).

```twig
{% cache globally using key craft.presto.cache %}
	{# Template Logic #}
{% endcache %}
```

### Parameters

```twig
{% craft.presto.cache({
	group: 'pjax',
	static: false
}) %}
```

| Parameter | Type    | Description                                                                                                                                                                  |
| --------- | ------- | -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| group     | string  | When set, the requested page will write into a sub-folder within the top-level cache directory. This is useful for pjax implementations where you load a separate template.  |
| static    | boolean | Setting to false will disable static caching for the request and fall back to native caching logic. The cache key will still be returned, but a static file won't be written.|

### Step 3 - Configure your server

Your host needs to check for matching static files before Craft handles the request. If the file exists, it's served statically. This block should typically be set immediately preceding the primary Craft "index.php" rewrite. Use these examples as a general guideline, your implementation may vary.

#### Apache

```apache
# Check Presto cache
RewriteCond %{REQUEST_FILENAME} !\.(css|eot|gif|ico|jpe?g|otf|png|svg|ttf|webp|woff2?)$ [NC]
RewriteCond %{REQUEST_METHOD} GET
RewriteCond %{DOCUMENT_ROOT}/cache/%{HTTP_HOST}/presto%{REQUEST_URI}/index.html -f
RewriteRule .* /cache/%{HTTP_HOST}/presto%{REQUEST_URI}/index.html [L,E=nocache:1]]

# Craft rewrite here
```

If you add a cache group, you'll need to add additional configuration. Below is an example of a pjax implementation:

```apache
RewriteCond %{REQUEST_FILENAME} !\.(css|eot|gif|ico|jpe?g|otf|png|svg|ttf|webp|woff2?)$ [NC]
RewriteCond %{HTTP:X-PJAX} true
RewriteCond %{REQUEST_METHOD} GET
RewriteCond %{DOCUMENT_ROOT}/cache/%{HTTP_HOST}/presto/pjax%{REQUEST_URI}/index.html -f
RewriteRule .* /cache/%{HTTP_HOST}/presto/pjax%{REQUEST_URI}/index.html [L,E=nocache:1]]
```

## Disable Caching

### Multi-enviroment

If you use a [multi-environment config](https://craftcms.com/docs/multi-environment-configs), set an arbitrary cache variable in your general config. Override this variable on environments where you don't want static caching (e.g. local development).

**General Config Variable**

```php
`cacheEnabled` => true
```

**Cache Tag**

```twig
{% cache globally using key craft.presto.cache if conf.cacheEnabled is defined and conf.cacheEnabled %}
	{# Template Logic #}
{% endcache %}
```

### Individual Templates

When using Presto the `for`, `until`, `if`, and `unless` parameters won't be respected on each request once the static html file is created. To disable the cache on individual templates, set a variable on the main cache tag. Override that variable on each template where you don't want static caching.

**Cache Tag**

```twig
{% cache globally using key craft.presto.cache if cacheEnabled is defined ? cacheEnabled : true %}
    <!doctype html>
    <html>
        <body>
            {{ block('content') }}
        </body>
    </html>
{% endcache %}
```

**Cache Template Override**

```twig
{% extends '[layout-template-path]' %}

{# Disable caching on this page #}
{% set cacheEnabled = false %}

{% block content %}
	{# page content #}
{% endblock %}
```

## Directory Structure

Presto resolves subdomain hosts automatically. Static html files are created inside a directory named after the requested host (i.e. coolwebsite.com, sub.coolwebsite.com). An additional directory called "presto" is created inside each host directory to avoid .htaccess filename conflicts. See below for an example cache file directory structure:

```
- cache
	- coolwebsite.com
		- presto
			- index.html
			- blog
				- index.html
```

## Purging the Cache

To purge the cache, navigate to the Presto plugin settings page (*Settings > Presto*) and click "Purge Cache" ([immediate](#immediate-purge)) or "Schedule Purge" ([cron](#cron-purge)).

**Note:** The Cron purge method does not clear the template cache. Remember to [purge the template cache](https://craftcms.com/docs/templating/cache#cache-clearing) before you schedule a purge.

## Purge Method

Presto provides two purge methods: immediate and cron.

### Immediate Purge

By default, Presto will purge the static cache and all related Craft template caches immediately. This only occurs in the server instance where the cache was cleared.

### Cron Purge

If you run Presto in an environment that spins up multiple server instances, set the [purgeMethod config](#config) to "cron". Set up a cron job on each server instance that runs the `presto check` [console command](https://craftcms.com/classreference/consolecommands/BaseCommand). The following example will run it every 10 minutes.

```bash
*/10 * * * *  /var/www/craft/app/etc/console/yiic presto check
```
	
## Disabled/Archived Entries

If an entry exists in the CMS but is not displayed on the site (e.g. status is disabled, entry is archived, etc.), enabling the entry will not clear any caches. Presto only clears related entries that are displayed on the site. In order to display your newly enabled entry, [purge the entire cache](#purging-the-cache).

## Config

Create a "presto.php" in the config folder (*craft > config*) file and configure as needed.

| Parameter   | Type   | Default                     | Description                                                                                                                              |
| ----------- | ------ | --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| purgeMethod | string | `immediate`                 | Changes how cache busting should be handled: immediately when Craft busts its cache, or via a cron job. Options: `immediate` and `cron`. |
| rootPath    | string | `$_SERVER['DOCUMENT_ROOT']` | Root public directory                                                                                                                    |

## Installation

1. Move the "presto" directory to "craft/plugins".
2. In the Craft admin, navigate to the Plugin section within Settings.
3. Click the Install button on the Presto entry.
4. Optionally change the default cache path in the Presto settings.
	* Note that you should exclude cache directory content from version control.
	
## Roadmap

- Display a list of cached pages in the admin
- Add ability to clear individual cached pages in the admin
- Warm cache after an entry is saved or created

## License

Copyright 2017 [Lewis Communications, LLC](http://www.lewiscommunications.com). Licensed under the [Apache License, Version 2.0](LICENSE).
