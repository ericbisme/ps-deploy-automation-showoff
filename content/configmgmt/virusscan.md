<!SLIDE[tpl=none]>
# Puppet Module - VirusScan.xml

    @@@ puppet
    # Place VirusScan.xml in PORTAL.war and PSIGW.war
    class ps_virusscan (
      $ps_config_home          = hiera('ps_config_home'),
      $pia_domain_name         = hiera('pia_domain_name'),
      $psft_runtime_user_name  = hiera('psft_runtime_user_name'),
      $psft_runtime_group_name = hiera('psft_runtime_group_name'),
      $squidproxy_domain       = $ps_virusscan::params::squidproxy_domain,
    ) inherits ps_virusscan::params {
    
      $path_start = "${ps_config_home}/webserv/${pia_domain_name}/ \
                     applications/peoplesoft"
      $path_end   = 'WEB-INF/classes/psft/pt8/virusscan'
      $servlets   = ['PSIGW.war', 'PORTAL.war']

      $servlets.each |String $servlet| {
        file { "${path_start}/${servlet}/${path_end}/VirusScan.xml" :
          content => template('ps_virusscan/VirusScan.xml.erb'),
          owner   => $psft_runtime_user_name,
          group   => $psft_runtime_group_name,
        }
      }
    }
