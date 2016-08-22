__Installing ELK Stack__  
_Elasticsearch:_  
```brew install elasticsearch``` 

_Logstash:_  
```brew install logstash```

_Kibana_  
* Download the latest kibana from [KibanaURL](https://www.elastic.co/downloads/kibana)  
```wget https://download.elastic.co/kibana/kibana/kibana-3.1.3.tar.gz```  
* Unarchive kibana into your webspace:  
```tar zxvf kibana-3.1.3.tar.gz```  

__Configuring ELK Stack__
Get elasticsearch configuration file information from homebrew:  
```brew info elasticsearch```

__Important configuration files:__  
```vi /usr/local/etc/elasticsearch/elasticsearch.yml```   
```vi /usr/local/etc/elasticsearch/logging.yml```

Add the following lines to elasticsearch.yml:  
```http.cors.allow-origin: "/.*/"```  
```http.cors.enabled: true```  

Change the cluster name to just elasticsearch in elasticsearch.yml:  
```cluster.name: elasticsearch```  

Create a patterns directory:  
```mkdir /usr/local/logstash/patterns.d/```  

Create an apache-errors file in ```/Users/username/patterns.d/apache-error:```  
```APACHE_ERROR_LOG \[(?<timestamp>%{DAY:day} %{MONTH:month} %{MONTHDAY} %{TIME} %{YEAR})\] \[.*:%{LOGLEVEL:loglevel}\] \[pid %{NUMBER:pid}\] \[client %{IP:clientip}:.*\] %{GREEDYDATA:message}```

Create /Users/username/logstash/logstash.conf:
<pre>input {
  file {
    path => [ "/var/log/*.log", "/var/log/messages", "/var/log/syslog" ]
    type => "syslog"
  }
  file {
    path => "/var/apache/logs/custom_log"
    type => "apache_access_log"
  }
  file {
    path => "/var/apache/logs/error_log"
    type => "apache_error_log"
  }
}
 
filter {
  if [type] == "apache_access_log" {
    mutate { replace => { "type" => "apache-access" } }
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
    date {
      match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
    }
  }
  if [type] == "apache_error_log" {
    mutate { replace => { "type" => "apache-error" } }
    grok {
      patterns_dir => [ "/Users/username/logstash/patterns.d" ]
      match => [ "message", '%{APACHE_ERROR_LOG}' ]
      overwrite => [ "message" ]
    }
    if !("_grokparsefailure" in [tags]) {
      date {
        match => [ "timestamp", "EEE MMM dd HH:mm:ss.SSSSSS yyyy" ]
      }
    }
  }
  if [type] == "syslog" {
    grok {
      match => [ "message", "%{SYSLOGBASE2}" ]
    }
  }
}
 
output {
  elasticsearch { host => localhost }
  stdout { codec => rubydebug }
} </pre>
 
__Testing Install__

To verify that elasticsearch is running, visit port 9200 on whatever host you've set it up on:  
http://localhost:9200/  

To perform a search:  
http://localhost:9200/_search?pretty  

To view the mappings:  
http://localhost:9200/_mappings?pretty  

To view cluster health:  
http://localhost:9200/_cluster/health?pretty  

__Start up__
<code>elasticsearch</code>  
<code>logstash agent -f ~/logstash/logstash.conf</code>
