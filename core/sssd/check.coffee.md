
    module.exports = header: 'SSSD Check', handler: ->

## Check NSS

Check if NSS is correctly configured by executing the command `getent passwd
$user`. The command is only executed if a test user is defined by the
"sssd.test_user" property.

      @call
        header: 'NSS'
        if: -> @config.sssd.test_user
        handler: ->
          {test_user} = ctx.config.sssd
          @system.execute
            cmd: "getent passwd #{test_user}"

## Check PAM

Check if PAM is correctly configured by executing the command
`sh -l $user -c 'whoami'`. This is only executed if a test
user is defined by the "sssd.test_user" property.

      @call
        header: 'PAM'
        if: -> @config.sssd.test_user
        handler: ->
          {test_user} = ctx.config.sssd
          @system.execute
            cmd: "su -l #{test_user} -c 'whoami'"
