<!SLIDE[tpl=none] smaller>
# Puppet Module - Automic
    @@@puppet
	class automic::agent::peoplesoft (
	  $agent_name              = 'PSHOST',
	  $archive_location_agents = $automic::archive_location_agents,
	  $ae_server_host          = $automic::ae_server_host,
	  $system_name             = $automic::system_name,
	  $version                 = $automic::version,
	  $runtime_user_name       = $automic::runtime_user_name,
	  $runtime_group_name      = $automic::runtime_group_name,
	  $install_location        = $automic::install_location,
	  $agentdir                = $automic::agentdir,
	  $agent_config_common     = $automic::agent_config_common,
	  $oracle_client_location  = $automic::oracle_client_location,
	  $ensure_svc              = $automic::ensure_svc,
	  $ps_home_location        = $automic::ps_home_location,
	  $tools_version           = $automic::tools_version,
	  $domain_conn_pwd         = $automic::domain_conn_pwd,
	  $appserver               = undef,
	) inherits automic::params {

	  require automic::filesystem

	  $agentinstalldir        = "${agentdir}/peoplesoft"
	  $agentbindir            = "${agentinstalldir}/bin"
	  $agentconfdir           = $agentbindir

	  # Archive names will need to be maintained here if they change in the Automic delivered installers
	  # Binary names will need to be maintained here if they change in the Automic delivered installers
	  $agent_archive      = "${archive_location_agents}/peoplesoft/unix/linux/x86/UCXJPSX.tar.gz"
	  $agent_binary       = 'UCXJPSX'

	  file { $agentinstalldir :
		ensure => directory,
		owner  => $runtime_user_name,
		group  => $runtime_group_name,
		mode   => '0755',
	  }

	  archive { "Automic Agent Archive ${agent_name}":
		path         => $agent_archive,
		extract      => true,
		extract_path => $agentinstalldir,
		creates      => $agentbindir,
		require      => File[$agentdir],
	  }

	  exec { "Automic Agent Permissions ${agent_name}":
		command   => "chown -R ${runtime_user_name}:${runtime_group_name} ${agentinstalldir}",
		path      => $::path,
		unless    => "test -f ${agentconfdir}/${agent_name}.ini",
		subscribe => Archive["Automic Agent Archive ${agent_name}"],
	  }

	  # Install 32bit Java for Automic PeopleSoft agent
	  archive { '/app_software/oracle/java/7u80/jdk-7u80-linux-i586.tar.gz':
		extract      => true,
		extract_path => $install_location,
		creates      => "${install_location}/jdk1.7.0_80",
	  }

	  File_line {
		path    => "/home/${runtime_user_name}/.bashrc",
		require => User[$runtime_user_name]
	  }

	  file_line { 'JAVA_HOME':
		line => "export JAVA_HOME=${install_location}/jdk1.7.0_80  #Required for Automic PeopleSoft Agent.  32-bit Java",
	  }

	  file_line { 'LD_LIBRARY_PATH':
		line => "export LD_LIBRARY_PATH=${agentbindir}:${agentinstalldir}/lib:\$JAVA_HOME/jre/lib/i386/client:${oracle_client_location}/lib:/usr/lib:/lib:\$LD_LIBRARY_PATH",
	  }

	  file_line { 'PATH':
		line => "export PATH=${agentbindir}:${oracle_client_location}/bin:\$PATH",
	  }

	  # Create Agent configuration file from hiera data
	  file { "${agentconfdir}/${agent_name}.ini":
		ensure  => file,
		owner   => $runtime_user_name,
		group   => $runtime_group_name,
		source  => "${agentconfdir}/ucxjpsx.ori.ini",
		replace => 'no',
		mode    => '0644',
		require => Archive["Automic Agent Archive ${agent_name}"],
	  }

	  file_line { 'Remove delivered CP line':
		ensure  => absent,
		path    => "${agentconfdir}/${agent_name}.ini",
		line    => 'CP=uc4srv01:2217',
		require => File["${agentconfdir}/${agent_name}.ini"],
	  }

	  $agent_config_peoplesoft = {
		'GLOBAL'        => {
		  'name'    => $agent_name,
		},
		'PS'                      => {
		  'version'               => $tools_version,
		  'APPSERVER'             => "${appserver}:${jolt_port}",
		  'DOMAIN_CONNECTION_PWD' => "${domain_conn_pwd}",
		},
		'PRCS_SBB_JAVA' => {
		  'Version' => 'V1.03',
		  'classes' => "${ps_home_location}/appserv/classes/psjoa.jar:${ps_home_location}/appserv/classes/psmanagement.jar:UCXJPS84.jar",
		},
	  }

	  $config = {
		'path' => "${agentconfdir}/${agent_name}.ini",
		require => File["${agentconfdir}/${agent_name}.ini"],
		ensure  => present,
	  }

	  create_ini_settings($agent_config_common,     $config)

	  create_ini_settings($agent_config_peoplesoft, $config)

	  # Set up Automic Agent Service
	  file { '/etc/systemd/system/automic_agent_peoplesoft.service':
		content => template('automic/automic_agent_platform.service.erb'),
	  }

	  service { 'automic_agent_peoplesoft':
		ensure    => undef,
		enable    => true,
		hasstatus => true,
		subscribe => File["${agentconfdir}/${agent_name}.ini"],
	  }
	}
