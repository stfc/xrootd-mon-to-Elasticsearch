# xrootd-mon-to-Elasticsearch
Collect xrootd monitoring from UDP and send to Elasticsearch. Based on https://github.com/opensciencegrid/gratia-probe/blob/master/xrootd-transfer/gratia-xrootd-transfer

The line
```
        self.es = elasticsearch.Elasticsearch(['hostname1:9200', 'hostname2:9200'])
```
will need to be modified as appropriate, but the list of Elasticsearch hosts should really be a command line option.
