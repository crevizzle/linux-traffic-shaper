# Linux Traffic Shaper

Based on CentOS 7 install

## test if tc is installed on your system

you should expect some usage output
if not, install the iproute or iproute2 package

    tc

## create the shaper service

    cd /usr/lib/systemd/system
    nano shaper.service

    [Unit]
    Description="tc traffic shaper"

    [Service]
    WorkingDirectory=/usr/local/shaper
    RemainAfterExit=yes
    ExecStart=/usr/local/shaper/traffic-shaper.sh start
    ExecStop=/usr/local/shaper/traffic-shaper.sh stop

    [Install]
    WantedBy=multi-user.target

save and exit

    mkdir /usr/local/shaper
    cd /usr/local/shaper/
    touch traffic-shaper.sh
    chmod +x traffic-shaper.sh
    nano traffic-shaper.sh

    #!/bin/bash
    
    # enable tc traffic shaping for speedtests
    
    # current config applies one rate to all traffic
    # flowids tie your matched traffic to classids, so tc knows what traffic rate to allow
    
    # If your speedtest server is commonly sending close to RATE of traffic, then you might want to consider
    # multiple class filters for groups of customer endpoint networks. Then make RATE the line rate of your server's interface
    # otherwise, your customers may soak the single 990mbit traffic rate and not get accurate speedtests
    # this possiblity assumes the link to your server is either a 10gig link or possibly multiple 1gigs in a bond
    # To do this, comment out the default filter statement, define CLASSRATE for class bandwidth rates.
    # And either define your customer networks with more variables, or manually enter IP networks in the filter statements
    
    ##### Multi Class Example
    # TC=/sbin/tc
    # IF=p1p1
    # RATE=10gbit
    # CLASSRATE=990mbit
    # NET2=192.168.200.0/24
    #
    # start() {
    #   $TC qdisc add dev $IF handle 1: root htb
    #   $TC class add dev $IF parent 1: classid 1:1 htb rate $RATE
    #   $TC class add dev $IF parent 1:0 classid 1:10 htb rate $CLASSRATE
    #   $TC class add dev $IF parent 1:0 classid 1:20 htb rate $CLASSRATE
    #   $TC filter add dev $IF parent 1:0 protocol ip prio 1 u32 match ip dst 192.168.100.0/24 flowid 1:10
    #   $TC filter add dev $IF parent 1:0 protocol ip prio 1 u32 match ip dst $NET2 flowid 1:20
    # }
    
    TC=/sbin/tc
    # change IF to whatever interface your server is using - ex em1, ens160, etc
    IF=p1p1
    RATE=990mbit
    IP=0.0.0.0/0
    
    start() {
      $TC qdisc add dev $IF handle 1: root htb
      $TC class add dev $IF parent 1: classid 1:1 htb rate $RATE
      $TC filter add dev $IF parent 1:0 protocol ip prio 1 u32 match ip dst $IP flowid 1:1
    }
    
    stop() {
      $TC qdisc del dev $IF root
    }
    
    restart() {
      stop
      sleep 1
      start
    }
    
    # show current statistics
    
    # only used when manually running this file, not systemctl - ex /usr/local/shaper/traffic-shaper.sh show
    
    show() {
      $TC -s qdisc ls dev $IF
      $TC -s class ls dev $IF
    }
    
    case "$1" in
      start)
        echo -n "Starting bandwidth shaper... "
        start
        echo -n "Started"
        ;;
    
      stop)
        echo -n "Stopping bandwidth shaper... "
        stop
        echo -n "Stopped"
        ;;
    
      show)
        show
        ;;
    
      *)
    
    esac
    
    exit 0

save and exit

## start the service and enable at boot

    systemctl daemon-reload
    systemctl enable shaper.service
    systemctl start shaper.service
    systemctl status shaper.service
