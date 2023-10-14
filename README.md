<p align="center">
    <a href="https://github.com/mastertoan" target="_blank">
        <img src="https://avatars2.githubusercontent.com/u/12951949" height="100px">
    </a>
    <h1 align="center">Crontab Extension for Yii 2</h1>
    <br>
</p>

This extension adds [Crontab](http://en.wikipedia.org/wiki/Crontab) setup support.

For license information check the [LICENSE](LICENSE.md)-file.

[![Latest Stable Version](https://poser.pugx.org/mastertoan/crontab/v/stable.png)](https://packagist.org/packages/mastertoan/crontab)
[![Total Downloads](https://poser.pugx.org/mastertoan/crontab/downloads.png)](https://packagist.org/packages/mastertoan/crontab)
[![Build Status](https://travis-ci.org/mastertoan/crontab.svg?branch=master)](https://travis-ci.org/mastertoan/crontab)


Requirements
------------

This extension requires Linux OS. 'crontab' should be installed and cron daemon should be running.


Installation
------------

The preferred way to install this extension is through [composer](http://getcomposer.org/download/).

Either run

```
php composer.phar require --prefer-dist mastertoan/crontab
```

or add

```json
"mastertoan/crontab": "*"
```

to the require section of your composer.json.


Usage
-----

You can setup cron tab using [[mastertoan\crontab\CronTab]], for example:

```php
use mastertoan\crontab\CronTab;

$cronTab = new CronTab();
$cronTab->setJobs([
    [
        'min' => '0',
        'hour' => '0',
        'command' => 'php /path/to/project/yii some-cron',
    ],
    [
        'line' => '0 0 * * * php /path/to/project/yii another-cron'
    ]
]);
$cronTab->apply();
```

You may specify particular cron job using [[mastertoan\crontab\CronJob]] instance, for example:

```php
use mastertoan\crontab\CronJob;
use mastertoan\crontab\CronTab;

$cronJob = new CronJob();
$cronJob->min = '0';
$cronJob->hour = '0';
$cronJob->command = 'php /path/to/project/yii some-cron';

$cronTab = new CronTab();
$cronTab->setJobs([
    $cronJob
]);
$cronTab->apply();
```

> Tip: [[mastertoan\crontab\CronJob]] is a descendant of [[yii\base\Model]] and have built in validation rules for each
  parameter, thus it can be used in the web forms to create a cron setup interface.


## Parsing cron jobs <span id="parsing-cron-jobs"></span>

[[mastertoan\crontab\CronJob]] composes a cron job line, like:

```
0 0 * * * php /path/to/my/project/yii some-cron
```

However it can also parse such lines filling up own internal attributes. For example:

```php
use mastertoan\crontab\CronJob;

$cronJob = new CronJob();
$cronJob->setLine('0 0 * * * php /path/to/my/project/yii some-cron');

echo $cronJob->min; // outputs: '0'
echo $cronJob->hour; // outputs: '0'
echo $cronJob->day; // outputs: '*'
echo $cronJob->month; // outputs: '*'
echo $cronJob->command; // outputs: 'php /path/to/my/project/yii some-cron'
```


## Merging cron jobs <span id="merging-cron-jobs"></span>

Method [[mastertoan\crontab\CronTab::apply()]] adds all specified cron jobs to crontab, keeping already exiting cron jobs
intact. For example, if current crontab is following:

```
0 0 * * * php /path/to/my/project/yii daily-cron
```

running following code:

```php
use mastertoan\crontab\CronTab;

$cronTab = new CronTab();
$cronTab->setJobs([
    [
        'min' => '0',
        'hour' => '0',
        'weekDay' => '5',
        'command' => 'php /path/to/project/yii weekly-cron',
    ],
]);
$cronTab->apply();
```

will produce following crontab:

```
0 0 * * * php /path/to/my/project/yii daily-cron
0 0 * * 5 php /path/to/my/project/yii weekly-cron
```

While merging crontab lines [[mastertoan\crontab\CronTab::apply()]] avoids duplication, so same cron job will never
be added twice. However while doing this, lines are compared by **exact** match, inlcuding command and time pattern.
If same command added twice with different time pattern - 2 crontab records will be present.
For example, if current crontab is following:

```
0 0 * * * php /path/to/my/project/yii some-cron
```

running following code:

```php
use mastertoan\crontab\CronTab;

$cronTab = new CronTab();
$cronTab->setJobs([
    [
        'min' => '15',
        'hour' => '2',
        'command' => 'php /path/to/project/yii some-cron',
    ],
]);
$cronTab->apply();
```

will produce following crontab:

```
0 0 * * * php /path/to/my/project/yii some-cron
15 2 * * * php /path/to/my/project/yii some-cron
```

You may interfere in merging process using [[mastertoan\crontab\CronTab::$mergeFilter]], which allows indicating
those existing cron jobs, which should be removed while merging. Its value could be a plain string - in this case
all lines, which contains this string as a substring will be removed, or a PHP callable of the following signature:
`bool function (string $line)` - if function returns `true` the line should be removed.
For example, if current crontab is following:

```
0 0 * * * php /path/to/my/project/yii some-cron
```

running following code:

```php
use mastertoan\crontab\CronTab;

$cronTab = new CronTab();
$cronTab->mergeFilter = '/path/to/project/yii'; // filter all invocation of Yii console
$cronTab->setJobs([
    [
        'min' => '15',
        'hour' => '2',
        'command' => 'php /path/to/project/yii some-cron',
    ],
]);
$cronTab->apply();
```

will produce following crontab:

```
15 2 * * * php /path/to/my/project/yii some-cron
```


## Extra lines setup <span id="extra-lines-setup"></span>

Crontab file may content additional lines beside jobs specifications. It may contain comments or extra
shell configuration. For example:

```
# this crontab created by my application
SHELL=/bin/sh
PATH=/usr/bin:/usr/sbin

0 0 * * * php /path/to/my/project/yii some-cron
```

You may append such extra lines into the crontab using [[mastertoan\crontab\CronTab::$headLines]]. For example:

```php
use mastertoan\crontab\CronTab;

$cronTab = new CronTab();
$cronTab->headLines = [
    '# this crontab created by my application',
    'SHELL=/bin/sh',
    'PATH=/usr/bin:/usr/sbin',
];
$cronTab->setJobs([
    [
        'min' => '0',
        'hour' => '0',
        'command' => 'php /path/to/project/yii some-cron',
    ],
]);
$cronTab->apply();
```

> Note: usage of the `headLines` may produce unexpected results, while merging crontab with existing one.


## User setup <span id="user-setup"></span>

Each Linux system user has his own crontab. Ownership of crontab affected by this extension is determined by the user
running the PHP script. For the web application it is usually 'apache', for the console application - current local user
or root. Thus crontab application from web application and from console application will produce 2 separated cron jobs list
for 2 different system users.

You may explicitly setup name of the user whose crontab is to be affected via [[mastertoan\crontab\CronTab::$username]].
For example:

```php
use mastertoan\crontab\CronTab;

$cronTab = new CronTab();
$cronTab->username = 'www-data'; // apply crontab for 'www-data' user
$cronTab->setJobs([
    [
        'min' => '0',
        'hour' => '0',
        'command' => 'php /path/to/project/yii some-cron',
    ],
]);
$cronTab->apply();
```

However, this will work only in case PHP script is running from privileged user (e.g. 'root').
