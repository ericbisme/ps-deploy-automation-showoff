<!SLIDE[tpl=none] small>
# Puppet Module - GreyHeller ERP Firewall Web

    @@@puppet
        class greyheller::erpfirewall::web (
          $archive_location       = hiera('erp_firewall_basedir'),
          $psft_install_user_name = hiera('psft_install_user_name'),
          $psft_runtime_user_name = hiera('psft_runtime_user_name'),
          $erpfirewall_failopen   = hiera('erpfirewall_failopen', false),
          $ps_home_location       = hiera('ps_home_location'),
          $ps_config_home         = hiera('ps_config_home'),
          $pia_domain_name        = hiera('pia_domain_name'),
        ){

          exec { 'erpfirewall_web':
            command => "/usr/bin/su -m -s /bin/bash - ${psft_runtime_user_name} -c \"${archive_location}/WebServer/Unix/gh_firewall_web.bin \
                       ${ps_config_home} ${pia_domain_name}\"",
            creates => "${ps_config_home}/webserv/${pia_domain_name}/applications/peoplesoft/PORTAL.war/WEB-INF/gsdocs",
          }

          file { "${ps_config_home}/webserv/${pia_domain_name}/applications/peoplesoft/PORTAL.war/WEB-INF/lib/psjoa.jar" :
            source  => "${ps_home_location}/appserv/classes/psjoa.jar",
            owner   => $psft_runtime_user_name,
            mode    => '0755',
            require => Exec['erpfirewall_web'],
          }

          augeas { 'erp-failopen':
            lens    => 'Xml.lns',
            incl    => "${ps_config_home}/webserv/${pia_domain_name}/applications/peoplesoft/PORTAL.war/WEB-INF/web.xml",
            context => "/files/${ps_config_home}/webserv/${pia_domain_name}/applications/peoplesoft/PORTAL.war/WEB-INF/web.xml/web-app",
            changes => "set filter[1]/init-param/param-value/#text ${erpfirewall_failopen}",
            onlyif  => 'get filter[1]/filter-name/#text == gs_erp_firewall',
            require => Exec['erpfirewall_web'],
          }
        }
