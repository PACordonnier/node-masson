
# Anaconda Install

Install anaconda.

    module.exports = header: 'Anaconda Install', handler: ->
      {anaconda} = @config
      @each anaconda.python_version, (options) ->
        version = options.key
        @call unless_exec: "#{anaconda.install_dir}/python#{version}/bin/python --version 2>&1 | grep #{anaconda.version}", handler: ->
          script = "#{anaconda.tmp_dir}/Anaconda#{version}-#{anaconda.version}-Linux-x86_64.sh"
          @system.mkdir anaconda.tmp_dir
          @system.mkdir anaconda.install_dir
          @system.mkdir "#{anaconda.install_dir}/python#{version}"
          @file.download
            source: anaconda.source["python#{version}"]
            target: script
            md5: true
          @system.chmod
            target: script
            mode: 0o755
          @system.execute
            cmd: "#{script} -b -f -p #{anaconda.install_dir}/python#{version}"
          @system.remove target: script
