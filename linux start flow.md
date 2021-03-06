转自：http://blog.csdn.net/fanbird2008/article/details/8349917
  1、init程序剖析
    init进程是内核引导过程完成时创建的第一个进程。Linux使用了init进程来对组成Linux的服务和应用程序进行初始化。
    当 init 进程启动时（使用传统的sysvinit版本），它会打开一个名为 /etc/inittab 的文件。这个文件是 init 的配置文件，定义了如何对系统进行初始化。这个文件还包含了有关出现电源故障时执行的操作（如果系统支持）、以及在检测到 Ctrl-Alt-Delete 键序列时应该如何反应的信息。请参看 清单 1 中该文件的简短片段，了解它所提供的内容。
    inittab 配置文件使用通用格式定义了几项内容：id:runlevels:action:process。其中 id 是惟一标识该项的字符序列。runlevels 定义了操作所使用的运行级别。action 指定了要执行的特定操作。最后，process 定义了要执行的进程。
清单 1. inittab 文件摘录

[python] view plaincopy
# The default runlevel   
id:2:initdefault  
  
# Boot-time system configuration/initialization script   
si::sysinit:/etc/init.d/rcS  
  
# Runlevels   
l0:0:wait:/etc/init.d/rc 0  
l1:1:wait:/etc/init.d/rc 1  
l2:2:wait:/etc/init.d/rc 2  
l3:3:wait:/etc/init.d/rc 3  
l4:4:wait:/etc/init.d/rc 4  
l5:5:wait:/etc/init.d/rc 5  
l6:6:wait:/etc/init.d/rc 6  
z6:6:respawn:/sbin/sulogin  
  
# How to react to ctrl-alt-del   
ca:12345:ctrlaltdel:/sbin/shutdown -t1 -a -r now  
[python] view plaincopy
# The default runlevel  
id:2:initdefault  
  
# Boot-time system configuration/initialization script  
si::sysinit:/etc/init.d/rcS  
  
# Runlevels  
l0:0:wait:/etc/init.d/rc 0  
l1:1:wait:/etc/init.d/rc 1  
l2:2:wait:/etc/init.d/rc 2  
l3:3:wait:/etc/init.d/rc 3  
l4:4:wait:/etc/init.d/rc 4  
l5:5:wait:/etc/init.d/rc 5  
l6:6:wait:/etc/init.d/rc 6  
z6:6:respawn:/sbin/sulogin  
  
# How to react to ctrl-alt-del  
ca:12345:ctrlaltdel:/sbin/shutdown -t1 -a -r now  
    在 init 加载 /etc/inittab 之后，就会将系统切换到 initdefault 操作所定义的运行级别。如 清单 1 所示，即运行级别 2。我们可以将运行级别看作是系统的状态。例如，运行级别 0 定义了系统挂起状态，运行级别 1 是单用户模式。运行级别 2 到 5 是多用户状态，运行级别 6 表示重启。（注意有些发行版对于运行级别的表示是不同的）。还可以以另一种方式考虑运行级别，即它是一种定义可以执行哪些进程（定义系统状态的进程）的方法。
    我们可以使用 telinit 工具（这是一个指向 init 工具的链接）与 init 进程进行通信。例如，如果目前在多用户模式下（runlevel 2），希望切换到单用户模式（runlevel 1），使用命令 telinit 1 即可（使用超级用户模式）。要查看系统的当前运行级别，请使用命令 runlevel。
    正如 清单 1 定义的一样， initdefault 指定默认的 init 级别是 2 （多用户模式）。在定义初始的运行级别之后，则调用 rc 脚本以及参数 2（运行级别）来启动系统。这个脚本然后会调用各种服务和应用程序脚本来启动或停止特定的元素。在本例中，文件都是在 /etc/rc2.d/ 中定义的。例如，如果要启动 MySQL 应用程序（例如系统启动），可以这样调用：/etc/rc2.d/S20mysql start。在关闭系统时，则使用 stop 参数调用相同的脚本集。
    最后，串行执行大量的脚本以启动各种需要的服务（通常可以在 Linux 的引导屏幕中看到）。即使这些服务彼此无关时，依然会顺次启动它们。其结果是引导过程非常耗时（尤其在具有很多服务的大型系统上更是如此）。
    关于这个问题的一个很明显的解决方案是去掉 init 命令的串行特性，将其替换成并行化操作。在很多多处理系统中都可以看到这种用法。例如，socket striping，或者使用两个或多个 socket 并行地移动数据，就是一个基于这个主题的解决方案。独立磁盘冗余阵列（RAID）系统也是通过将磁盘分成条状（通常是并行的）来提高 I/O 性能。
    修改初始化进程非常的简单。在引导时（使用 LILO 或 GRUB），指定一个新进程来开始处理系统初始化。指定 init=/sbin/mynewinit 作为内核引导行的一部分从而调用这个进程，而不是默认的 init 进程。在 /init/main.c:kernel_init()-->init_post()的内核源代码中您可以看到这种用法。如果在内核引导行中提供了一个 init 命令，引导时就会使用它。否则，内核就会尝试启动 4 个备选方法之一（第一个是 /sbin/init，接着是/etc/init, /bin/init, /bin/sh）。    
    init进程是由内核启动的第一个也是惟一的一个用户进程，它是后续所有进程的发起者，比如init进程启动/bin/sh程序后，才能够在控制台上输入各种命令。
    init执行的基本流程如下：
    （1）解析/etc/inittab：执行sysinit命令指定的进程，以前通常是/etc/init.d/rcS，在新版本的init程序中则通常是/etc/rc.d/rc.sysinit脚本。
    （2）执行/etc/rc.d/rc.sysinit：这是由init执行的第一个脚本，此步进行的工作包括配置网络、配置内核参数、挂载root文件系统、检查文件系统、设置系统时钟、配置机器、开启交换空间等。
    （3）执行/etc/rc.d/rcX.d/[K...][S...]：根据定义的initdefault运行级别，执行对应wait命令指定的程序，这会运行对应目录下的各个程序，并等待它们运行完。在rcX.d目录下，首先终止K开头的服务(用来关闭一个服务)，然后启动S开头的服务(用来启动一个服务)。对每一个运行级别来说，在/etc/rc.d子目录中都有一个对应的下级目录。这些运行级别的下级子目录的命名方法为rcX.d, 其中X就是代表运行级别的数字。在各个运行级别的子目录中，都建立有到/etc/rc.d/init.d子目录中命令脚本程序的符号链接，链接的名称在K与S后有一个数字，表示执行顺序，数字小的先执行，例如K01tog-pegasus、S00microcode_ctl。对以K开头的脚本执行时系统会传递stop参数，而S开头的脚本系统会传递start参数。   
    （4）执行/etc/rc.d/rc.local：Redhat中运行模式2,3,5都把/etc/rc.d/rc.local作为初始化脚本中的最后一个文件，所以用户可以自己在这个文件中添加一些需要在其他初始化工作之后，登陆之前执行的命令。
    （5）执行getty程序：为每个联机终端使用fork()创建一个子进程，并在子进程中运行getty程序，init进程则调用wait()，进入等待子进程结束状态。getty程序设置终端类型、属性、速度和线路规程等。对于字符界面的运行级别（如级别2和3），它会打开并初始化一个tty端口，显示提示信息。通常，若/etc/issue文本文件存在，则getty会首先显示其中的文本信息，然后显示登录提示信息（例如“plinux login:” ），出现字符登录界面，并等待用户键入用户名和口令。可以在inittab文件中配置使用哪一种getty程序（在“id:runlevels:action:process”的process部分指定，并可以传递相应的getty参数），如agetty, getty, mgetty, uugetty, mingetty,fbgetty等。getty程序只能由超级用户执行。
    注意如果第1步中的inittab文件指定的默认运行级别是图形用户界面形式（如级别5），则init程序会转向去执行/etc/X11/prefdm脚本，它会执行/usr/sbin/gdm，启动图形登录界面。GDM管理的不只是X的启动，还有登录，注销，挂起等一系列操作。
    启动登录界面（图形或字符界面），并输入完用户名后，getty会调用login程序。
    （6）执行login程序：getty调用exec()执行login程序，以核对输入的用户名和口令。由于调用了exec（而不是fork），login的执行环境会覆盖getty的执行环境。login进程会读取/etc/passwd，以用户名和口令。login根据用户输入的用户名，从口令文件passwd中取得对应用户的登录项，然后调用getpass()以显示”password:”提示信息，读取用户键入的密码，然后使用加密算法对键入的密码进行加密处理，并与口令文件中该用户项中pw_passwd字段作比较。如果用户几次键入的密码均无效，则login程序会以出错码1退出执行，表示此次登录过程失败。此时父进程（进程init）的wait()会返回该退出进程的pid，因此会根据记录下来的信息再次创建一个子进程，并在该子进程中针对该终端设备再次执行getty程序，重复上述过程。
    如果用户键入的密码正确，则login就会把当前工作目录（Currend Work Directory）修改成口令文件中指定的起始工作目录。并把对该终端设备的访问权限修改成用户读/写和组写，设置进程的组ID。然后利用所得到的信息初始化环境变量信息，例如起始目录（HOME=）、使用的shell程序（SHELL=）、用户名（USER=和LOGNAME=）和系统执行程序的默认路径序列（PATH=）。接着显示/etc/motd文件（message-of-the-day）中的文本信息，并检查并显示该用户是否有邮件的信息。最后login程序改变成登录用户的用户ID，并执行口令文件中该用户项中指定的shell程序，如/bin/bash或/bin/csh等。有关login程序的一些执行选项和特殊访问限制的说明，可参见Linux系统中的在线手册页（man -8 login）。
    （7）执行shell程序或x-windows：如果用户名和口令正确，login调用exec执行shell命令行解释程序（当然，也可以执行X-windows的图形界面，如果用户设置了的话）。登录shell会首先从/etc/profile文件以及$HOME/.bash\_profile文件（或.bashrc文件，若存在的话）读取命令并执行。因此用户可以把每次登录时都要执行的命令放在.bash\_profile文件中。如果在进入shell时设置了ENV环境变量（或者在.bash\_profile文件中设置了该变量），则shell还会从$ENV指定的文件中读去命令并执行。因此我们也可以把每次运行shell都要执行的命令放在ENV变量指定的文件中。设置ENV环境变量的方法是把下列语句放在你起始目录的.bash\_profile文件中： ENV=$HOME/.anyfilename; export ENV。
    运行shell时，原来的getty进程最终被替换成了bash进程，对应的getty,login,bash这三个程序也就具有相同的进程ID。在成功登录到Linux系统后，你会发现（使用”top”或”ps –ax”命令）自己终端原来的getty进程已经找不到了。因为getty进程执行了login程序，被替换成了login进程，并且最后被替换成你的登录shell进程。对于图形用户界面，login程序最后会被替换成图形界面进程（如gnome-session程序）。
    （8）Linux运行时：init进程会负责收取孤儿进程。如果某个进程创建子进程之后，在子进程终止之前终止，则子进程成为孤儿进程。在Linux中所有的进程必须属于单棵进程树，所以孤立进程必须被收取。一旦进程成为孤儿，它会立即成为init进程的子进程。这是为了保持进程树的完整性。
    （9）用户注销：当某个终端或虚拟控制台上的用户注销之后，该终端上的所有进程都会被终止（killed），包括bash。然后，init进程就会调用fork为该终端或虚拟控制台重新创建一个getty进程，以便能够让其他用户登录。这是为什么呢？你应该发现，当用户登录时，“getty”用的是“exec”而不是“fork”系统调用来执行“login”，这样，“login”在执行的时候会覆盖“getty”的执行环境（同理，用户注册成功后，“login”的执行环境也会被shell占用）。所以，如果想再次使用同一终端，必须再启动一个“getty”。对于图形界面，用户注销后会回到图形登录界面。
    （10）系统关闭：init负责杀死所有其它的进程，卸载所有的文件系统并停止处理器的工作，以及任何其它被配置成要做的工作。 
    以Fedora 14桌面系统中为例（它使用新的upstart init程序，不过它兼容sysvinit），/etc/inittab文件内容如下：
[python] view plaincopy
# inittab is only used by upstart for the default runlevel.   
#   
# ADDING OTHER CONFIGURATION HERE WILL HAVE NO EFFECT ON YOUR SYSTEM.   
#   
# System initialization is started by /etc/init/rcS.conf   
#   
# Individual runlevels are started by /etc/init/rc.conf   
#   
# Ctrl-Alt-Delete is handled by /etc/init/control-alt-delete.conf   
#   
# Terminal gettys are handled by /etc/init/tty.conf and /etc/init/serial.conf,   
# with configuration in /etc/sysconfig/init.   
#   
# For information on how to write upstart event handlers, or how   
# upstart works, see init(5), init(8), and initctl(8).   
#   
# Default runlevel. The runlevels used are:   
#   0 - halt (Do NOT set initdefault to this)   
#   1 - Single user mode   
#   2 - Multiuser, without NFS (The same as 3, if you do not have networking)   
#   3 - Full multiuser mode   
#   4 - unused   
#   5 - X11   
#   6 - reboot (Do NOT set initdefault to this)   
#    
id:5:initdefault:  
[python] view plaincopy
# inittab is only used by upstart for the default runlevel.  
#  
# ADDING OTHER CONFIGURATION HERE WILL HAVE NO EFFECT ON YOUR SYSTEM.  
#  
# System initialization is started by /etc/init/rcS.conf  
#  
# Individual runlevels are started by /etc/init/rc.conf  
#  
# Ctrl-Alt-Delete is handled by /etc/init/control-alt-delete.conf  
#  
# Terminal gettys are handled by /etc/init/tty.conf and /etc/init/serial.conf,  
# with configuration in /etc/sysconfig/init.  
#  
# For information on how to write upstart event handlers, or how  
# upstart works, see init(5), init(8), and initctl(8).  
#  
# Default runlevel. The runlevels used are:  
#   0 - halt (Do NOT set initdefault to this)  
#   1 - Single user mode  
#   2 - Multiuser, without NFS (The same as 3, if you do not have networking)  
#   3 - Full multiuser mode  
#   4 - unused  
#   5 - X11  
#   6 - reboot (Do NOT set initdefault to this)  
#   
id:5:initdefault:  
    Fedora的默认运行级别为5，是多用户的x-windows图形界面。与传统的sysvinit有所不同，在upstart中，只会为默认运行级别使用inittab文件，要添加其他的运行级别，应该放到/etc/init/rc.conf中，而不是inittab中。upstart系统现在首先运行的是/etc/init/rcS.conf，其内容如下(Fedora 14中)：
[python] view plaincopy
start on startup  
  
stop on runlevel  
  
task  
  
# Note: there can be no previous runlevel here, if we have one it's bad   
# information (we enter rc1 not rcS for maintenance).  Run /etc/rc.d/rc   
# without information so that it defaults to previous=N runlevel=S.   
console output  
exec /etc/rc.d/rc.sysinit  
post-stop script  
    if [ "$UPSTART_EVENTS" = "startup" ]; then  
        [ -f /etc/inittab ] && runlevel=$(/bin/awk -F ':' '$3 == "initdefault" && $1 !~ "^#" { print $2 }' /etc/inittab)  
        [ -z "$runlevel" ] && runlevel="3"  
        for t in $(cat /proc/cmdline); do  
            case $t in  
                -s|single|S|s) runlevel="S" ;;  
                [1-9])       runlevel="$t" ;;  
            esac  
        done  
        exec telinit $runlevel  
    fi  
end script  
[python] view plaincopy
start on startup  
  
stop on runlevel  
  
task  
  
# Note: there can be no previous runlevel here, if we have one it's bad  
# information (we enter rc1 not rcS for maintenance).  Run /etc/rc.d/rc  
# without information so that it defaults to previous=N runlevel=S.  
console output  
exec /etc/rc.d/rc.sysinit  
post-stop script  
    if [ "$UPSTART_EVENTS" = "startup" ]; then  
        [ -f /etc/inittab ] && runlevel=$(/bin/awk -F ':' '$3 == "initdefault" && $1 !~ "^#" { print $2 }' /etc/inittab)  
        [ -z "$runlevel" ] && runlevel="3"  
        for t in $(cat /proc/cmdline); do  
            case $t in  
                -s|single|S|s) runlevel="S" ;;  
                [1-9])       runlevel="$t" ;;  
            esac  
        done  
        exec telinit $runlevel  
    fi  
end script  
    upstart首先在系统引导时首先运行rc.sysinit脚本，然后搜索inittab的initdefault，用telinit切换到initdefault的级别来运行。upstart把原来/etc/inittab的功能被分散到/etc/init下的各个conf文件中。
    注意不同的Linux发行版会对upstart做一些不同的定制。在ubuntu中，甚至默认不再有/etc/inittab文件了。当然他仍然会处理这个文件（如果有的话），如果你有需要，可以创建这个文件，添加需要的内容。Ubuntu 10.04中的/etc/init/rcS.conf内容如下：
[python] view plaincopy
# rcS - System V single-user mode compatibility   
#   
# This task handles the old System V-style single-user mode, this is   
# distinct from the other runlevels since running the rc script would   
# be bad.   
  
description     "System V single-user mode compatibility"  
author          "Scott James Remnant <scott@netsplit.com>"  
  
start on runlevel S  
stop on runlevel [!S]  
  
console owner  
script  
    if [ -x /usr/share/recovery-mode/recovery-menu ]; then  
        exec /usr/share/recovery-mode/recovery-menu  
    else  
        exec /sbin/sulogin  
    fi  
end script  
  
post-stop script  
    # Don't switch runlevels if we were stopped by an event, since that   
    # means we're already switching runlevels   
    if [ -n "${UPSTART_STOP_EVENTS}" ]  
    then  
        exit 0  
    fi  
  
    # Switch, passing a magic flag   
    start --no-wait rc-sysinit FROM_SINGLE_USER_MODE=y  
end script  
[python] view plaincopy
# rcS - System V single-user mode compatibility  
#  
# This task handles the old System V-style single-user mode, this is  
# distinct from the other runlevels since running the rc script would  
# be bad.  
  
description     "System V single-user mode compatibility"  
author          "Scott James Remnant <scott@netsplit.com>"  
  
start on runlevel S  
stop on runlevel [!S]  
  
console owner  
script  
    if [ -x /usr/share/recovery-mode/recovery-menu ]; then  
        exec /usr/share/recovery-mode/recovery-menu  
    else  
        exec /sbin/sulogin  
    fi  
end script  
  
post-stop script  
    # Don't switch runlevels if we were stopped by an event, since that  
    # means we're already switching runlevels  
    if [ -n "${UPSTART_STOP_EVENTS}" ]  
    then  
        exit 0  
    fi  
  
    # Switch, passing a magic flag  
    start --no-wait rc-sysinit FROM_SINGLE_USER_MODE=y  
end script  
    这里先做一些前期检查，与Fedora不同的是，第一个执行的脚本换成了/etc/init/rc-sysinit.conf，其内容如下：
[python] view plaincopy
# rc-sysinit - System V initialisation compatibility   
#   
# This task runs the old System V-style system initialisation scripts,   
# and enters the default runlevel when finished.   
  
description "System V initialisation compatibility"  
author      "Scott James Remnant <scott@netsplit.com>"  
  
start on filesystem and net-device-up IFACE=lo  
stop on runlevel  
  
# Default runlevel, this may be overriden on the kernel command-line   
# or by faking an old /etc/inittab entry   
env DEFAULT_RUNLEVEL=2  
  
# There can be no previous runlevel here, but there might be old   
# information in /var/run/utmp that we pick up, and we don't want   
# that.   
#   
# These override that   
env RUNLEVEL=  
env PREVLEVEL=  
  
console output  
env INIT_VERBOSE  
  
task  
  
script  
    # Check for default runlevel in /etc/inittab   
    if [ -r /etc/inittab ]  
    then  
    eval "$(sed -nre 's/^[^#][^:]*:([0-6sS]):initdefault:.*/DEFAULT_RUNLEVEL="\1";/p' /etc/inittab || true)"  
    fi  
  
    # Check kernel command-line for typical arguments   
    for ARG in $(cat /proc/cmdline)  
    do  
    case "${ARG}" in  
    -b|emergency)  
        # Emergency shell   
        [ -n "${FROM_SINGLE_USER_MODE}" ] || sulogin  
        ;;  
    [0123456sS])  
        # Override runlevel   
        DEFAULT_RUNLEVEL="${ARG}"  
        ;;  
    -s|single)  
        # Single user mode   
        [ -n "${FROM_SINGLE_USER_MODE}" ] || DEFAULT_RUNLEVEL=S  
        ;;  
    esac  
    done  
  
    # Run the system initialisation scripts   
    [ -n "${FROM_SINGLE_USER_MODE}" ] || /etc/init.d/rcS  
  
    # Switch into the default runlevel   
    telinit "${DEFAULT_RUNLEVEL}"  
end script  
[python] view plaincopy
# rc-sysinit - System V initialisation compatibility  
#  
# This task runs the old System V-style system initialisation scripts,  
# and enters the default runlevel when finished.  
  
description "System V initialisation compatibility"  
author      "Scott James Remnant <scott@netsplit.com>"  
  
start on filesystem and net-device-up IFACE=lo  
stop on runlevel  
  
# Default runlevel, this may be overriden on the kernel command-line  
# or by faking an old /etc/inittab entry  
env DEFAULT_RUNLEVEL=2  
  
# There can be no previous runlevel here, but there might be old  
# information in /var/run/utmp that we pick up, and we don't want  
# that.  
#  
# These override that  
env RUNLEVEL=  
env PREVLEVEL=  
  
console output  
env INIT_VERBOSE  
  
task  
  
script  
    # Check for default runlevel in /etc/inittab  
    if [ -r /etc/inittab ]  
    then  
    eval "$(sed -nre 's/^[^#][^:]*:([0-6sS]):initdefault:.*/DEFAULT_RUNLEVEL="\1";/p' /etc/inittab || true)"  
    fi  
  
    # Check kernel command-line for typical arguments  
    for ARG in $(cat /proc/cmdline)  
    do  
    case "${ARG}" in  
    -b|emergency)  
        # Emergency shell  
        [ -n "${FROM_SINGLE_USER_MODE}" ] || sulogin  
        ;;  
    [0123456sS])  
        # Override runlevel  
        DEFAULT_RUNLEVEL="${ARG}"  
        ;;  
    -s|single)  
        # Single user mode  
        [ -n "${FROM_SINGLE_USER_MODE}" ] || DEFAULT_RUNLEVEL=S  
        ;;  
    esac  
    done  
  
    # Run the system initialisation scripts  
    [ -n "${FROM_SINGLE_USER_MODE}" ] || /etc/init.d/rcS  
  
    # Switch into the default runlevel  
    telinit "${DEFAULT_RUNLEVEL}"  
end script  
    可见Ubuntu是在rc-sysinit.conf中才处理inittab并切换到initdefault级别来运行。
    init的完整初始化过程如下（包括启动字符界面和图形界面）：

[python] view plaincopy
/sbin/init  
    --->/etc/init/rcS.conf  
        --->exec /etc/rc.d/rc.sysinit        执行第一个脚本（Ubuntu中为/etc/init/rc-sysinit.conf）  
            --->/bin/hostname        获取主机名(设置$HOSTNAME)  
            --->/etc/sysconfig/network       配置网络基本参数  
            --->/proc/mounts     检测并挂载procfs,sysfs到/proc,/sys  
            --->/etc/init.d/functions  包含一些通用函数，会被/etc/init.d（是到rc.d/init.d的链接）下的脚本用到  
                --->/etc/sysconfig/i18n      设置终端字符集  
                --->/etc/sysconfig/init      设置终端和图形界面的一些参数  
                --->deamon(),killproc(),pidofproc()      一些通用函数  
                --->status(),echo_success(),  
                --->update_boot_stage(),strstr()  
            --->/selinux/enforce     检查SELinux的状态  
            --->/etc/system-release      打印熟悉的发行版信息 “Welcome to Fedora ..."  
            --->/proc/cmdline        获取内核启动的命令行参数  
            --->/proc/sys/kernel/modprobe        获取modprobe的位置（为/sbin/modprobe）  
                --->/sbin/sysctl     初始化硬件（通过sysctl设置运行时内核参数）  
            --->kill $nashpid        杀死所有的nash进程（我们在initrd中使用的shell）  
            --->/sbin/start_udev     启动udev（(动态设备管理进程）  
            --->/bin/taskset  设置进程的默认CPU亲合值（即优先使用哪个CPU，用在多处理器环境中）  
            --->/etc/sysconfig/modules/*.modules     加载其他用户自定义的模块  
            --->sysctl -e -p /etc/sysctl.conf        配置内核参数  
            --->/proc/devices        获取设备号及相应设备名，以便进行设备初始化  
            --->/sbin/dmraid     激活software raid  
            --->/sbin/kpartx “/dev/mapper/..."  为software raid上的每块硬盘创建设备映射  
            --->/.autofsck       是否自动执行文件系统检查  
                --->sulogin      若为单用户模式，执行单用户登录程序  
                --->plymouth --show-splash       显示启动时的背景画面  
            --->/etc/sysconfig/readonly-root     设置root文件系统挂载方式  
            --->从/etc/fstab挂载暂存设备  
            --->/etc/rwtab, /etc/rwtab.d/*       挂载其他有卷标的分区  
            --->ip addr show         获取并设置网上ip地址  
            --->从/etc/fstab挂载持久数据的存储设备  
            --->/etc/statetab, /etc/statetab.d/*     持载其他持久数据的存储设备  
            --->/sbin/fsck       检查文件系统  
                --->umount -a & reboot -f        如果检查失败，卸载文件系统并重启  
            --->以读写方式重新挂载root文件系统        如果文件系统检查没有失败  
            --->挂载所有其他的文件系统  
            --->cat /var/lib/random-seed > /dev/urandom       初始化伪随机数生成器  
            --->/usr/sbin/system-config-keyboard,passwd,...      配置机器相关参数（如果有需要的话）  
            --->/etc/sysconfig/network       重新读取网络配置数据，并重设hostname  
            --->清除相关的/, /var,/tmp数据  
            --->/sbin/swapon     开启各个交换区分（根据/proc/swaps）  
            --->/usr/sbin/system-config-network-cmd    执行引导时的网络配置（传递内核启动的netprofile参数）   
            --->dmesg -s 131072 > /var/log/dmesg      转储内核启动的消息信息               
        --->/etc/inittab  
            --->id:5:initdefault:        查找initdefault定义的运行级别（为5，图形用户界面）  
        --->telinit $runlevel        切换到对应级别运行  
    --->/etc/init/rc.conf  
        --->exec /etc/rc.d/rc $RUNLEVEL  
            --->/etc/profile.d/lang.sh       设置语言环境  
            --->/etc/rc.d/rc5.d/KNxxxx       先关闭相关服务（在关闭系统时也会执行）  
                --->/etc/rc.d/init.d/xxxx      
            --->/etc/rc.d/rc5.d/SNxxxx       再开启相关服务  
                --->etc/rc.d/rc5.d/xxxx    
    --->/etc/rc.d/rc.local  在所有init脚本运行完之后运行，可在些添加自己的初始化命令（Ubuntu中为/etc/rc.local）  
    --->/etc/init/start-ttys.conf        启动tty1-tty6设备  
        --->/etc/sysconfig/init      指定tty设备，通常为/dev/tty1-/dev/tty6  
        --->/etc/init/tty.conf  
            --->exec /sbin/mingetty $TTY     在每个tty设备上启动mingetty  
    --->成功后就可以通过Ctrl+Alt+F1..F6在各个不同的tty之间切换  
################################################# 字符界面 ################################################   
    --->fork()--->/sbin/mingetty      运行mingetty程序，出现字符登录界面  
        --->/etc/issue       在登录界面上显示发行版信息  
        --->exec("/bin/login",...)       运行/bin/login程序，验证用户名和口令  
            --->/etc/passwd      读取passwd文件核对用户名和口令  
                --->jackzhou:x:500:500:jackzhou:/home/jackzhou:/bin/bash  
            --->切换到工作目录/home/jackzhou  
            --->初始化环境变量$HOME,$PATH等  
            --->/etc/motd        显示当天的消息  
            --->检查新邮件  
            --->exec("/bin/bash",...)        运行bash程序  
                --->/etc/profile     执行这些脚本中的命令  
                --->.bash_profile或.bashrc          
                    --->ENV=$HOME/.anyfilename; export ENV  运行$ENV指向的脚本（如果设置了的话）  
    --->bash运行中          mingetty,login最后替换成了bash,登录成功  
################################################## 图形界面 #############################################   
    --->/etc/init/prefdm.conf                                   
        --->exec /etc/X11/prefdm -nodaemon       准备启动指定的X图形界面(X Display Manager)  
            --->/etc/sysconfig/i18n      设置语言环境  
            --->/etc/sysconfig/desktop       读取指定的DM配置（如果有的话）  
            --->exec /usr/sbin/gdm       启动指定的DM（gdm, kdm, wdm或xdm，默认为/usr/sbin/gdm）  
                --->启动X server窗口  
                --->/etc/gdm/custom.conf     根据配置在X窗口中显示登录界面  
                --->用户选择语言、键盘布局、会话等  
                    --->/usr/share/xsessions/gnome.desktop       读取会话要显示的名称  
                        --->Exec=gnome-session       指定默认的会话程序  
                --->用户输入用户名和密码  
                --->用/bin/login验证用户名和密码                          
                --->/etc/gdm/PreSession/*        执行会话前的一些任务（比如更改X窗口的默认背景）  
                --->/etc/gdm/PostLogin/*     执行一些登录后立即需要运行的命令  
                --->/etc/gdm/Xsession gnome-session--->/etc/X11/xinit/Xsession        启动GNOME会话  
                    --->/etc/X11/xinit/xinitrc-common        导入Xsession与xinitrc共用的代码  
                        --->/etc/profile.d/lang.sh       设置i18n环境  
                        --->/etc/X11/Xresources      读取用户登录时需要载入的全局资源  
                        --->/etc/X11/Xmodmap  读取的全局的键盘配置（用于xdm和xinit，用startx启动图形界面时要用到）  
                        --->/etc/X11/xinit/xinitrc.d/*       运行所有的xinitrc脚本  
                    --->exec -l $SHELL -c gnome-session  执行特定的环境设置（以前是执行./Xclients.d/Xclients.gnome-session.sh）  
                    --->/etc/X11/xinit/Xclients  运行各个X客户端的脚本（或者$HOME/.xsession,或者$HOME/.Xclients）  
                        --->/etc/sysconfig/desktop       读取指定的会话程序配置（如果有的话）  
                        --->exec "$(type -p gnome-session)"  默认运行gnome-session，进入GNOME桌面  
    --->GNOME桌面运行中               mingetty,login最后替换成了gnome程序,登录成功  
    --->/etc/gdm/PostSession/*       GNOME会话结束时运行的脚本  
#################################### 在字符界面下通过startx启动图形界面 ##############################################   
    --->/bin/bash                在字符界面的Shell下  
        --->/usr/bin/startx  
            --->记录$HOME目录和/etc/X11/xinit下的.xinitrc和.xserverrc文件  以$HOME目录下的为优先  
            --->解析用户指定的client、server、display参数及其选项  
            --->没有指定参数时就设为前面记录的.xinitrc和.xserverrc文件  
            --->XAUTHORITY=$HOME/.Xauthority     设置XAUTHORITY环境变量  
            --->设置X server的权限信息  
            --->xinit $client $clientargs -- $server $display $serverargs    启动X server和第一个X client  
                --->/etc/X11/xinit/xinitrc   用来运行各个X client（上面没有指定第一个client时）  
                    --->/etc/X11/xinit/xinitrc-common        导入Xsession与xinitrc共用的代码  
                        --->/etc/profile.d/lang.sh       设置i18n环境  
                        --->/etc/X11/Xresources    读取用户登录时需要载入的全局资源  
                        --->/etc/X11/Xmodmap 读取全局的键盘配置  
                        --->/etc/X11/xinit/xinitrc.d/*   运行所有的xinitrc脚本  
                    --->/etc/X11/xinit/Xclients  运行各个X client的脚本（或者$HOME/.Xclients）  
                        --->/etc/sysconfig/desktop   读取指定的会话程序配置（如果有的话）  
                        --->exec "$(type -p gnome-session)"  默认运行gnome-session，进入GNOME桌面  
      
    --->GNOME桌面运行中       mingetty,login最后替换成了gnome程序,登录成功  
    --->/etc/gdm/PostSession/*       GNOME会话结束时运行的脚本  
[python] view plaincopy
/sbin/init  
    --->/etc/init/rcS.conf  
        --->exec /etc/rc.d/rc.sysinit        执行第一个脚本（Ubuntu中为/etc/init/rc-sysinit.conf）  
            --->/bin/hostname        获取主机名(设置$HOSTNAME)  
            --->/etc/sysconfig/network       配置网络基本参数  
            --->/proc/mounts     检测并挂载procfs,sysfs到/proc,/sys  
            --->/etc/init.d/functions  包含一些通用函数，会被/etc/init.d（是到rc.d/init.d的链接）下的脚本用到  
                --->/etc/sysconfig/i18n      设置终端字符集  
                --->/etc/sysconfig/init      设置终端和图形界面的一些参数  
                --->deamon(),killproc(),pidofproc()      一些通用函数  
                --->status(),echo_success(),  
                --->update_boot_stage(),strstr()  
            --->/selinux/enforce     检查SELinux的状态  
            --->/etc/system-release      打印熟悉的发行版信息 “Welcome to Fedora ..."  
            --->/proc/cmdline        获取内核启动的命令行参数  
            --->/proc/sys/kernel/modprobe        获取modprobe的位置（为/sbin/modprobe）  
                --->/sbin/sysctl     初始化硬件（通过sysctl设置运行时内核参数）  
            --->kill $nashpid        杀死所有的nash进程（我们在initrd中使用的shell）  
            --->/sbin/start_udev     启动udev（(动态设备管理进程）  
            --->/bin/taskset  设置进程的默认CPU亲合值（即优先使用哪个CPU，用在多处理器环境中）  
            --->/etc/sysconfig/modules/*.modules     加载其他用户自定义的模块  
            --->sysctl -e -p /etc/sysctl.conf        配置内核参数  
            --->/proc/devices        获取设备号及相应设备名，以便进行设备初始化  
            --->/sbin/dmraid     激活software raid  
            --->/sbin/kpartx “/dev/mapper/..."  为software raid上的每块硬盘创建设备映射  
            --->/.autofsck       是否自动执行文件系统检查  
                --->sulogin      若为单用户模式，执行单用户登录程序  
                --->plymouth --show-splash       显示启动时的背景画面  
            --->/etc/sysconfig/readonly-root     设置root文件系统挂载方式  
            --->从/etc/fstab挂载暂存设备  
            --->/etc/rwtab, /etc/rwtab.d/*       挂载其他有卷标的分区  
            --->ip addr show         获取并设置网上ip地址  
            --->从/etc/fstab挂载持久数据的存储设备  
            --->/etc/statetab, /etc/statetab.d/*     持载其他持久数据的存储设备  
            --->/sbin/fsck       检查文件系统  
                --->umount -a & reboot -f        如果检查失败，卸载文件系统并重启  
            --->以读写方式重新挂载root文件系统        如果文件系统检查没有失败  
            --->挂载所有其他的文件系统  
            --->cat /var/lib/random-seed > /dev/urandom       初始化伪随机数生成器  
            --->/usr/sbin/system-config-keyboard,passwd,...      配置机器相关参数（如果有需要的话）  
            --->/etc/sysconfig/network       重新读取网络配置数据，并重设hostname  
            --->清除相关的/, /var,/tmp数据  
            --->/sbin/swapon     开启各个交换区分（根据/proc/swaps）  
            --->/usr/sbin/system-config-network-cmd    执行引导时的网络配置（传递内核启动的netprofile参数）   
            --->dmesg -s 131072 > /var/log/dmesg      转储内核启动的消息信息               
        --->/etc/inittab  
            --->id:5:initdefault:        查找initdefault定义的运行级别（为5，图形用户界面）  
        --->telinit $runlevel        切换到对应级别运行  
    --->/etc/init/rc.conf  
        --->exec /etc/rc.d/rc $RUNLEVEL  
            --->/etc/profile.d/lang.sh       设置语言环境  
            --->/etc/rc.d/rc5.d/KNxxxx       先关闭相关服务（在关闭系统时也会执行）  
                --->/etc/rc.d/init.d/xxxx      
            --->/etc/rc.d/rc5.d/SNxxxx       再开启相关服务  
                --->etc/rc.d/rc5.d/xxxx    
    --->/etc/rc.d/rc.local  在所有init脚本运行完之后运行，可在些添加自己的初始化命令（Ubuntu中为/etc/rc.local）  
    --->/etc/init/start-ttys.conf        启动tty1-tty6设备  
        --->/etc/sysconfig/init      指定tty设备，通常为/dev/tty1-/dev/tty6  
        --->/etc/init/tty.conf  
            --->exec /sbin/mingetty $TTY     在每个tty设备上启动mingetty  
    --->成功后就可以通过Ctrl+Alt+F1..F6在各个不同的tty之间切换  
################################################# 字符界面 ################################################  
    --->fork()--->/sbin/mingetty      运行mingetty程序，出现字符登录界面  
        --->/etc/issue       在登录界面上显示发行版信息  
        --->exec("/bin/login",...)       运行/bin/login程序，验证用户名和口令  
            --->/etc/passwd      读取passwd文件核对用户名和口令  
                --->jackzhou:x:500:500:jackzhou:/home/jackzhou:/bin/bash  
            --->切换到工作目录/home/jackzhou  
            --->初始化环境变量$HOME,$PATH等  
            --->/etc/motd        显示当天的消息  
            --->检查新邮件  
            --->exec("/bin/bash",...)        运行bash程序  
                --->/etc/profile     执行这些脚本中的命令  
                --->.bash_profile或.bashrc          
                    --->ENV=$HOME/.anyfilename; export ENV  运行$ENV指向的脚本（如果设置了的话）  
    --->bash运行中          mingetty,login最后替换成了bash,登录成功  
################################################## 图形界面 #############################################  
    --->/etc/init/prefdm.conf                                   
        --->exec /etc/X11/prefdm -nodaemon       准备启动指定的X图形界面(X Display Manager)  
            --->/etc/sysconfig/i18n      设置语言环境  
            --->/etc/sysconfig/desktop       读取指定的DM配置（如果有的话）  
            --->exec /usr/sbin/gdm       启动指定的DM（gdm, kdm, wdm或xdm，默认为/usr/sbin/gdm）  
                --->启动X server窗口  
                --->/etc/gdm/custom.conf     根据配置在X窗口中显示登录界面  
                --->用户选择语言、键盘布局、会话等  
                    --->/usr/share/xsessions/gnome.desktop       读取会话要显示的名称  
                        --->Exec=gnome-session       指定默认的会话程序  
                --->用户输入用户名和密码  
                --->用/bin/login验证用户名和密码                          
                --->/etc/gdm/PreSession/*        执行会话前的一些任务（比如更改X窗口的默认背景）  
                --->/etc/gdm/PostLogin/*     执行一些登录后立即需要运行的命令  
                --->/etc/gdm/Xsession gnome-session--->/etc/X11/xinit/Xsession        启动GNOME会话  
                    --->/etc/X11/xinit/xinitrc-common        导入Xsession与xinitrc共用的代码  
                        --->/etc/profile.d/lang.sh       设置i18n环境  
                        --->/etc/X11/Xresources      读取用户登录时需要载入的全局资源  
                        --->/etc/X11/Xmodmap  读取的全局的键盘配置（用于xdm和xinit，用startx启动图形界面时要用到）  
                        --->/etc/X11/xinit/xinitrc.d/*       运行所有的xinitrc脚本  
                    --->exec -l $SHELL -c gnome-session  执行特定的环境设置（以前是执行./Xclients.d/Xclients.gnome-session.sh）  
                    --->/etc/X11/xinit/Xclients  运行各个X客户端的脚本（或者$HOME/.xsession,或者$HOME/.Xclients）  
                        --->/etc/sysconfig/desktop       读取指定的会话程序配置（如果有的话）  
                        --->exec "$(type -p gnome-session)"  默认运行gnome-session，进入GNOME桌面  
    --->GNOME桌面运行中               mingetty,login最后替换成了gnome程序,登录成功  
    --->/etc/gdm/PostSession/*       GNOME会话结束时运行的脚本  
#################################### 在字符界面下通过startx启动图形界面 ##############################################  
    --->/bin/bash                在字符界面的Shell下  
        --->/usr/bin/startx  
            --->记录$HOME目录和/etc/X11/xinit下的.xinitrc和.xserverrc文件  以$HOME目录下的为优先  
            --->解析用户指定的client、server、display参数及其选项  
            --->没有指定参数时就设为前面记录的.xinitrc和.xserverrc文件  
            --->XAUTHORITY=$HOME/.Xauthority     设置XAUTHORITY环境变量  
            --->设置X server的权限信息  
            --->xinit $client $clientargs -- $server $display $serverargs    启动X server和第一个X client  
                --->/etc/X11/xinit/xinitrc   用来运行各个X client（上面没有指定第一个client时）  
                    --->/etc/X11/xinit/xinitrc-common        导入Xsession与xinitrc共用的代码  
                        --->/etc/profile.d/lang.sh       设置i18n环境  
                        --->/etc/X11/Xresources    读取用户登录时需要载入的全局资源  
                        --->/etc/X11/Xmodmap 读取全局的键盘配置  
                        --->/etc/X11/xinit/xinitrc.d/*   运行所有的xinitrc脚本  
                    --->/etc/X11/xinit/Xclients  运行各个X client的脚本（或者$HOME/.Xclients）  
                        --->/etc/sysconfig/desktop   读取指定的会话程序配置（如果有的话）  
                        --->exec "$(type -p gnome-session)"  默认运行gnome-session，进入GNOME桌面  
      
    --->GNOME桌面运行中       mingetty,login最后替换成了gnome程序,登录成功  
    --->/etc/gdm/PostSession/*       GNOME会话结束时运行的脚本  
    注意在rc.sysinit加载完/etc/sysconfig/modules/中（如果你希望额外加载一些比如遥控器之类的模块，你可以在这里增加脚本）的用户自定义的模块后，就会由update_boot_stage通知图形化的启动界面，准备进入启动画面，内核启动的命令行参数（在grub中可以看到）中会指定rhgb程序。rhgb程序的作用是在启动的时候建立一个临时的仅使用loopback网络的X窗口服务器，然后在这个窗口上显示启动进度，init程序的其他部分可以通过rhgb-client程序向这个进度窗口发送消息。在“配置机器相关参数”这一步中，如果存在/.unconfigured文件，会先调用rhgb-client向进度窗口发送消息，然后调用/usr/bin/system-config-keyboard配置键盘，调用 /usr/bin/passwd root配置超级用户密码，调用/usr/sbin/system-config-network-tui配置网络，调用/usr/sbin/timeconfig配置时区，调用 /usr/sbin/authconfig-tui --nostart配置网络登录，调用/usr/sbin/ntsysv配置默认的运行级别。然后清空包括/var/lock/，/var/run/， /tmp等在内的临时目录，并开启交换空间。运行完rc.sysinit后，rcS.conf就会查找inittab中的默认运行级别并切换到这个级别。

转到rc.conf，它调用/etc/rc.d/rc脚本，运行指定级别目录下的各个启动脚本。首先按照名称顺序运行那些K打头的脚本，然后按照名称顺序运行那些S打头的脚本。启动脚本（是符号链接）中的数字是怎么来的呢，它是由你指定的，如果你要增加自己的启动脚本到相应的启动级别中去，这个数字当然应该由你指定，因为只有你才知道这个脚本应该以什么样的优先级启动。但是对于那些已经存在的启动脚本，它们的优先级是在脚本中最前面的注释行中的chkconfig这一行指定的，在这一行中，你可以看到类似# chkconfig: 35 99 95的字样，它的含义是：这个脚本应该增加到运行级别3和运行级别5中，启动的优先级是99,关闭的优先级是95,当然，这些数字是由那些Fedora的开发者测试过没有问题，所以才写在这里的。二进制的程序/sbin/chkconfig将会读取这一行，并且在将服务增加到启动级别中去的时候自动生成文件名。文件名中的第一个字符S和K代表了什么含义呢？它代表了你在services控制面板中选择了打开这个服务还是关闭这个服务，如果你在那里打开了这个服务，则以S作为前导符，否则为K。结合我们前面介绍的启动过程，你就可以知道，在启动的时候，Fedora会首先保证那些K打头的脚本是关闭的(通过以stop参数调用这个脚本)，然后才会逐个启动那些S打头的脚本(通过以start参数调用这个脚本)。对于每个启动脚本文件，如果想知道启动了时候都做了些什么，可以查看相应脚本中的start()函数，比如对于avahi-dnsconfd这个脚本，我们可以看到，它只是运行了/usr/sbin/avahi-dnsconfd -D这个命令。

    最后执行的初始化脚本是rc.local，你可以在这个脚本中添加自定义的需要启动的服务或需要执行的命令。在所有需要启动的服务都启动完毕以后，rc脚本通过rhgb-client程序通知rhgb图形界面退出，rhgb的使命就完成了。
    init程序在运行完rc.local后，执行/etc/init/start-ttys.conf的配置，它查找init配置文件中的$ACTIVE_CONSOLES指定的每个tty设备（为tty1-tty6)，并调用tty.conf启动这些tty设备。tty.conf会在/dev/tty1-/dev/tty6设备上启动/sbin/mingetty。从现在开始，你可以通过Ctrl+Alt+F1..F6在各个不同的tty之间进行切换了。
    上面“fork()--->/sbin/mingetty”这一部分是指传统字符界面的启动及登录过程，这个过程在前面已经介绍比较清楚了。现在的Linux桌面系统都是登录到图形用户界面，登录屏幕是图形界面的形式。下面“/etc/init/start-ttys.conf”这一部分是指图形界面的启动和登录过程。
    1）直接启动图形界面
    在启动mingetty后，如果运行级别为5，init程序会执行prefdm.conf的配置，它调用/etc/X11/prefdm脚本，准备启动图形界面登录管理器。prefdm将会读取位于/etc/sysconfig/desktop中的配置文件，如果没有指定任何配置文件，prefdm运行的顺序依次为gdm,kdm,wdm和xdm，从而出现图形登录界面。后面的启动部分就属于GNOME，KDE或者其它相应的窗口管理器了。
    注意，对于Ubuntu，默认运行级别为2，在/etc/rc2.d目录中包含了启动登录管理器gdm的脚本。    
    无论是gdm，xdm还是kdm，所做的事情都是类似的，即启动一个X server窗口，基于这个X窗口提供一个图形化的用户登录界面，以便在实际进入X窗口系统之前，对用户进行验证，并且提供用户选择自己希望的语言，窗口管理器等的机会。除此之外，dm程序一般还支持别的一些操作，比如提供直接关机的选项以及根据配置决定是否打开XDMCP服务的端口等。gdm的配置定义在/etc/gdm/custom.conf中。
    XDMCP服务是X窗口显示管理器控制协议的缩写，它允许用户在远程电脑上运行X窗口服务，然后通过XDMCP协议使用本地的XDM登录，登录以后的后续操作将使用远程的X窗口作为显示系统。一个很简单的例子，首先使用gdmsetup程序（系统管理菜单中的登录窗口）打开XDMCP的支持(远程选项卡更改为与本地相同)，然后打开一个终端窗口，运行Xnest :1 -query 127.0.0.1命令(Xnest并不是默认安装的命令)，你将在一个新开的窗口中看到和你的登录屏幕一模一样的登录屏幕，你可以登录其它用户，进行所有和本地用户一样的操作。显然如果你是在另外一台电脑上，只需要把相应的ip地址改掉就可以了。并不一定非要使用Xnest程序，你甚至可以在远程的Win32系统上进行基于XDMCP的远程登录，这首先需要你在你的windows系统上运行一个X 窗口系统，有很多种类似的实现，包括X-win32和cygwin在内的各种免费和收费版本都是一个不错的选择，事实上，一台强劲的服务器通过这种方法可以将N台落魄的486PC转变成可以运行高级科学运算的X终端。除了这些方法，你还可以使用内置于gnome之中的vino程序，这个程序可以基于本地的X窗口打开一个兼容于vnc的服务，你可以使用各种类型的vncviewer来连接这个服务并进行远程操作（参见首选项菜单中的远程桌面），这种实现方式下，远程显示的屏幕和本地屏幕是完全相同的。或者你也可以使用单独的vncserver，这种使用方式和XDMCP的使用方法类似，只是登录的用户和使用的窗口管理器都是由vncserver指定好的。
    在登录界面上，用户可以选择语言、键盘布局、会话等。系统内建的几个会话包括安全模式终端，安全模式gnome以及上一次的成功登录等，其它的会话则是从配置文件中读取的，gdm将会在多个目录中寻找设定的会话，包括/etc/X11/sessions/，/usr/share/gdm/BuiltInSessions/，/usr/share/xsessions/等，路径可以通过daemon/SessionDesktopDir配置项进行更改，gdm在这些目录中寻找扩展名为desktop的文件，比如默认会话对应的文件是 /usr/share/gdm/BuiltInSessions/default.desktop，而gnome会话对应的文件为/usr/share/xsessions/gnome.desktop。这些配置文件定义了在不同的语言中这个会话要显示的名称。
    当用户选择了一个X会话，输入了正确的用户名和密码以后，gdm执行命令的顺序依次是，首先它将执行位于daemon/PreSessionScriptDir配置项路径下(默认为/etc/gdm/PreSession/)的所有脚本文件，来执行启动会话前的一些任务，比如更改X窗口的默认背景之类，然后它将调用位于daemon/PostLoginScriptDir配置的目录中(默认为/etc/gdm/PostLogin)的脚本，执行一些在刚刚登录以后需要运行的命令，然后它将以前面提到的desktop文件中定义的exec参数的值作为参数，调用daemon/BaseXsession配置项指定的脚本(默认为/etc/gdm/Xsession)，比如如果你选择的是默认会话，那么执行的命令将会是/etc/gdm/Xsession default,如果你查看这个文件你将发现，在这种情况下,它将首先检查是否存在主目录的.xsession文件，如果存在就执行它，否则检查是否存在主目录下的.Xclients文件，如果存在则执行它，否则就将执行/etc/X11/xinit/Xclients文件，这个文件根据 /etc/sysconfig/desktop配置文件中的设置运行各个X client，第一个X client默认为执行gnome-session。
    运行完gnome-session，我们就进入了GNOME桌面环境。而配置在daemon/PostSessionScriptDir配置项(默认值为/etc/gdm/PostSession/)所设定的目录中的脚本将在GNOME会话结束以后运行，这意味着无论出于什么原因，gnome程序已经完全退出了，也许是你选择了注销命令，也许是X窗口崩溃了，如果你有这方面的需要，可以将相应的脚本放在对应的目录中。
    2）通过startx启动图形界面
    还有一种启动图形界面的方式，就是在登录到字符界面后，通过运行startx脚本来启动X图形界面。startx并不使用gdm来启动X窗口，而是用xinit程序启动X。/usr/bin/xinit是一个二进制文件，并非是一个脚本。它的主要功能是启动一个X服务器，同时启动第一个基于X的客户端应用程序。当第一个Client退出时，xinit将杀死X server进程，然后自己终止运行。
    参考xinit的man文档，可知其用法为：xinit [[client] options ] [-- [server] [display] options]。其中client用于指定第一个X客户端应用程序，client后面的options是传给这个应用程序的参数，server是用于指定启动哪个X服务器，一般为/usr/bin/X或/usr/bin/Xorg，display用于指定display number，一般为0，表示第一个display，option为传给server的参数。
    如果不指定client，xinit会查找HOME(环境变量)目录下的.xinitrc文件，如果存在这个文件，xinit直接调用execvp函数执行该文件，以启动指定的X客户端程序。如果这个文件不存在，那么client及其options默认为： xterm -geometry +1+1 -n login -display :0 。
    如果不指定server，xinit会查找HOME(环境变量)目录下的.xserverrc 文件，如果存在这个文件，xinit直接调用execvp函数执行该文件，以启动指定的X服务器。如果这个文件不存在，那么server及其display为： X :0 。如果系统目录中不存在X命令，那么我们需要在系统目录下建立一个名为X的链接，使其指向真正的X server命令（Ubuntu下为Xorg）。
    下面是几个关于xinit应用的例子：
    （1）xinit /usr/bin/xclock -- /usr/bin/X :0
    该例子将启动X server， 同时将会启动xclock。请注意指定client或server时，需要用绝对路径，否则xinit将因无法区别是传给xterm或server的参数还是指定的client或server而直接当成是参数处理。
    （2）在HOME下新建.xinitrc文件，并加入以下几行：
[python] view plaincopy
xsetroot -solid gray &  
xclock -g 50x50-0+0 -bw 0 &  
xterm -g 80x24+0+0 &  
xterm -g 80x24+0-0 &  
twm  
[python] view plaincopy
xsetroot -solid gray &  
xclock -g 50x50-0+0 -bw 0 &  
xterm -g 80x24+0+0 &  
xterm -g 80x24+0-0 &  
twm  
    当xinit启动时，它会先启动X server，然后启动一个clock，两个xterm，最后启动窗口管理器twm。请注意最后一个命令不能后台运行，否则所有命令都后台运行的话xinit就会返回退出，同样的，除最后一个命令外都必须后台运行，否则后面的命令将只有在该命令退出后才能运行。
    看到这里，眼尖的人或许早以看出xinit的功能完全可以由脚本来实现，例如要启动X Server和一个xterm，就像xinit默认启动的那样，只需要在新建一个脚本或在rc.local中加入：
[python] view plaincopy
X&  
export DISPLAY=:0.0  
xterm  
[python] view plaincopy
X&  
export DISPLAY=:0.0  
xterm  
    这个实现完全正确，然而却并没有完全实现xinit所具有的功能，xinit的功能就是当最后一个启动的client（如上面第二个例子中的twm窗口管理器）退出后，X服务器也会退出。而我们的脚本实现中当我们退出xterm后并不会退出X server。
    startx可以看作是xinit的前端，用法和xinit的基本一样：startx [[client] options ] [-- [server] [display] options]。为什么呢？这是因为startx其实就是一个脚本，它启动X server就是通过调用xinit命令实现的，startx的参数将全部传给xinit。因此，这些参数的意义和xinit的参数是一样的。
    下面是两个关于startx命令的简单例子：
    （1）startx -- -depth 16：以16位色启动X 服务器。
    （2）startx -- -dpi 100：以100的dpi启动X 服务器。
    startx会先记录$HOME下的.xinitrc和.xserverrc文件（如果有的话），系统目录/etc/X11/xinit/下的xinitrc和.xserverrc文件。然后解析用户的参数，如果该用户指定了该参数，那么startx就会以该参数来启动xinit，否则就会解析$HOME目录下的.xinitrc和.xserverrc文件，如果该文件不存在，就会解析系统目录下（/etc/X11/xinit/）的xinitrc和xserverrc文件，如果这个文件也不存在，那 startx就将以默认的client（xterm）和server（/usr/bin/X）为参数来启动xinit。
    在Fedora 14中，只有/etc/X11/xinit/xinitrc文件，由它来运行Xclients脚本，这个脚本用于运行各个指定的X client，其中的第一个X client即为gnome-session，这就是GNOME桌面环境。从代码可知，xinitrc的功能与Xsession几乎一样，只有一些细微的差别（在Ubuntu中xinitrc是直接调用Xsession的）。

完整的Linux init程序启动过程如下图：
![image](http://hi.csdn.net/attachment/201201/2/0_1325519173X0xm.gif)
图1 Linux init程序启动过程

2、initng介绍
  由于传统的init进程（sysvinit）是一个串行化的进程，因此可对这部分系统进行充分优化。实际上，您可以使用任何方法来对init进程进行优化。其中最简单方法是禁用不必要的服务。例如，如果您运行的是一个桌面系统（而不是一个服务器），就可以禁用诸如 apache、sendmail 和 mysql 之类的服务，这样可以缩短init序列。
    其他的一些init程序版本解决了这个问题。一种是基于依赖关系的（即使用依赖关系来提供并行化，如新版的initng），一种是一个基于事件的系统（即进程依赖于事件来表示自己何时启动或停止，如upstart）。
    initng（下一代 init）可以完全取代异步启动进程的init，它能更加快速地完成init进程。initng 背后的基本思想是只要满足了服务的依赖关系就可启动。这样系统就可以在 CPU 和 I/O 之间实现较好的平衡。当从磁盘上加载一个脚本或等待硬件设备启动的同时，可以运行另一个脚本来启动另外一个服务。
    作为一个基于依赖关系的解决方案，initng 使用自己的初始化脚本集，它们对服务和守护进程的依赖性进行了编码。清单 2 展示了一个示例。这个脚本指定了需要为给定的运行级别启动的服务。该服务具有两个依赖关系，使用 need 关键字定义，分别是 system/initial 和 net/all。在 system/my_service 可以启动之前，这些服务必须是可用的。当这些服务可用时，exec 关键字就开始起作用了。exec 关键字（以及 start 选项）定义了如何使用任何可用的选项启动服务。要停止这个服务，就会使用 exec 关键字以及 stop 选项。
清单 2. 为 initng 定义服务
[python] view plaincopy
service system/my_service {  
  
  need = system/initial net/all;  
  
  exec start = /sbin/my_service --start --option;  
  exec stop = /sbin/my_service --stop --option;  
  
}  
[python] view plaincopy
service system/my_service {  
  
  need = system/initial net/all;  
  
  exec start = /sbin/my_service --start --option;  
  exec stop = /sbin/my_service --stop --option;  
  
}  
    您可以使用服务定义对整个系统进行编码，如清单 2 所示。那些没有依赖关系的服务可以立即（并行地）启动，而具有依赖关系的服务则必须等待以安全启动。您可以将 initng 看作一个基于目标的系统。其目标就是要启动的服务。没有进行显式的规划；相反，依赖关系简单地定义了服务初始化的流程，这个过程中隐含着并行化的操作。
    initng 的典型安装需要 initng 发行版（源代码或二进制文件）和 ifiles 发行版。您可以使用 ./configure、make 和 make install 编译自己的 initng 发行版。您必须使用 cmake 来编译 ifiles 文件（这是脚本文件）。根据系统需求的不同，您可能需要创建新的服务/守护进程定义（不过很可能 initng 社区中已经有人这样做了）。然后您还必须修改 LILO 或 GRUB 的配置以指向新的 /sbin/initng。要控制 initng，需要使用 ngc（对应telinit 与传统的 init）。它们的语法有些不同，不过功能是相同的。
