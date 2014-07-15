# Simple Rakefile to build Redhat based docker (lxc) containers

Usage:
  * export IMAGE_NAME='name_of_container' sudo -E rake build_it
  * sudo -E rake tar_it
  * sudo -E rake clean_up

Requires yum installed and generally expects to build from whatever repos are in /etc/yum.repos.d/
