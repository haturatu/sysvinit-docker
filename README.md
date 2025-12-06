# sysvinit-docker
This script is a start-stop-daemon script for starting Docker on sysvinit systems when using cgroups v2.

```bash
sudo mv docker /etc/init.d/docker
```

```bash
$ sdiff docker /etc/init.d/docker
#!/bin/sh                                                       #!/bin/sh
set -e                                                          set -e

### BEGIN INIT INFO                                             ### BEGIN INIT INFO
# Provides:           docker                                    # Provides:           docker
# Required-Start:     $syslog $remote_fs                        # Required-Start:     $syslog $remote_fs
# Required-Stop:      $syslog $remote_fs                        # Required-Stop:      $syslog $remote_fs
# Should-Start:       cgroupfs-mount cgroup-lite                # Should-Start:       cgroupfs-mount cgroup-lite
# Should-Stop:        cgroupfs-mount cgroup-lite                # Should-Stop:        cgroupfs-mount cgroup-lite
# Default-Start:      2 3 4 5                                   # Default-Start:      2 3 4 5
# Default-Stop:       0 1 6                                     # Default-Stop:       0 1 6
# Short-Description:  Create lightweight, portable, self-suff   # Short-Description:  Create lightweight, portable, self-suff
# Description:                                                  # Description:
#  Docker is an open-source project to easily create lightwei   #  Docker is an open-source project to easily create lightwei
#  self-sufficient containers from any application. The same    #  self-sufficient containers from any application. The same
#  developer builds and tests on a laptop can run at scale, i   #  developer builds and tests on a laptop can run at scale, i
#  VMs, bare metal, OpenStack clusters, public clouds and mor   #  VMs, bare metal, OpenStack clusters, public clouds and mor
### END INIT INFO                                               ### END INIT INFO

export PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/sbin:/us   export PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/sbin:/us

BASE=docker                                                     BASE=docker

# modify these in /etc/default/$BASE (/etc/default/docker)      # modify these in /etc/default/$BASE (/etc/default/docker)
DOCKERD=/usr/sbin/dockerd                                       DOCKERD=/usr/sbin/dockerd
# This is the pid file managed by docker itself                 # This is the pid file managed by docker itself
DOCKER_PIDFILE=/var/run/$BASE.pid                               DOCKER_PIDFILE=/var/run/$BASE.pid
# This is the pid file created/managed by start-stop-daemon     # This is the pid file created/managed by start-stop-daemon
DOCKER_SSD_PIDFILE=/var/run/$BASE-ssd.pid                       DOCKER_SSD_PIDFILE=/var/run/$BASE-ssd.pid
DOCKER_LOGFILE=/var/log/$BASE.log                               DOCKER_LOGFILE=/var/log/$BASE.log
DOCKER_OPTS=                                                    DOCKER_OPTS=
DOCKER_DESC="Docker"                                            DOCKER_DESC="Docker"

# Get lsb functions                                             # Get lsb functions
. /lib/lsb/init-functions                                       . /lib/lsb/init-functions

if [ -f /etc/default/$BASE ]; then                              if [ -f /etc/default/$BASE ]; then
        . /etc/default/$BASE                                            . /etc/default/$BASE
fi                                                              fi

# Check docker is present                                       # Check docker is present
if [ ! -x $DOCKERD ]; then                                      if [ ! -x $DOCKERD ]; then
        log_failure_msg "$DOCKERD not present or not executab           log_failure_msg "$DOCKERD not present or not executab
        exit 1                                                          exit 1
fi                                                              fi

fail_unless_root() {                                            fail_unless_root() {
        if [ "$(id -u)" != '0' ]; then                                  if [ "$(id -u)" != '0' ]; then
                log_failure_msg "$DOCKER_DESC must be run as                    log_failure_msg "$DOCKER_DESC must be run as
                exit 1                                                          exit 1
        fi                                                              fi
}                                                               }

cgroupfs_mount() {                                              cgroupfs_mount() {
        # see also https://github.com/tianon/cgroupfs-mount/b |         # Check if cgroups v2 is already mounted
        if grep -v '^#' /etc/fstab | grep -q cgroup \         |         if mountpoint -q /sys/fs/cgroup; then
                || [ ! -e /proc/cgroups ] \                   |                 # Check if it's cgroups v2 (unified hierarchy
                || [ ! -d /sys/fs/cgroup ]; then              |                 if [ -f /sys/fs/cgroup/cgroup.controllers ];
                return                                        |                         log_success_msg "cgroups v2 already m
                                                              >                         return 0
                                                              >                 fi
        fi                                                              fi
                                                              >
                                                              >         # Check if cgroups v2 is supported
                                                              >         if [ ! -e /sys/fs/cgroup ]; then
                                                              >                 log_warning_msg "cgroups filesystem not avail
                                                              >                 return 1
                                                              >         fi
                                                              >
                                                              >         # Mount cgroups v2 unified hierarchy
        if ! mountpoint -q /sys/fs/cgroup; then                         if ! mountpoint -q /sys/fs/cgroup; then
                mount -t tmpfs -o uid=0,gid=0,mode=0755 cgrou |                 log_begin_msg "Mounting cgroups v2 unified hi
                                                              >                 mount -t cgroup2 none /sys/fs/cgroup
                                                              >                 log_end_msg $?
                                                              >         fi
                                                              >
                                                              >         # Enable controllers for Docker
                                                              >         if [ -f /sys/fs/cgroup/cgroup.subtree_control ]; then
                                                              >                 # Enable all available controllers
                                                              >                 AVAILABLE_CONTROLLERS=$(cat /sys/fs/cgroup/cg
                                                              >                 if [ -n "$AVAILABLE_CONTROLLERS" ]; then
                                                              >                         for controller in $AVAILABLE_CONTROLL
                                                              >                                 echo "+$controller" > /sys/fs
                                                              >                         done
                                                              >                 fi
        fi                                                              fi
        (                                                     <
                cd /sys/fs/cgroup                             <
                for sys in $(awk '!/^#/ { if ($4 == 1) print  <
                        mkdir -p $sys                         <
                        if ! mountpoint -q $sys; then         <
                                if ! mount -n -t cgroup -o $s <
                                        rmdir $sys || true    <
                                fi                            <
                        fi                                    <
                done                                          <
        )                                                     <
}                                                               }

case "$1" in                                                    case "$1" in
        start)                                                          start)
                fail_unless_root                                                fail_unless_root

                                                              >                 # Mount cgroups v2
                cgroupfs_mount                                                  cgroupfs_mount

                touch "$DOCKER_LOGFILE"                                         touch "$DOCKER_LOGFILE"
                chgrp docker "$DOCKER_LOGFILE"                |                 chgrp docker "$DOCKER_LOGFILE" 2>/dev/null ||

                ulimit -n 1048576                                               ulimit -n 1048576

                # Having non-zero limits causes performance p                   # Having non-zero limits causes performance p
                # in the kernel. We recommend using cgroups t                   # in the kernel. We recommend using cgroups t
                if [ "$BASH" ]; then                                            if [ "$BASH" ]; then
                        ulimit -u unlimited                                             ulimit -u unlimited
                else                                                            else
                        ulimit -p unlimited                                             ulimit -p unlimited
                fi                                                              fi

                log_begin_msg "Starting $DOCKER_DESC: $BASE"                    log_begin_msg "Starting $DOCKER_DESC: $BASE"
                $0 status >>/dev/null \                                         $0 status >>/dev/null \
                || start-stop-daemon --start --background \                     || start-stop-daemon --start --background \
                        --no-close \                                                    --no-close \
                        --exec "$DOCKERD" \                                             --exec "$DOCKERD" \
                        --pidfile "$DOCKER_SSD_PIDFILE" \                               --pidfile "$DOCKER_SSD_PIDFILE" \
                        --make-pidfile \                                                --make-pidfile \
                        -- \                                                            -- \
                        -p "$DOCKER_PIDFILE" \                                          -p "$DOCKER_PIDFILE" \
                        $DOCKER_OPTS \                                                  $DOCKER_OPTS \
                        >> "$DOCKER_LOGFILE" 2>&1                                       >> "$DOCKER_LOGFILE" 2>&1
                log_end_msg $?                                                  log_end_msg $?
                ;;                                                              ;;

        stop)                                                           stop)
                fail_unless_root                                                fail_unless_root
                if [ -f "$DOCKER_SSD_PIDFILE" ]; then                           if [ -f "$DOCKER_SSD_PIDFILE" ]; then
                        log_begin_msg "Stopping $DOCKER_DESC:                           log_begin_msg "Stopping $DOCKER_DESC:
                        start-stop-daemon --stop --pidfile "$                           start-stop-daemon --stop --pidfile "$
                        log_end_msg $?                                                  log_end_msg $?
                else                                                            else
                        log_warning_msg "Docker already stopp                           log_warning_msg "Docker already stopp
                fi                                                              fi
                ;;                                                              ;;

        restart)                                                        restart)
                fail_unless_root                                                fail_unless_root
                docker_pid=$(cat "$DOCKER_SSD_PIDFILE" 2> /de                   docker_pid=$(cat "$DOCKER_SSD_PIDFILE" 2> /de
                [ -n "$docker_pid" ] \                                          [ -n "$docker_pid" ] \
                        && ps -p $docker_pid > /dev/null 2>&1                           && ps -p $docker_pid > /dev/null 2>&1
                        && $0 stop                                                      && $0 stop
                $0 start                                                        $0 start
                ;;                                                              ;;

        force-reload)                                                   force-reload)
                fail_unless_root                                                fail_unless_root
                $0 restart                                                      $0 restart
                ;;                                                              ;;

        status)                                                         status)
                status_of_proc -p "$DOCKER_SSD_PIDFILE" "$DOC                   status_of_proc -p "$DOCKER_SSD_PIDFILE" "$DOC
                ;;                                                              ;;

        *)                                                              *)
                echo "Usage: service docker {start|stop|resta                   echo "Usage: service docker {start|stop|resta
                exit 1                                                          exit 1
                ;;                                                              ;;
esac                                                            esac
```
