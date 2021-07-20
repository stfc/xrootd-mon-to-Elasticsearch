# xrootd-fstream-collector

This script collects UDP packets from XRootD servers containing transfer summaries and other information, and indexes them in Elasticsearch.

## Detailed description

It uses the [f-stream](https://xrootd.slac.stanford.edu/doc/dev51/xrd_monitoring.htm#_Toc49119286) monitoring stream from xrootd server, which is lighter weight than the t-stream monitoring we were previously using.

It currently parses the user connection messages, the f-stream open and f-stream close messages only. It only parses the Hdr, Xfr and Ops structures in the close message, not the SSQ (yet).

It will index all of the above events in elastic search, but will attempt to find previous messages for a transfer, first in a small memory cache, and then in elastic search. If messages are found they will be combined into the current message. e.g. An f-stream close event will have the f-stream open, and the user connection event fields added to it if they have been received by a collector.

If both the open and close messages are present when processing the close, the collector will attempt to derive some stats about the transfer, based on the time between the open and close events and the amount of data transferred. These are very 'hand wavy' for two reasons:

   1. We don't get individual timestamps for messages, just a beginning and end timestamp for all messages in the reporting interval (the 10s in the monitoring directive below). We derive a likely timestamp based on the two times and the position of the message in the bundle, but the accuracy depends heavily on the size of the reporting interval and the number of events that occurred in the interval.
   2. The client is not guaranteed to be transferring data just because the file is open. For some transfers (e.g. an xrdcp) this is mostly true and the derived read/write rates are reasonably accurate, but for clients opening the file, doing a readV, doing some processing, doing another readV etc. the derived rates can be very low.

## Dependencies

This script has been tested on python 3.6. The only dependency is the python elasticsearch6 client which can be installed via pip or yum (if on centos 7):

```
        yum install python36-elasticsearch6
```

## Configuration

The xrootd monitoring incantation needed to get the right messages for this collector is:

```
        xrootd.monitor all auth fstat 10s ops lfn xfr 5 ident 5m dest fstat info user redir IP:PORT
```

Although we are not parsing xfr messages, without specifying XFR there are no xfr or ops structures in the close message.

The collector is configured entirely by command line options:

```
        Usage: xrootd-fstream-collector [options]
        
        Options:
          -h, --help            show this help message and exit
          -p PORT, --port=PORT  UDP Port to listen on for messages.
          -b BIND, --bind=BIND  Listen for messages on a specific address; defaults to
                                0.0.0.0.
          -s ES_SERVERS, --servers=ES_SERVERS
                                ES servers to send events to. Can specify multiple
                                times, one working server will be used.
          -c CACHE_TIME, --cache-time=CACHE_TIME
                                how long to keep events in memory.
          -e DOC_TYPE, --evt-type=DOC_TYPE
                                ES type field for the events (not the '_type'field,
                                which should not be changed from 'doc')
          -t SLEEP_TIME, --time=SLEEP_TIME
                                Time to collect messages before sending (seconds)
          -i, --insecure        Disable SSL verification when connecting to ES
```

For example, we use these options at RAL:

```
        xrootd-fstream-collector \
            --port=9931 \
            --evt-type=echo_xrd_fstream \
            --time=10 \
            --cache-time=3600 \
            --insecure \
            -s https://SERVER1:9200 \
            -s https://SERVER2:9200 \
            -s https://SERVER3:9200
```

## Elastic Search event format

Since the user connection and fstream open events are also present in this event, I haven't documented those.

```
{
    # event metadata
    "@timestamp": "2021-07-13T17:29:22Z",
    "type": "echo_xrd_fstream",
    "xrd_msg_type": "fstream_close", # can be user_connection, fstream_open, fstream_close
    
    # the following indicate which previous events are included in this event
    # and can be used for filter out 'incomplete' messages if needed
    "xrd_hasOpenInfo": true, 
    "xrd_hasUserInfo": true,
    "xrd_hasDerivedStats": true,

    # if an assosciated file open was found, the fields are added
    "xrd_file_uniq_id": "130.246.176.147-1626181535.1379",
    "xrd_fileOpenSize": 1073741824,
    "xrd_openedRW": false,
    "xrd_hasLFN": true,
    "xrd_user_uniq_id": "130.246.176.147-1626181535.1378",
    "xrd_lfn": "dteam:test/tom_byrne/1g",
    "xrd_open_timestamp_req_count": 6,
    "xrd_open_timestamp_tBeg": 1626197346,
    "xrd_open_timestamp_tEnd": 1626197355,
    "xrd_open_req_pos": 3,
    "xrd_server": "ceph-gw2.gridpp.rl.ac.uk",
    "xrd_derived_open_time": 1626197349,

    # if an associated user connection was found, the fields are added
    "xrd_prot": "xroot",
    "xrd_user": "vwa13372",
    "xrd_pid": "160465",
    "xrd_sid": "148034106559908",
    "xrd_host": "lcgui05.gridpp.rl.ac.uk",
    "xrd_auth_auth_protocol": "gsi",
    "xrd_auth_dn": "dteamuser",
    "xrd_auth_client_hostname": "lcgui05.gridpp.rl.ac.uk",
    "xrd_auth_xrootd_ver": "v5.2.0",
    "xrd_auth_xeqname": "xrdcp",
    "xrd_auth_ip_ver": "6",
    "xrd_client": "lcgui05.gridpp.rl.ac.uk",
    "xrd_origin_site": "gridpp.rl.ac.uk",

    # finally, the fields from the close message
    "xrd_record_add_time": 1626197365.8836944,
    "xrd_forceClose": false,
    "xrd_close_timestamp_req_count": 6,
    "xrd_close_timestamp_tBeg": 1626197356,
    "xrd_close_timestamp_tEnd": 1626197365,
    "xrd_close_req_pos": 5,
    "xrd_derived_close_time": 1626197362,
    "xrd_XFR_read": 1073741824,
    "xrd_XFR_readv": 0,
    "xrd_XFR_write": 0,
    "xrd_OPS_read": 128,
    "xrd_OPS_readv": 0,
    "xrd_OPS_write": 0,
    "xrd_OPS_rsMin": 0,
    "xrd_OPS_rsMax": 0,
    "xrd_OPS_rsegs": 0,
    "xrd_OPS_rdMin": 8388608,
    "xrd_OPS_rdMax": 8388608,
    "xrd_OPS_rvMin": 0,
    "xrd_OPS_rvMax": 0,
    "xrd_OPS_wrMin": 0,
    "xrd_OPS_wrMax": 0,

    # and then any derived stats (calculated by collector)
    "xrd_derived_duration": 13,
    "xrd_derived_write_rate": 0,
    "xrd_derived_read_rate": 82595524.92307693,
    "xrd_derived_readv_rate": 0,
    "xrd_derived_read_total_bytes": 1073741824,
    "xrd_derived_read_total_rate": 82595524.92307693,
    "xrd_derived_total_bytes": 1073741824,
    "xrd_derived_data_transferred": true,
    "xrd_derived_is_write": false,
    "xrd_derived_is_read": true,
    "xrd_derived_is_readv": false
}
```
