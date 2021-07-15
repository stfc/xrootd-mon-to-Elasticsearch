# xrootd-fstream-collector

Collect xrootd monitoring from UDP and send to Elasticsearch.

It uses the [[https://xrootd.slac.stanford.edu/doc/dev51/xrd_monitoring.htm#_Toc49119286][f-stream]] monitoring stream from xrootd server, which is lighter weight than the t-stream monitoring we were previously using.

It currently parses the user connection messages, the f-stream open and f-stream close messages only. It only parses the Hdr, Xfr and Ops structures in the close message, not the SSQ (yet).

It will index all of the above events in elastic search, but will attempt to find previous messages for a transfer, first in a small memory cache, and then in elastic search. If messages are found they will be combined into the current message. e.g. An f-stream close event will have the f-stream open, and the user connection event fields added to it if they have been received by a collector.

If both the open and close messages are present when processing the close, the collector will attempt to derive some stats about the transfer, based on the time between the open and close events and the amount of data transferred. These are very 'hand wavy' for two reasons:

   1. We don't get individual timestamps for messages, just a beginning and end timestamp for all messages in the reporting interval (the 10s in the monitoring directive below). We derive a likely timestamp based on the two times and the position of the message in the bundle, but the accuracy depends heavily on the size of the reporting interval and the number of events that occurred in the interval.
   2. The client is not guaranteed to be transferring data just because the file is open. For some transfers (e.g. an xrdcp) this is mostly true and the derived read/write rates are reasonably accurate, but for clients opening the file, doing a readV, doing some processing, doing another readV etc. the derived rates can be very low.

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
