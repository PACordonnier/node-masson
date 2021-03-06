
# SSSD Intall

## Install

Install the services defined by the "sssd.services" property. By default, the
following service: "sssd", "sssd-client", "pam\_krb5", "pam\_ldap" and
"sssd-tools". It also ensures SSSD is marked as a startup service.

    module.exports = header: 'SSSD Install', handler: ->
      {sssd} = @config

## Identities

By default, the "openldap-servers" package create the following entries:

```bash
cat /etc/passwd | grep sssd
sssd:x:996:994:User for sssd:/:/sbin/nologin
cat /etc/group | grep sssd
sssd:x:994:
```

      @call header: 'Identities', ->
        @system.group sssd.group
        @system.user sssd.user


## Packages

      @service
        name: 'sssd'
        startup: true # 2015/09 was 345
      @service name: 'sssd-client'
      @service name: 'pam_krb5'
      @service name: 'pam_ldap'
      @service name: 'authconfig'

## Certificates

Certificates are temporarily uploaded to the "/tmp" folder and registered with
the command `authconfig --update --ldaploadcacert={file}`.

      @call header: 'Certificates', handler: ->
        {certificates} = @config.sssd
        @each certificates, (options, callback) ->
          certificate = options.key
          hash = crypto.createHash('md5').update(certificate).digest('hex')
          filename = null
          @file.download
            source: certificate
            target: "/tmp/#{hash}"
            shy: true
          @system.execute # openssh is executed remotely
            cmd: "openssl x509 -noout -hash -in /tmp/#{hash}; rm -rf /tmp/#{hash}"
            shy: true
          , (err, _, stdout) ->
            filename = stdout.trim() unless err
          @call ->
            @file.download
              source: certificate
              target: "/etc/openldap/cacerts/#{filename}.0"
              # target: "#{config.TLS_CACERTDIR}/#{filename}.0"
              unless_exists: true
              mode: 0o444
          @then callback
      @service
        name: 'sssd'
        action: 'restart'
        if: -> @status -1

## Configure

Update the SSSD configuration file present in "/etc/sssd/sssd.conf" with the
values defined in the "sssd.config" property. The target file is by
default overwritten unless the "sssd.merge" is `true`.

      @call header: 'Configure', handler: ->
        {merge, config} = @config.sssd
        @file.ini
          content: config
          target: '/etc/sssd/sssd.conf'
          merge: merge
          mode: 0o0600
          backup: true
        options =
          # Configures the password, shadow, group, and netgroups services maps to use the SSSD module
          # https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Deployment_Guide/Configuration_Options-NSS_Configuration_Options.html
          sssd: true
          # Create home directories for users on their first login
          mkhomedir: true
          # To use an LDAP identity store, use the --enableldap. To use LDAP as the authentication source, use --enableldapauth.
          # https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Deployment_Guide/ch-Configuring_Authentication.html#sect-The_Authentication_Configuration_Tool-Command_Line_Version
          ldap: false
          ldapauth: false
          # Enable SSSD for system authentication
          # https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Deployment_Guide/Configuration_Options-PAM_Configuration_Options.html
          sssdauth: true # Allow ldap user to login with their password
        # Update configuration files with changed settings
        cmd = 'authconfig --update'
        for k, v of options
          cmd += if v then " --enable#{k}" else " --disable#{k}"
        {ldap_uri, ldap_search_base} = config['domain/default']?
        cmd += "--ldapserver=#{ldap_uri}" if ldap_uri
        cmd += "--ldapbasedn=#{ldap_search_base}" if ldap_search_base
        @system.execute
          cmd: cmd
          if: -> @status -1
        @service
          name: 'sssd'
          action: 'restart'
          if: -> @status -2

## Dependencies

    crypto = require 'crypto'
