#!/usr/bin/env bash
#
# getsysperf.com
# A simple benchmark script to test your Disk, Network and CPU performance
#


#
#
# VALIDATION
#
#

# Check if at least Debian 10 or Ubuntu 1904 is used
if grep -qs "ubuntu" /etc/os-release; then
	os="ubuntu"
	os_version=$(grep 'VERSION_ID' /etc/os-release | cut -d '"' -f 2 | tr -d '.')
	group_name="nogroup"
elif [[ -e /etc/debian_version ]]; then
	os="debian"
	os_version=$(grep -oE '[0-9]+' /etc/debian_version | head -1)
	group_name="nogroup"
else
	echo "Looks like you aren't running this installer on Debian or Ubuntu"
	exit
fi
if [[ "$os" == "ubuntu" && "$os_version" -lt 1804 ]]; then
	echo "Ubuntu 18.04 or higher is required to use this installer
This version of Ubuntu is too old and unsupported"
	exit
fi

if [[ "$os" == "debian" && "$os_version" -lt 10 ]]; then
	echo "Debian 10 or higher is required to use this installer
This version of Debian is too old and unsupported"
	exit
fi

# Check if user is root
if [[ "$EUID" -ne 0 ]]; then
	echo "Sorry, you need to run this as root"
	exit
fi

# Check if the required packages are installed
if  [ ! -e '/usr/bin/wget' ] || [ ! -e '/usr/bin/fio' ] || [ ! -e '/usr/bin/curl' ] || [ ! -e '/usr/bin/jq' ]; then
    echo "Couldn't find [wget, fio, curl, jq]"    	
    read -n 1 -r -s -p  "Please press enter to install the required packages automatically" 
   
   apt update && apt install -y curl jq fio wget
fi




# Test IPv6 connectivity
ipv6=$( wget -qO- -t1 -T2 ipv6.icanhazip.com )
# Get public IP for ASN/ ISP check
as_check=$( wget -qO- -t1 -T2 icanhazip.com )


# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;36m'
PLAIN='\033[0m'


#
#
# FUNCTIONS
#
#

get_netinfo() {
    isp=$(curl -s http://ip-api.com/json/$as_check | jq '.isp' | sed 's/"//g')
    as=$(curl -s http://ip-api.com/json/$as_check | jq '.as' | sed 's/"//g')
}

get_opsy() {
    [ -f /etc/redhat-release ] && awk '{print ($1,$3~/^[0-9]/?$3:$4)}' /etc/redhat-release && return
    [ -f /etc/os-release ] && awk -F'[= "]' '/PRETTY_NAME/{print $3,$4,$5}' /etc/os-release && return
    [ -f /etc/lsb-release ] && awk -F'[="]+' '/DESCRIPTION/{print $2}' /etc/lsb-release && return
}
get_sys_type() {
    if [ $(systemd-detect-virt) == none ]; then 
            sys_type="Baremetal"
        elif [ $(systemd-detect-virt) == kvm ]; then
            sys_type="KVM"
        elif [ $(systemd-detect-virt) == lxc ]; then
            sys_type="LXC"
        elif [ $(systemd-detect-virt) == openvz ]; then
            sys_type="OpenVZ"
    fi
}

next() {
    printf "%-5s\n" "-" | sed 's/\s/-/g'
}

speed_test() {
    local output=$(LANG=C wget -O /dev/null -T30 $1 2>&1)
    local speedtest=$(printf '%s' "$output" | awk '/\/dev\/null/ {speed=$3 $4} END {gsub(/\(|\)/,"",speed); print speed}')
    local nodeName=$2
    printf "${YELLOW}%-32s${GREEN}%-24s${RED}%-14s${PLAIN}\n" "${nodeName}" "${speedtest}"
}

speed_result() {
    speed_test 'http://speedtest.frankfurt.linode.com/100MB-frankfurt.bin' 'Frankfurt, Linode'
    speed_test 'https://fra-de-ping.vultr.com/vultr.com.100MB.bin' 'Frankfurt, Vultr'
    speed_test 'http://fra36-speedtest-1.tele2.net/100MB.zip' 'Frankfurt, TELE2'
    speed_test 'https://lg.combahton.net/100MB.test' 'Frankfurt, Combahton'
    speed_test 'http://fsn.icmp.hetzner.com/100MB.bin' 'Falkenstein, Hetzner'
    speed_test 'https://speed.hetzner.de/100MB.bin' 'Nuremberg, Hetzner'
    speed_test 'http://hel.icmp.hetzner.com/100MB.bin' 'Helsinki, Hetzner'
    speed_test 'http://ash.icmp.hetzner.com/100MB.bin' 'Ashburn, Hetzner'
    speed_test 'http://speedtest-ams2.digitalocean.com/100mb.test' 'Amsterdam, Digitalocean'
    speed_test 'https://ams.lg.core-backbone.com/files/100MB.test' 'Amsterdam, Core-Backbone'
    speed_test 'http://speedtest.london.linode.com/100MB-london.bin' 'London, Linode'
    speed_test 'https://par-fr-ping.vultr.com/vultr.com.100MB.bin' 'Paris, Vultr'
    speed_test 'http://speedtest.newark.linode.com/100MB-newark.bin' 'Newark, Linode'
    speed_test 'http://speedtest.fremont.linode.com/100MB-newark.bin' 'Fremont, Linode'
    speed_test 'https://tx-us-ping.vultr.com/vultr.com.100MB.bin' 'Texas, Vultr'
    speed_test 'http://speedtest.singapore.linode.com/100MB-singapore.bin' 'Singapore, Linode'
}

ping_test() {
    local nodename=$2
    local output=$(ping -w 2 $1 | grep rtt | cut -d'/' -f4 | awk '{ print $3 }')
    local output="$output ms"
    printf "${YELLOW}%-32s${GREEN}%-24s${RED}%-14s${PLAIN}\n" "${nodename}" "${output}"
}

ping_result() {
  ping_test 'speedtest.frankfurt.linode.com' 'Frankfurt'
  ping_test 'speedtest.newark.linode.com' 'New York'
  ping_test 'speedtest.fremont.linode.com' 'San Francisco'
  ping_test 'speedtest.singapore.linode.com' 'Singapore'
  ping_test 'speedtest.tokyo2.linode.com' 'Tokyo'
}

io_test() {
   # Run rand read/write mixed 512kb fio test
   fio --name=rand_rw_512k --ioengine=libaio --rw=randrw --rwmixread=50 --bs=512k --iodepth=64 --numjobs=2 --size=2G --runtime=30 --gtod_reduce=1 --direct=1 --filename=$RANDOM"_IOTEST" | grep "WRITE:" | awk '{print $3}' | sed 's/,//g' | sed 's/(//g' | sed 's/)//g'
   rm *_IOTEST
   sleep 10
}


geekbench5() {
  # Check if system has at least 2GB RAM in order to run Geekbench 5
  ram=$(free -m | awk '/Mem/ {print $2}')
  if [ $ram -lt 2048 ]
  then
      echo "RAM:" $ram"MB"
      echo "Error: Not enough memory available. Geekbench 5 requires as least 2GB"
      exit 1
  fi
	echo -e "Running Geekbench5 CPU Benchmark. This will take a few minutes..."
	GEEKBENCH_PATH=$HOME/geekbench
	mkdir -p $GEEKBENCH_PATH
	curl -s http://cdn.geekbench.com/Geekbench-5.1.0-Linux.tar.gz  | tar xz --strip-components=1 -C $GEEKBENCH_PATH
	GEEKBENCH_TEST=$($GEEKBENCH_PATH/geekbench5 | grep "https://browser")
	GEEKBENCH_URL=$(echo -e $GEEKBENCH_TEST | head -1)
	GEEKBENCH_URL_CLAIM=$(echo $GEEKBENCH_URL | awk '{ print $2 }')
	GEEKBENCH_URL=$(echo $GEEKBENCH_URL | awk '{ print $1 }')
	sleep 10
	GEEKBENCH_SCORES_SINGLE=$(curl -s $GEEKBENCH_URL | grep "class='score'" | head -2 | tr -d \'\" | sed 's/class=score//g' | sed "s/ //g" | sed -n 's:.*<div>\(.*\)</div>.*:\1:p' | head -n 1)
	GEEKBENCH_SCORES_MULTI=$(curl -s $GEEKBENCH_URL | grep "class='score'" | head -2 | tr -d \'\" | sed 's/class=score//g' | sed "s/ //g" | sed -n 's:.*<div>\(.*\)</div>.*:\1:p' | sed -n 2p)
	echo -ne "\e[1A"; echo -ne "\033[0K\r"
	echo -e "Geekbench v5"
	echo -e "Single Core          :" ${BLUE} $GEEKBENCH_SCORES_SINGLE ${PLAIN}
	echo -e "Multi Core           :" ${YELLOW} $GEEKBENCH_SCORES_MULTI ${PLAIN}
	rm -rf geekbench/
}

calc_disk() {
    local total_size=0
    local array=$@
    for size in ${array[@]}
    do
        [ "${size}" == "0" ] && size_t=0 || size_t=`echo ${size:0:${#size}-1}`
        [ "`echo ${size:(-1)}`" == "K" ] && size=0
        [ "`echo ${size:(-1)}`" == "M" ] && size=$( awk 'BEGIN{printf "%.1f", '$size_t' / 1024}' )
        [ "`echo ${size:(-1)}`" == "T" ] && size=$( awk 'BEGIN{printf "%.1f", '$size_t' * 1024}' )
        [ "`echo ${size:(-1)}`" == "G" ] && size=${size_t}
        total_size=$( awk 'BEGIN{printf "%.1f", '$total_size' + '$size'}' )
    done
    echo ${total_size}
}

cname=$( awk -F: '/model name/ {name=$2} END {print name}' /proc/cpuinfo | sed 's/^[ \t]*//;s/[ \t]*$//' )
cores=$( awk -F: '/model name/ {core++} END {print core}' /proc/cpuinfo )
freq=$( awk -F: '/cpu MHz/ {freq=$2} END {print freq}' /proc/cpuinfo | sed 's/^[ \t]*//;s/[ \t]*$//' )
tram=$( free -m | awk '/Mem/ {print $2}' )
uram=$( free -m | awk '/Mem/ {print $3}' )
swap=$( free -m | awk '/Swap/ {print $2}' )
uswap=$( free -m | awk '/Swap/ {print $3}' )
up=$( awk '{a=$1/86400;b=($1%86400)/3600;c=($1%3600)/60} {printf("%d days, %d hour %d min\n",a,b,c)}' /proc/uptime )
load=$( w | head -1 | awk -F'load average:' '{print $2}' | sed 's/^[ \t]*//;s/[ \t]*$//' )
opsy=$( get_opsy )
arch=$( uname -m )
lbit=$( getconf LONG_BIT )
kern=$( uname -r )
disk_size1=($( LANG=C df -hPl | grep -wvE '\-|none|tmpfs|devtmpfs|by-uuid|chroot|Filesystem|udev|docker' | awk '{print $2}' ))
disk_size2=($( LANG=C df -hPl | grep -wvE '\-|none|tmpfs|devtmpfs|by-uuid|chroot|Filesystem|udev|docker' | awk '{print $3}' ))
disk_total_size=$( calc_disk "${disk_size1[@]}" )
disk_used_size=$( calc_disk "${disk_size2[@]}" )

clear
next
echo -e "${RED}getsysperf.com${PLAIN} -- ${YELLOW}https://github.com/iandk/sysperf${PLAIN}"
next
get_netinfo
get_sys_type
echo -e "System type          : ${BLUE}$sys_type${PLAIN}"
echo -e "CPU model            : ${BLUE}$cname${PLAIN}"
echo -e "Number of cores      : ${BLUE}$cores${PLAIN}"
echo -e "CPU frequency        : ${BLUE}$freq MHz${PLAIN}"
echo -e "Total size of Disk   : ${BLUE}$disk_total_size GB ($disk_used_size GB Used)${PLAIN}"
echo -e "Total amount of Mem  : ${BLUE}$tram MB ($uram MB Used)${PLAIN}"
echo -e "Total amount of Swap : ${BLUE}$swap MB ($uswap MB Used)${PLAIN}"
echo -e "System uptime        : ${BLUE}$up${PLAIN}"
echo -e "Load average         : ${BLUE}$load${PLAIN}"
echo -e "OS                   : ${BLUE}$opsy${PLAIN}"
echo -e "Arch                 : ${BLUE}$arch ($lbit Bit)${PLAIN}"
echo -e "Kernel               : ${BLUE}$kern${PLAIN}"
echo -e "ISP                  : ${BLUE}$isp${PLAIN}"
echo -e "ASN                  : ${BLUE}$as${PLAIN}"
if [[ "$ipv6" != "" ]]; then
echo -e "IPv6 Support         : ${BLUE}Yes${PLAIN}"
else
echo -e "IPv6 Support         : ${BLUE}No${PLAIN}"
fi
next
io1=$( io_test )
echo -e "I/O bandwidth (1st run)  : ${YELLOW}$io1${PLAIN}"
io2=$( io_test )
echo -e "I/O bandwidth (2nd run)  : ${YELLOW}$io2${PLAIN}"
io3=$( io_test )
echo -e "I/O bandwidth (3rd run)  : ${YELLOW}$io3${PLAIN}"
rawio1=$( echo $io1 | sed 's/MB//g' | sed 's/s//g' | sed 's/.$//')
rawio2=$( echo $io2 | sed 's/MB//g' | sed 's/s//g' | sed 's/.$//')
rawio3=$( echo $io3 | sed 's/MB//g' | sed 's/s//g' | sed 's/.$//')
ioall=$( awk 'BEGIN{print '$rawio1' + '$rawio2' + '$rawio3'}' )
ioavg=$( awk 'BEGIN{printf "%.1f", '$ioall' / 3}' )
echo -e "Average I/O bandwidth    : ${YELLOW}$ioavg MB/s${PLAIN}"
next
printf "%-32s%-24s%-14s\n" "Location" "Latency"
ping_result
next
printf "%-32s%-24s%-14s\n" "Location" "Speed"
    
speed_result && next
geekbench5 && next
