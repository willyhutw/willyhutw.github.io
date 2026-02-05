---
date: 2024-04-25T22:43:00+08:00
title: 'pfSense Filterlog Collection'
type: 'post'
tags: ['pfsense', 'syslog', 'fluentd', 'loki', 'grafana']
draft: false
---

```goat
                              .------.   datasource   .---------.
                              | Loki |<---------------+ Grafana |
                              '------'                '---------'
                                ^
                                | write
                                |
.---------.   rsyslog     .-----+---.
| pfSense +-------------->| fluentd |
'---------'               '---------'
```

## Introduction
This post is about pfSense syslog(especially to filterlog) collection and visualization by using ***Fluentd***, ***Loki*** and ***Grafana***. We assume you already have running fluentd, loki and grafana instances. The following instruction won't cover any installation.

## Run fluentd and check the syslog of pfSense
1. First of all, we prepare a very basic fluentd config file (`~/fluentd/fluent.conf`) for receiving pfsense syslog and printing them out.
    ```text
    <source>
      ### https://docs.fluentd.org/input/syslog
      @type syslog
      port 5141
      bind 0.0.0.0
      <transport udp>
      </transport>
      <parse>
        message_format rfc5424
      </parse>
      tag pf
    </source>

    <match pf>
      @type stdout
    </match>
    ```
2. Run fluentd with the config file.
    ```bash
    ### https://docs.fluentd.org/installation
    /opt/fluent/bin/fluentd -c ~/fluentd/fluent.conf
    ```
3. Enable and configure the remote logging on pfsense host.
    - Status >> System Logs >> Settings >> General Logging Options 
        - Log Message Format: `syslog (RFC 5424, with RFC 3339 ...)`
    - Status >> System Logs >> Settings >> Remote Logging Options
        - Remote Log Servers: `fluentd_listening_address:port (Ex: 192.168.1.99:5141)`
        - Remote Syslog Contents: `Firewall Events`
4. Now you should see the log from pfsense keep comming in.
    ```text
    2024-04-25 06:09:34.632899000 +0000 pfsense.local0.info: {"host":"pfsense.willyhu.local","ident":"filterlog","pid":"44226","msgid":"-","extradata":"-","message":"4,,,1000000103,pppoe0,match,block,in,4,0x0,,242,26160,0,none,6,tcp,44,89.248.165.17,125.229.96.130,44961,30129,0,S,3258086147,,1025,,mss"}
    2024-04-25 06:09:41.934155000 +0000 pfsense.local0.info: {"host":"pfsense.willyhu.local","ident":"filterlog","pid":"44226","msgid":"-","extradata":"-","message":"4,,,1000000103,pppoe0,match,block,in,4,0x0,,233,61911,0,none,6,tcp,40,45.141.87.109,125.229.96.130,44210,5040,0,S,246121189,,1024,,"}
    2024-04-25 06:09:42.959281000 +0000 pfsense.local0.info: {"host":"pfsense.willyhu.local","ident":"filterlog","pid":"44226","msgid":"-","extradata":"-","message":"4,,,1000000103,pppoe0,match,block,in,4,0x0,,107,19362,0,DF,6,tcp,52,177.229.216.18,125.229.96.130,51305,445,0,S,1581211380,,8192,,mss;nop;wscale;nop;nop;sackOK"}
    ```
5. As we've configured the message format(rfc5424) in fluentd config, it has already extracted several fields and values for us. However, as you can see. We still have to parse the `message` field before insert them into Loki server.
    - host: `pfsense.willyhu.local`
    - ident: `filterlog`
    - pid: `44226`
    - msgid: `-`
    - extradata: `-`
    - message: `4,,,1000000103,pppoe0,match,block,in,4,0x0,,111,23330,0,DF,6,tcp,52,115.73.209.120,125.229.96.130,56735,445,0,S,2152994813,,8192,,mss;nop;wscale;nop;nop;sackOK`

## Parsing the raw message of filterlog
Every kind of log has its own format. In this case, we can find the very detailed explanation of filterlog format at [pfsense official document](https://docs.netgate.com/pfsense/en/latest/monitoring/logs/raw-filter-format.html). That `<CSV data>` in the document is actually the value of the `message` field. So, let's break it down!

1. Install extra fluentd plugins: `rewrite_tag_filter` and `filter_record_modifier`.
    ```bash
    sudo /opt/fluent/bin/fluent-gem install fluent-plugin-rewrite-tag-filter
    sudo /opt/fluent/bin/fluent-gem install fluent-plugin-record-modifier
    ```
2. Add following filters to `~/fluentd/fluent.conf` after `<source>` section.
    ```text
    ### add log_type field
    <filter pf>
      @type record_modifier
      <record>
        log_type ${record['ident']}
      </record>
    </filter>

    ### rewrite tag: pf.${log_type}
    <match pf>
      @type rewrite_tag_filter
      <rule>
        key log_type
        pattern ^(.+)$
        tag ${tag}.$1
      </rule>
    </match>

    ### parsing ip_version and protocol_text for later tag rewrite.
    <filter pf.filterlog>
      @type parser
      key_name message
      reserve_data true
      reserve_time true
      <parse>
        @type regexp
        expression /^(?:[^,]*,){8}(?<ip_version>[^,]*),(?:[^,]*,){7}(?<protocol_text>[^,]*)/
      </parse>
    </filter>

    ### rewrite tag: pf.${ident}.${ip_version}
    <match pf.filterlog>
      @type rewrite_tag_filter
      <rule>
        key ip_version
        pattern ^(.+)$
        tag ${tag}.$1
      </rule>
    </match>

    ### rewrite tag: pf.${ident}.${ip_version}.${protocol_text}
    <match pf.filterlog.*>
      @type rewrite_tag_filter
      <rule>
        key protocol_text
        pattern ^(.+)$
        tag ${tag}.$1
      </rule>
    </match>

    ### parsing filterlog: ipv4/tcp
    <filter pf.filterlog.4.tcp>
      @type parser
      key_name message
      reserve_time true
      <parse>
        @type regexp
        expression /^(?<rule_number>[^,]*),(?<sub_rule_number>[^,]*),(?<anchor>[^,]*),(?<tracker>[^,]*),(?<real_interface>[^,]*),(?<reason>[^,]*),(?<action>[^,]*),(?<direction>[^,]*),(?<ip_version>[^,]*),(?<tos>[^,]*),(?<ecn>[^,]*),(?<ttl>[^,]*),(?<id>[^,]*),(?<offset>[^,]*),(?<flags>[^,]*),(?<protocol_id>[^,]*),(?<protocol_text>[^,]*),(?<length>[^,]*),(?<source_address>[^,]*),(?<destination_address>[^,]*),(?<source_port>[^,]*),(?<destination_port>[^,]*),(?<data_length>[^,]*),(?<tcp_flags>[^,]*),(?<sequence_number>[^,]*),(?<ack_number>[^,]*),(?<tcp_window>[^,]*),(?<urg>[^,]*),(?<tcp_options>[^,]*)/
      </parse>
    </filter>

    ### parsing filterlog: ipv4/udp
    <filter pf.filterlog.4.udp>
      @type parser
      key_name message
      reserve_time true
      <parse>
        @type regexp
        expression /^(?<rule_number>[^,]*),(?<sub_rule_number>[^,]*),(?<anchor>[^,]*),(?<tracker>[^,]*),(?<real_interface>[^,]*),(?<reason>[^,]*),(?<action>[^,]*),(?<direction>[^,]*),(?<ip_version>[^,]*),(?<tos>[^,]*),(?<ecn>[^,]*),(?<ttl>[^,]*),(?<id>[^,]*),(?<offset>[^,]*),(?<flags>[^,]*),(?<protocol_id>[^,]*),(?<protocol_text>[^,]*),(?<length>[^,]*),(?<source_address>[^,]*),(?<destination_address>[^,]*),(?<source_port>[^,]*),(?<destination_port>[^,]*),(?<data_length>[^,]*)/
      </parse>
    </filter>

    ### parsing filterlog: ipv6/tcp
    <filter pf.filterlog.6.tcp>
      @type parser
      key_name message
      reserve_time true
      <parse>
        @type regexp
        expression /^(?<rule_number>[^,]*),(?<sub_rule_number>[^,]*),(?<anchor>[^,]*),(?<tracker>[^,]*),(?<real_interface>[^,]*),(?<reason>[^,]*),(?<action>[^,]*),(?<direction>[^,]*),(?<ip_version>[^,]*),(?<class>[^,]*),(?<flow_label>[^,]*),(?<hop_limit>[^,]*),(?<protocol_text>[^,]*),(?<protocol_id>[^,]*),(?<length>[^,]*),(?<source_address>[^,]*),(?<destination_address>[^,]*),(?<source_port>[^,]*),(?<destination_port>[^,]*),(?<data_length>[^,]*),(?<tcp_flags>[^,]*),(?<sequence_number>[^,]*),(?<ack_number>[^,]*),(?<tcp_window>[^,]*),(?<urg>[^,]*),(?<tcp_options>[^,]*)/
      </parse>
    </filter>

    ### parsing filterlog: ipv6/udp
    <filter pf.filterlog.6.udp>
      @type parser
      key_name message
      reserve_time true
      <parse>
        @type regexp
        expression /^(?<rule_number>[^,]*),(?<sub_rule_number>[^,]*),(?<anchor>[^,]*),(?<tracker>[^,]*),(?<real_interface>[^,]*),(?<reason>[^,]*),(?<action>[^,]*),(?<direction>[^,]*),(?<ip_version>[^,]*),(?<class>[^,]*),(?<flow_label>[^,]*),(?<hop_limit>[^,]*),(?<protocol_text>[^,]*),(?<protocol_id>[^,]*),(?<length>[^,]*),(?<source_address>[^,]*),(?<destination_address>[^,]*),(?<source_port>[^,]*),(?<destination_port>[^,]*),(?<data_length>[^,]*)/
      </parse>
    </filter>

    ### remove unused fields in this case
    <filter pf.**>
      @type record_modifier
      remove_keys message,host,pri,ident,pid,msgid,extradata,rule_number,sub_rule_number,anchor,tracker,reason,tos,ecn,ttl,id,offset,flags,protocol_id,length,source_port,data_length,sequence_number,ack_number,tcp_window,urg,tcp_options,class,flow_label,hop_limit
    </filter>

    ### print out results
    <match pf.**>
      @type stdout
    </match>
    ```
3. Re-run fluentd again you should see the message has been extracted (exclude the fields we deleted) to key/value pair.
    - real_interface: `pppoe0`
    - action: `block`
    - direction: `in`
    - ip_version: `4`
    - protocol_text: `tcp`
    - source_address: `157.245.252.5`
    - destination_address: `125.229.96.130`
    - destination_port: `22`
    - tcp_flags: `S`
    - log_type: `filterlog`

## IP address lookup - The Geolocation
We can find more information from ip address. It's helpful for further aggregation and visualization.

1. Install extra fluentd `geoip` plugin.
    ```bash
    ### https://github.com/y-ken/fluent-plugin-geoip?tab=readme-ov-file#dependency
    sudo /opt/fluent/bin/fluent-gem install fluent-plugin-geoip
    ```
2. Download the geoip database.
    ```bash
    sudo curl -sSfL -o /opt/GeoLite2-City.mmdb https://github.com/P3TERX/GeoLite.mmdb/raw/download/GeoLite2-City.mmdb
    ```
3. Add geoip filter below to `~/fluentd/fluent.conf` before the `record_modifier` section.
    ```text
    <filter pf.filterlog.**>
      @type geoip
      geoip_lookup_keys source_address,destination_address
      geoip2_database /opt/GeoLite2-City.mmdb
      <record>
        source_country ${country.names.en["source_address"]}
        source_country_code ${country.iso_code["source_address"]}
        source_city ${city.names.en["source_address"]}
        destination_country ${country.names.en["destination_address"]}
        destination_country_code ${country.iso_code["destination_address"]}
        destination_city ${city.names.en["destination_address"]}
      </record>
    </filter>
    ```
4. Re-run fluentd you should see extra 6 fields that we've extracted through geoip filter:
    - source_country: `The Netherlands`
    - source_country_code: `NL`
    - source_city: `Amsterdam`
    - destination_country: `Taiwan`
    - destination_country_code: `TW`
    - destination_city: `Zhongli District`

## Writing logs to Loki
We've done a lot so far. Time to add an output and send logs to our lightweight log storage: ***Loki***.

1. Install extra fluentd `fluent-plugin-grafana-loki` plugin.
    ```bash
    ### https://grafana.com/docs/loki/latest/send-data/fluentd/
    sudo /opt/fluent/bin/fluent-gem install fluent-plugin-grafana-loki
    ```
2. Add output config below to the bottom of `~/fluentd/fluent.conf` and don't forget to update the address of the loki server. The whole fluentd config for this case can be downloaded [HERE](/files/fluent-pfsense-filterlog-loki.conf).
    ```text
    <match pf.**>
      @type loki
      url "http://loki:3100"
      include_thread_label false
      drop_single_key true
      <label>
        log_type $.log_type
        real_interface $.real_interface
        direction $.direction
        ip_version $.ip_version
        protocol_text $.protocol_text
        action $.action
        source_address $.source_address
        source_country $.source_country
        source_country_code $.source_country_code
        destination_address $.destination_address
        destination_country $.destination_country
        destination_country_code $.destination_country_code
        destination_port $.destination_port
      </label>
      <buffer>
        flush_interval 10s
        flush_at_shutdown true
      </buffer>
    </match>
    ```
## Visualization - Grafana
Once logs have been written into Loki with proper labels. We can easily create a dashboard in Grafana with Loki datasource. Here is an example dashboard of pfSense filterlog in this case. As usual, try [download](/files/pfsense-filterlog-20240425-2228.json) and import into your own Grafana server.
 
![The Grafana dashboard of pfSense filterlog.](/images/pfsense-filterlog-grafana.png)
