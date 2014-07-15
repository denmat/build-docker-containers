require 'fileutils'
require 'logger'

def pre_check
  fail("need IMAGE_NAME env before running\nexport IMAGE_NAME='name' first").print_red unless ENV['IMAGE_NAME'] 
end

@log = Logger.new(STDOUT)

@current_dir = Dir.pwd
@image_dir = @current_dir + '/' + ENV['IMAGE_NAME'] if ENV['IMAGE_NAME']

@files_to_be_changed = {
  '/var/lock'       => [ 'lock' ],
  '/var/spool/mail' => [ 'mail' ],
  '/usr/bin/wall'   => [ 'tty' ],
  '/usr/bin/write'  => [ 'tty' ],
  '/dev/pts'        => [ 'root' ]
}

@mknods = {
  '/dev/console'    => [ 'root', '660', 'c 5 1' ],
  '/dev/tty'        => [ 'tty', '640', 'c 5 0' ],
  '/dev/tty0'       => [ 'tty', '640', 'c 4 0' ],
  '/dev/tty1'       => [ 'root', '600', 'c 4 1' ],
  '/dev/full'       => [ 'root', '666', 'c 1 7' ],
  '/dev/null'       => [ 'root', '666', 'c 1 3' ],
  '/dev/ptmx'       => [ 'tty', '666', 'c 5 2' ],
  '/dev/vcs'        => [ 'tty', '660', 'c 7 0' ],
  '/dev/random'     => [ 'root', '666', 'c 1 8' ],
  '/dev/urandom'    => [ 'root', '666', 'c 1 9' ],
  '/dev/zero'       => [ 'root', '666', 'c 1 5' ],
  '/dev/initctl'    => [ 'root', '666', 'p' ]
}

@sudoers = %Q(Defaults    env_keep += "SSH_AUTH_SOCKS"
vagrant ALL=NOPASSWD: ALL
docker ALL=NOPASSWD: ALL
)

@eth0_interface = %Q(DEVICE=eth0
BOOTPROT=dhcp
ONBOOT=yes
)

@pub_key = 'ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6NF8iallvQVp22WDkTkyrtvp9eWW6A8YVr+kz4TjGYe7gHzIw+niNltGEFHzD8+v1I2YJ6oXevct1YeS0o9HZyN1Q9qgCgzUFtdOKLv6IedplqoPkcmF0aYet2PkEDo3MlTBckFXPITAMzF8dJSIFo9D8HfdOV0IAdx4O7PtixWKn5y2hMNG0zQPyUecp4pzC6kivAIhyfHilFR61RGL+GPXQ2MWZWFYbAGjyiYJnAmCP3NOTd0jMZEnDkbUvxhMmBYSdETk1rRgm+R4LOzFUGaHqHDLKLX+FIPKcF96hrucXzcWyLbIbEgE98OHlnVYCzRdK8jlqm8tehUc9c9WhQ== vagrant insecure public key'

def files_setup
  %x[rm -rf #{@image_dir}/dev/*]
  @files_to_be_changed.each_pair do |file, group|
    FileUtils.mkdir("#{@image_dir}#{file}") unless File.exists?("#{@image_dir}#{file}")
    %x[chgrp #{group} #{@image_dir}#{file}]
  end
end

def make_nodes
 @mknods.each_pair do |node, value|
   %x[mknod -m #{value[1]} #{@image_dir}#{node} #{value[2]}]
   unless value[0] == 'root'
     %x[chgrp #{value[0]} #{@image_dir}#{node}]
   end
  end
end

namespace :setup do 
  desc 'setup file directories'
  task :dir_setup do
    pre_check
    @log.info("creating #{ENV['IMAGE_NAME']}")
    Dir.mkdir ENV['IMAGE_NAME'] unless File.exists?(ENV['IMAGE_NAME'])
  end
end

namespace :install do

  desc 'install core into image dir'
  task :core_image do
    pre_check
    @log.info("installing into " + @image_dir + "\n")
    Dir.chdir(@image_dir) do
      %x[yum groupinstall core -y --installroot #{@image_dir}]
    end
  end

  desc 'install sundry into image dir'
  task :sundry_packages do
    pre_check
    @log.info("installing sundry into " + @image_dir)
    Dir.chdir(@image_dir) do
      unless ENV['IMAGE_OTHERS'] 
        %x[yum install -y dhclient git openssh-server bind-utils wget nc dstat --installroot #{@image_dir}]
      else
        ENV['IMAGE_OTHERS'].split(/\s/).each do |p|
          @log.info("installing other package.. #{p}")
          %x[yum install #{p} --installroot #{@image_dir}]
        end
      end
    end
  end
end
  
namespace :post do
    desc 'setup eth0'
    task :config_eth0 do
      pre_check
      @log.info("setting eth0 interface")
      f = File.open("#{@image_dir}/etc/sysconfig/network-scripts/ifcfg-eth0", 'w')
      puts f << @eth0_interface
      f.close
      @log.info("setting resolv.conf to google dns")
      f = File.open("#{@image_dir}/etc/resolv.conf", 'w')
      puts f << "nameserver 8.8.8.8\nnameserver 8.8.4.4\noptions timeout:2 attempts:2 rotate"
      f.close
    end

    desc 'create devices, set perms'
    task :create_devices do
      @log.info("creating file in /dev")
      files_setup
      make_nodes
    end

    desc "setup default user"
    task :setup_user do
      pre_check
      users = [ 'docker', 'vagrant' ]
      users.each do |user|
        @log.info("adding user #{user}..")
        id_num = rand(10) + 400
        unless File.read("#{@image_dir}/etc/passwd").match(/^#{user}/) 
          f = File.open("#{@image_dir}/etc/passwd", 'a') 
          puts f << "#{user}:x:#{id_num}:#{id_num}:#{user} for docker,,,:/home/#{user}:/bin/bash\n"
          f.close
        end
        unless File.read("#{@image_dir}/etc/group").match(/^#{user}/)
          f = File.open("#{@image_dir}/etc/group", 'a') 
          puts f << "#{user}:x:#{id_num}:\n"
          f.close
        end
        unless File.read("#{@image_dir}/etc/shadow").match(/^#{user}/)
          f = File.open("#{@image_dir}/etc/shadow", 'a') 
          puts f << "#{user}:!:15958:0:99999:7:::\n"
          f.close
        end
        %x[mkdir -p #{@image_dir}/home/#{user}/.ssh ] unless File.exists?("#{@image_dir}/home/#{user}/.ssh")
        f = File.open("#{@image_dir}/home/#{user}/.ssh/authorized_keys", 'w', 0600) 
        puts f << @pub_key
        f.close
        %x[chown #{id_num}:#{id_num} -R #{@image_dir}/home/#{user} && chmod 700 #{@image_dir}/home/#{user}/.ssh]
        f = File.open("#{@image_dir}/etc/sudoers.d/containers", "w+", 0600)
          puts f << @sudoers
        f.close
      end
  end
end

desc "do it all"
task :build_it => ["setup:dir_setup", "install:core_image", "install:sundry_packages", "post:config_eth0", "post:create_devices", "post:setup_user"]

desc "put it into a tar ball"
task :tar_it do
  pre_check
  Dir.chdir(@image_dir) do
    %x[ rm -rf var/cache/yum ]
    %x[ tar -zcf ../#{ENV['IMAGE_NAME']}.#{Time.now.strftime("%s")}.tar.gz . ]
  end
end

desc "clean any existing image dir of the same name as #{ENV['IMAGE_NAME']}"
task :clean_up do
  pre_check
  %x[rm -rf #{@image_dir}]
end

desc "import into docker"
task :import_image, :image_name, :tag_name  do |t, args|
  %x[cat #{args[:image_name]} | sudo docker import - #{args[:tag_name]}]
end
  
task :default do
  mesg = %Q(This installs the core package group from oracle. 

It requires you to set an ENV var otherwise it will fail: export IMAGE_NAME=somename.

We also install the following sundry packages,
dhclient git openssh-server bind-utils wget nc dstat

To add different packages use the IMAGE_OTHERS ENV variable.

You will need to run sudo with the -E option like so:

export IMAGE_OTHERS='this that other' sudo -E rake install:sundry_packages)

   puts mesg
end

