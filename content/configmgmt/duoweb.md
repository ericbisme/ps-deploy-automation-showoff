<!SLIDE[tpl=none]>
# Puppet Module - DuoWeb

    @@@ puppet
    # Place Duo Security DuoWeb components
    class cu_duoweb::duoweb {
      $archive_location        = hiera('duoweb_archive_location')
      $ps_custhome_hiera       = hiera('ps_cust_home', '')
      $ps_custhome_location    = $ps_custhome_hiera['location']
      $psft_runtime_user_name  = hiera('psft_runtime_user_name')
      $psft_runtime_group_name = hiera('psft_runtime_group_name')
    
      file {"${ps_custhome_location}/appserv/classes/DuoWeb-1.0.jar":
        ensure => file,
        source => "${archive_location}/1.0/DuoWeb-1.0.jar",
        owner  => $psft_runtime_user_name,
        group  => $psft_runtime_group_name,
        mode   => '0755',
      }
    }
