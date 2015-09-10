
# SSSD Intall

    exports = module.exports = []
    exports.push 'masson/bootstrap'
    exports.push 'masson/core/yum'
    exports.push 'masson/core/openldap_client'
    # exports.push require('./').configure

## Install

Install the services defined by the "sssd.services" property. By default, the
following service: "sssd", "sssd-client", "pam\_krb5", "pam\_ldap" and
"sssd-tools". It also ensures SSSD is marked as a startup service.

    exports.push name: 'SSSD # Install', timeout: -1, handler: ->
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

    exports.push name: 'SSSD # Certificates', handler: ->
      {certificates} = @config.sssd
      for certificate in certificates then do =>
        hash = crypto.createHash('md5').update(certificate).digest('hex')
        filename = null
        @upload
          source: certificate
          destination: "/tmp/#{hash}"
          shy: true
        @execute # openssh is executed remotely
          cmd: "openssl x509 -noout -hash -in /tmp/#{hash}; rm -rf /tmp/#{hash}"
          shy: true
        , (err, _, stdout) ->
          filename = stdout.trim() unless err
        @call ->
          @upload 
            source: certificate
            destination: "/etc/openldap/cacerts/#{filename}.0"
            # destination: "#{config.TLS_CACERTDIR}/#{filename}.0"
            not_if_exists: true


## Configure

Update the SSSD configuration file present in "/etc/sssd/sssd.conf" with the
values defined in the "sssd.config" property. The destination file is by
default overwritten unless the "sssd.merge" is `true`.

    exports.push name: 'SSSD # Configure', timeout: -1, handler: ->
      {merge, config} = @config.sssd
      @ini
        content: config
        destination: '/etc/sssd/sssd.conf'
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
      @execute
        cmd: cmd
        if: -> @status -1
      @service
        name: 'sssd'
        action: 'restart'
        if: -> @status -2

## Clean "sssd" Cache

If the command `sss_cache -E` fail, the cache may be manually removed with:

```
cp -rp /var/lib/sss/db /var/lib/sss/db.bck
rm -rf /var/lib/sss/db/*
service sssd restart
```

## Dependencies

    crypto = require 'crypto'
    each = require 'each'