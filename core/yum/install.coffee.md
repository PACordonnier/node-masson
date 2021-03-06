
# YUM Install

    module.exports = header: 'YUM Install', handler: (options) ->

## Locked

Make sure Yum isnt already running.

      @call header: 'Locked', shy: true, handler: (_, callback) ->
        pidfile_running @options.ssh, '/var/run/yum.pid', (err, running) ->
          err = Error 'Yum is already running' if running
          callback err

## Configuration

Read the existing configuration in '/etc/yum.conf', 
merge server configuration and write the content back.

More information about configuring the proxy settings 
is available on [the centos website](http://www.centos.org/docs/5/html/yum/sn-yum-proxy-server.html)

      @file.ini
        header: 'Configuration'
        content: options.config
        target: '/etc/yum.conf'
        merge: true
        backup: true

## Yum Repositories

Upload the YUM repository file definition present in 
"options.repo" to the yum repository directory 
in "/etc/yum.repos.d"

      @tools.repo
        if: options.source?
        header: 'Repo'
        source: options.source
        update: options.update
        target: '/etc/yum.repos.d/centos.repo'
        clean: 'CentOS*'

## Custom Repositories

Allow administrators to upload additional repos.

      @call
        header: 'Custom Repos',
        if: options.additional_repos
      , ->
        for name, repo of options.additional_repos
          @tools.repo
            if: repo.source?
            source: repo.source
            update: repo.update
            target: repo.target
            name: repo.name
            clean: repo.clean

## Epel

Install the Epel repository. This is by default activated and the repository is
deployed by installing the "epel-release" package. It may also be installed from
an url by defining the "yum.epel.url" property. To disable Epel, simply set the
property "yum.epel" to false.

      @call
        header: 'Epel'
        if: options.epel?.enabled
      , ->
        epel_rpm_tmp = '/tmp/epel-release.rpm'
        @call
          if: options.epel.url?
          unless_exec: 'rpm -qa | grep epel-release'
          timeout: 100000
        , ->
          @file.download
            source: options.epel.url
            target: epel_rpm_tmp
            shy: true
          @system.execute
            cmd: "rpm -Uvh #{epel_rpm_tmp}" 
          @system.remove
            target: epel_rpm_tmp
            shy: true
        @tools.repo
          if: options.epel.source?
          source: options.epel.source
          target: '/etc/yum.repos.d/epel.repo'
          clean: 'epel*'
        @service
          name: 'epel-release'

## User Packages

      @service (
        header: "Install #{name}"
        name: name
        if: active
      ) for name, active of options.packages

## Dependencies

    glob = require 'glob'
    pidfile_running = require 'nikita/lib/misc/pidfile_running'
