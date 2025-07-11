BGP on loobpack
dynamic BGP for shorcuts
Hub-side steering BGP priority from SLA metrics in per-overlay Branch SD-WAN probes
No remote signaling to WEST-CORE

# ----

I validated in Lab a "light segmentation" where traffic segmentation is based on firewall policies (ie, no VRFs).

I built a solution based on:
- BGP Route Target extended community to tag the subnets of every segments
- firewall addresses of type route-tag to group all subnets of the same segment inside a fw address object
- firewall policies using the route-tag fw address to ensure traffic is only allowed between subnets in the same segment
    . route-tag addresses can only be used as dstaddr, not as scraddr
    . so the intra-segment control is done "close to the source of traffic"
 
Only intra-segment traffic is allowed.
It is however possible to configure fw policies to allow inter-segment traffic:
    . WEST-DC1 10.1.0.7 is not tagged with any color. It is recheable from BLUE/RED/YELLOW segments
 
ADVPN is supported (intra-regional and inter-regional)


# WEST-BR1 ###################################################################

config router bgp
    config neighbor
        edit "10.200.1.254"
            set route-map-in "TAG_SEGMENTS"
        next
        edit "10.200.1.253"
            set route-map-in "TAG_SEGMENTS"
        next
    end
    config neighbor-group
        edit "ADVPN_WEST"
            set route-map-in "TAG_SEGMENTS"
        next
        edit "ADVPN_REGIONS"
            set route-map-in "TAG_SEGMENTS"
            set route-map-out "LOCAL_LAN_ONLY"
        next
    end
    config neighbor-range
        edit 1
            set prefix 10.200.1.0 255.255.255.0
            set neighbor-group "ADVPN_WEST"
        next
        edit 2
            set prefix 10.200.0.0 255.255.0.0
            set neighbor-group "ADVPN_REGIONS"
        next
    end
    config network
        edit 1
            set prefix 10.0.1.0 255.255.255.0
            set route-map "SET_LOCAL_LAN_BLUE"
        next
        edit 2
            set prefix 10.0.11.0 255.255.255.0
            set route-map "SET_LOCAL_LAN_YELLOW"
        next
        edit 3
            set prefix 10.0.12.0 255.255.255.0
            set route-map "SET_LOCAL_LAN_RED"
        next
    end
end

config router route-map
    edit "LOCAL_LAN_ONLY"
        config rule
            edit 1
                set match-tag 32768
            next
        end
    next
    edit "SET_LOCAL_LAN_YELLOW"
        config rule
            edit 1
                set set-extcommunity-rt "65000:11"
                set set-tag 32768
            next
        end
    next
    edit "SET_LOCAL_LAN_RED"
        config rule
            edit 1
                set set-extcommunity-rt "65000:12"
                set set-tag 32768
            next
        end
    next
    edit "SET_LOCAL_LAN_BLUE"
        config rule
            edit 1
                set set-extcommunity-rt "65000:13"
                set set-tag 32768
            next
        end
    next
    edit "TAG_SEGMENTS"
        config rule
            edit 1
                set match-extcommunity "BLUE_COMMUNITY"
                set set-route-tag 13
            next
            edit 2
                set match-extcommunity "YELLOW_COMMUNITY"
                set set-route-tag 11
            next
            edit 3
                set match-extcommunity "RED_COMMUNITY"
                set set-route-tag 12
            next
            edit 4
            next
        end
    next
end

config router extcommunity-list
    edit "BLUE_COMMUNITY"
        config rule
            edit 1
                set action permit
                set match "65000:13"
            next
        end
    next
    edit "YELLOW_COMMUNITY"
        config rule
            edit 1
                set action permit
                set match "65000:11"
            next
        end
    next
    edit "RED_COMMUNITY"
        config rule
            edit 1
                set action permit
                set match "65000:12"
            next
        end
    next
end

config firewall address
    edit "LAN_BLUE"
        set subnet 10.0.1.0 255.255.255.0
    next
    edit "LAN_YELLOW"
        set subnet 10.0.11.0 255.255.255.0
    next
    edit "LAN_RED"
        set subnet 10.0.12.0 255.255.255.0
    next
    edit "BLUE_SEGMENTS"
        set type route-tag
        set route-tag 13
    next
    edit "YELLOW_SEGMENTS"
        set type route-tag
        set route-tag 11
    next
    edit "RED_SEGMENTS"
        set type route-tag
        set route-tag 12
    next
    edit "WEST-DC1-BLUE"
        set subnet 10.1.0.7 255.255.255.255
    next
end

config firewall policy
    edit 10
        set name "BGP and ADVPN HC"
        set srcintf "VPN"
        set dstintf "lo-BGP"
        set action accept
        set srcaddr "Corporate"
        set dstaddr "Corporate"
        set schedule "always"
        set service "BGP" "PING"
        set logtraffic all
        set global-label "VPN in"
    next
    edit 11
        set name "VPN In"
        set srcintf "VPN"
        set dstintf "port5" "LAN_RED" "LAN_YELLOW"
        set action accept
        set srcaddr "Corporate"
        set dstaddr "LAN_BLUE" "LAN_YELLOW" "LAN_RED"
        set schedule "always"
        set service "ALL"
        set logtraffic all
        set comments "no UTM inbound - it is applied outbound"
    next
    edit 12
        set name "BLUE Out"
        set srcintf "port5"
        set dstintf "VPN"
        set action accept
        set srcaddr "LAN_BLUE"
        set dstaddr "BLUE_SEGMENTS"
        set schedule "always"
        set service "ALL"
        set utm-status enable
        set ssl-ssh-profile "deep-inspection"
        set application-list "default"
        set logtraffic all
        set global-label "BLUE out"
    next
    edit 13
        set name "YELLOW Out"
        set srcintf "LAN_YELLOW"
        set dstintf "VPN"
        set action accept
        set srcaddr "LAN_YELLOW"
        set dstaddr "YELLOW_SEGMENTS"
        set schedule "always"
        set service "ALL"
        set utm-status enable
        set ssl-ssh-profile "deep-inspection"
        set application-list "default"
        set logtraffic all
        set global-label "YELLOW out"
    next
    edit 14
        set name "RED Out"
        set srcintf "LAN_RED"
        set dstintf "VPN"
        set action accept
        set srcaddr "LAN_RED"
        set dstaddr "RED_SEGMENTS"
        set schedule "always"
        set service "ALL"
        set utm-status enable
        set ssl-ssh-profile "deep-inspection"
        set application-list "default"
        set logtraffic all
        set global-label "RED out"
    next
    edit 15
        set name "WEST-DC1-BLUE"
        set srcintf "LAN_RED" "LAN_YELLOW" "port5"
        set dstintf "VPN"
        set action accept
        set srcaddr "LAN_RED" "LAN_YELLOW" "LAN_BLUE"
        set dstaddr "WEST-DC1-BLUE"
        set schedule "always"
        set service "ALL"
        set utm-status enable
        set ssl-ssh-profile "deep-inspection"
        set application-list "default"
        set logtraffic all
        set global-label "MULTI-COLORS out"
    next
end


# WEST-DC1 ###################################################################

config router bgp
    config neighbor
        edit "10.200.2.254"
            set route-map-in "TAG_SEGMENTS"
        next
    end
    config neighbor-group
        edit "EDGE"
            set route-reflector-client enable
            set next-hop-self-rr enable
            set route-map-in "TAG_SEGMENTS"
        next
        edit "ADVPN_REGIONS"
            set route-map-in "TAG_SEGMENTS"
            set route-map-out "LOCAL_LAN_ONLY"
        next
    end
    config neighbor-range
        edit 1
            set prefix 10.200.1.0 255.255.255.0
            set neighbor-group "EDGE"
        next
        edit 2
            set prefix 10.200.0.0 255.255.0.0
            set neighbor-group "ADVPN_REGIONS"
        next
    end
    config network
        edit 1
            set prefix 10.200.1.0 255.255.255.0
        next
        edit 2
            set prefix 10.1.0.0 255.255.255.0   # multi-colors subnet
        next
        edit 3
            set prefix 10.1.1.0 255.255.255.0
            set route-map "SET_LOCAL_LAN_YELLOW"
        next
        edit 4
            set prefix 10.1.2.0 255.255.255.0
            set route-map "SET_LOCAL_LAN_RED"
        next
    end
end

config firewall policy
    edit 10
        set name "BGP and SDWAN HC"
        set srcintf "BRANCHES" "REGIONS"
        set dstintf "lo-BGP" "lo-HC"
        set action accept
        set srcaddr "Corporate"
        set dstaddr "Corporate"
        set schedule "always"
        set service "PING" "BGP"
        set logtraffic all
        set global-label "VPN in"
    next
    edit 11
        set name "VPN <-> VPN"
        set srcintf "BRANCHES" "REGIONS"
        set dstintf "BRANCHES" "REGIONS"
        set action accept
        set srcaddr "Corporate"
        set dstaddr "Corporate"
        set schedule "always"
        set service "ALL"
        set anti-replay disable
        set tcp-session-without-syn all
        set logtraffic all
        set logtraffic-start enable
    next
    edit 12
        set name "VPN->DC"
        set srcintf "BRANCHES" "REGIONS"
        set dstintf "port5" "LAN_RED" "LAN_YELLOW"
        set action accept
        set srcaddr "Corporate"
        set dstaddr "LAN_BLUE" "LAN_YELLOW" "LAN_RED"
        set schedule "always"
        set service "ALL"
        set anti-replay disable
        set tcp-session-without-syn all
        set logtraffic all
        set nat enable
    next
    edit 13
        set name "BLUE Out"
        set srcintf "port5"
        set dstintf "BRANCHES" "REGIONS"
        set action accept
        set srcaddr "Corporate"
        set dstaddr "BLUE_SEGMENTS"
        set schedule "always"
        set service "ALL"
        set anti-replay disable
        set tcp-session-without-syn all
        set utm-status enable
        set ssl-ssh-profile "deep-inspection"
        set application-list "default"
        set logtraffic all
        set global-label "BLUE out"
    next
    edit 14
        set name "YELLOW Out"
        set srcintf "LAN_YELLOW"
        set dstintf "BRANCHES" "REGIONS"
        set action accept
        set srcaddr "Corporate"
        set dstaddr "YELLOW_SEGMENTS"
        set schedule "always"
        set service "ALL"
        set anti-replay disable
        set tcp-session-without-syn all
        set utm-status enable
        set ssl-ssh-profile "deep-inspection"
        set application-list "default"
        set logtraffic all
        set global-label "YELLOW out"
    next
    edit 15
        set name "RED Out"
        set srcintf "LAN_RED"
        set dstintf "BRANCHES" "REGIONS"
        set action accept
        set srcaddr "Corporate"
        set dstaddr "RED_SEGMENTS"
        set schedule "always"
        set service "ALL"
        set anti-replay disable
        set tcp-session-without-syn all
        set utm-status enable
        set ssl-ssh-profile "deep-inspection"
        set application-list "default"
        set logtraffic all
        set global-label "RED out"
    next
    edit 16
        set name "WEST-DC1-BLUE"
        set srcintf "port5"
        set dstintf "BRANCHES" "REGIONS"
        set action accept
        set srcaddr "WEST-DC1-BLUE"
        set dstaddr "Corporate"
        set schedule "always"
        set service "ALL"
        set anti-replay disable
        set tcp-session-without-syn all
        set utm-status enable
        set ssl-ssh-profile "deep-inspection"
        set application-list "default"
        set logtraffic all
        set global-label "MULTI-COLORS out"
    next
end
