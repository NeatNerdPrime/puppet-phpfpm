# puppet-phpfpm
[![Build Status](https://github.com/Slashbunny/puppet-phpfpm/actions/workflows/puppet.yml/badge.svg?branch=master)](https://github.com/Slashbunny/puppet-phpfpm/actions/workflows/puppet.yml)
[![Puppet Forge](https://img.shields.io/puppetforge/v/Slashbunny/phpfpm.svg)](https://forge.puppet.com/modules/Slashbunny/phpfpm)
[![Puppet Forge Downloads](https://img.shields.io/puppetforge/dt/Slashbunny/phpfpm.svg)](https://forge.puppet.com/modules/Slashbunny/phpfpm)
[![Puppet Forge Scores](https://img.shields.io/puppetforge/f/Slashbunny/phpfpm.svg)](https://forge.puppet.com/modules/Slashbunny/phpfpm)
## Overview

This module manages the php-fpm daemon and pools **only**. Managing PHP, PHP extensions, Composer, PEAR, PECL, nginx, Apache, etc are out of the scope of this module.

The module has been tested on Ubuntu, CentOS/RHEL, Arch Linux, Amazon AMI, FreeBSD, and OpenBSD. It should be able to easily support any distribution that has an official phpfpm package with a few configuration additions.

* `phpfpm` : Class that installs and configures php-fpm itself.
* `phpfpm::pool` : Definition used to configure a fpm pool

## Parameters

The name of the parameters mirror the name of the config variables in the php-fpm configuration file and the pool configuration file. However, be sure to replace periods with underscores, as puppet does not support parameter names with periods.

For example, if your pool configuration should set the `pm.status_path` option to "/mystatus", the `pm.max_requests` option to "900", and `chroot` to "/www", you would use the following parameters in your manifest:

```puppet
phpfpm::pool { 'mypool':
    chroot          => '/www',
    pm_status_path  => '/mystatus',
    pm_max_requests => 900,
}
```

Please see the php-fpm configuration file comments for detailed explanations about what each option does.

## Custom Parameters

`$phpfpm::poold_purge` : Delete all files in the pool.d folder that aren't managed by Puppet.

## Examples

**You must include the phpfpm class prior to configuring pools.**

Install php-fpm with default options and a default pool called 'www' (packaging defaults on Ubuntu).

```puppet
include phpfpm
```

Install php-fpm with non-default options:

```puppet
class { 'phpfpm':
    process_max => 20,
    log_level   => 'warning',
    error_log   => '/var/log/phpfpm.log',
}
```

Install php-fpm and remove the default pool that ships with Ubuntu:

```puppet
include phpfpm

phpfpm::pool { 'www':
    ensure => 'absent',
}
```

Do the same and add a pool named "main":

```puppet
include phpfpm

phpfpm::pool { 'www':
    ensure => 'absent',
}

# TCP pool using 127.0.0.1, port 9000, upstream defaults
phpfpm::pool { 'main': }
```

Alternatively, use the purge flag to remove all non-managed pools, then create a pool named "main":

```puppet
class { 'phpfpm':
    poold_purge => true,
}

# TCP pool using 127.0.0.1, port 9000, upstream defaults
phpfpm::pool { 'main': }
```

Use a custom template file, which you must provide, to generate the main
FPM configuration file:

```puppet
class { 'phpfpm':
    config_template_file => 'site/phpfpm/my-php-fpm.conf.erb',
}
```

Add a few custom pools with advanced options:

```puppet
class { 'phpfpm':
    poold_purge => true,
}

# Pool running as a different user
phpfpm::pool { 'user_bob':
    listen => '127.0.0.1:9999',
    user   => 'bob',
    group  => 'users',
}

# Pool with dynamic process manager, TCP socket
phpfpm::pool { 'main':
    listen                 => '127.0.0.1:9000',
    listen_allowed_clients => '127.0.0.1',
    pm                     => 'dynamic',
    pm_max_children        => 10,
    pm_start_servers       => 4,
    pm_min_spare_servers   => 2,
    pm_max_spare_servers   => 6,
    pm_max_requests        => 500,
    pm_status_path         => '/status',
    ping_path              => '/ping',
    ping_response          => 'pong',
    env                    => {
        'ODBCINI' => '"/etc/odbc.ini"',
    },
    php_admin_flag         => {
        'expose_php' => 'Off',
    },
    php_admin_value        => {
        'max_execution_time' => '300',
    },
}

# Pool using a custom template file that you provide, rather than the stock template
phpfpm::pool { 'www':
    listen             => '127.0.0.1:9001',
    pool_template_file => 'site/phpfpm/mypool.conf.erb',
}

```

Notify the php-fpm daemon of your custom php configuration changes:

```puppet
class { 'phpfpm':
    poold_purge => true,
}

phpfpm::pool { 'main': }

file { '/etc/php5/conf.d/pdo.ini':
    ensure  => 'present',
    content => template('web/pdo.ini.erb'),
    notify  => Class['phpfpm::service'],
}
```

