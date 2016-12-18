<!SLIDE[tpl=none] smaller>
# Puppet Module - Microfocus COBOL

    @@@puppet
	class microfocus_cobol::compiler (
	  $ensure            = hiera('ensure', 'present'),
	  $compiler_pkg      = undef,
	  $ulp_pkg           = undef,
	  $archive_location  = undef,
	  $version           = undef,
	  $cobol             = undef,
	  $license_file      = undef,
	) {

	  $compiler_archive = "${archive_location}/serverexpress/${version}/${compiler_pkg}"
	  $ulp_archive      = "${archive_location}/ulp/${ulp_pkg}"

	  $deploy_location  = $cobol['location']
	  $license_sn       = $cobol['license_serial_number']
	  $license_key      = $cobol['license_key']

	  # Prereq: 32-bit Libs requried for COBOL Compiler installation
	  # Prereq: expect to run COBOL installation scripts
	  package { 'glibc.i686':         ensure => present }
	  package { 'libgcc.i686':        ensure => present }
	  package { 'libstdc++.i686':     ensure => present }
	  package { 'expect':             ensure => present }
	  package { 'gcc':                ensure => present }

	  exec { 'Create Microfocus Install Directory Structure':
		command => "/usr/bin/mkdir -p ${deploy_location}",
		creates => $deploy_location,
	  }

	  archive { $compiler_archive :
		extract      => true,
		extract_path => $deploy_location,
		creates      => "${deploy_location}/install",
		require      => Exec['Create Microfocus Install Directory Structure'],
	  }

	  file { '/tmp/cobol_setup_install.exp':
		source => 'puppet:///modules/microfocus_cobol/cobol_setup_install.exp',
		mode   => '0755',
	  }

	  file { '/tmp/cobol_license_install.exp':
		source => 'puppet:///modules/microfocus_cobol/cobol_license_install.exp',
		mode   => '0755',
	  }

	  file { "${deploy_location}/install":
		mode    => '0755',
		require => Archive[$compiler_archive],
	  }

	  file_line { 'remove_more_line':
		ensure => absent,
		path   => "${deploy_location}/install",
		line   => 'more docs/env.txt',
	  }

	  exec { 'cobol_svr_install':
		command     => "/tmp/cobol_setup_install.exp ${deploy_location}",
		environment => ["COBDIR=${deploy_location}", 'LIBPATH=$COBDIR/lib:$LIBPATH', 'LD_LIBRARY_PATH=$COBDIR/lib:/usr/lib64:/usr/lib:$LD_LIBRARY_PATH'],
		cwd         => $deploy_location,
		creates     => "${deploy_location}/mflmf",
		require     => [ Archive[$compiler_archive], Package['expect'], File['/tmp/cobol_setup_install.exp'], File_line['remove_more_line'] ],
	  }

	  exec { 'cobopt_gcc_lib':
		command => "/usr/bin/sed -i \"s/^set GCC_LIB=.*/set GCC_LIB=\/usr\/lib\/gcc\/x86_64-redhat-linux\/`/bin/rpm -q --queryformat %{VERSION} gcc`/\" ${deploy_location}/etc/cobopt64",
		require => Exec['cobol_svr_install'],
		unless  => "/usr/bin/grep GCC_LIB=\/usr\/lib\/gcc\/x86_64-redhat-linux\/`/bin/rpm -q --queryformat %{VERSION} gcc` ${deploy_location}/etc/cobopt64 2>&1>/dev/null",
	  }

	  # ULP Should only be installed in production.
	  # If installed it must be installed before the development/compile licenses
	  if $facts[region] == 'prod' {

		file { "${deploy_location}/ULP":
		  ensure  => directory,
		  mode    => '0755',
		  require => Exec['cobol_svr_install'],
		}

		archive { $ulp_archive :
		  extract      => true,
		  extract_path => "${deploy_location}/ULP",
		  creates      => "${deploy_location}/ULP/psauto64",
		  require      => [Exec['cobol_svr_install'], File["${deploy_location}/ULP"]],
		  before       => Exec['cobol_lic_install'],
		}

		exec{ 'cobol_ulp_install':
		  command     => "${deploy_location}/ULP/psauto64",
		  environment => ["COBDIR=${deploy_location}", 'LIBPATH=$COBDIR/lib:$LIBPATH', 'LD_LIBRARY_PATH=$COBDIR/lib:/usr/lib64:/usr/lib:$LD_LIBRARY_PATH'],
		  cwd         => "${deploy_location}/ULP",
		  require     => Archive[$ulp_archive],
		  before      => Exec['cobol_lic_install'],
		  unless      => "/bin/bash -c 'COBDIR=${deploy_location} LIBPATH=\$COBDIR/lib LD_LIBRARY_PATH=\$COBDIR/lib ${deploy_location}/bin/MFLicenseCheck | grep Server.for.COBOL.*Concurrent.Use.Trans.*Uses.*5000 2>&1>/dev/null'",
		}
	  }

	  exec { 'cobol_lic_install':
		command     => "/tmp/cobol_license_install.exp ${deploy_location} ${license_sn} \"${license_key}\"",
		environment => ["COBDIR=${deploy_location}", 'LIBPATH=$COBDIR/lib:$LIBPATH', 'LD_LIBRARY_PATH=$COBDIR/lib:$LD_LIBRARY_PATH'],
		cwd         => "${deploy_location}/mflmf",
		returns     => ["0", "1"],
		require     => Exec['cobol_svr_install'],
		unless      => "/bin/bash -c 'COBDIR=${deploy_location} LIBPATH=\$COBDIR/lib LD_LIBRARY_PATH=\$COBDIR/lib ${deploy_location}/bin/MFLicenseCheck | grep Server.for.COBOL.*Developer.Test.License.*Use.*15 2>&1>/dev/null'",
	  }

	  file { '/etc/systemd/system/cobol_license_manager.service':
		content => template('microfocus_cobol/cobol_license_manager.service.erb'),
	  }

	  service { 'cobol_license_manager.service':
		ensure     => running,
		enable     => true,
		hasrestart => true,
		hasstatus  => false,
		require    => [File['/etc/systemd/system/cobol_license_manager.service'], Exec['cobol_lic_install'] ],
	  }
	}
