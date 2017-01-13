Sensu-client handling
==

Automatic registration / deregistration using a handler.

Registration
===

Sensu-client


client.json
```
{
  "client": {
    "name": "client.example.org",
    "subscriptions": [
      "default",
    ],
    "registration": {
      "handler": "addcmdb"
    }
  }
}
```

Sensu-server handlers.json

```
{
  "handlers": {
    "addcmdb": {
      "type": "pipe",
      "command": "addcmdb.rb"
    }
  } 
}
```

The file /etc/sensu/plugins/addcmdb.rb 


Deregistration
===

The moment sensu-client is stopped, it will be deleted.

client.json
```
{
  "client": {
    "name": "client.example.org",
    "subscriptions": [
      "default",
    ],
    "deregister": true,
    "deregistration": {
      "handler": "deregister"
    }
  }
}
```
Handlers on Sensu-Server
/etc/sensu/conf.d/handlers.json
```
{
  "handlers": {
    "deregister": {
      "type": "pipe",
      "command": "deregister.rb"
    }
  } 
}

```

/etc/sensu/plugins/deregister.rb

```
#!/usr/bin/env ruby

require 'rubygems'
require 'sensu-handler'

class Deregister < Sensu::Handler
  def handle
    delete_sensu_client!
  end

  def delete_sensu_client!
    response = api_request(:DELETE, '/clients/' + @event['client']['name']).code
    deletion_status(response)
  end

  def deletion_status(code)
    case code
    when '202'
      puts "202: Successfully deleted Sensu client: #{@event['client']['name']}"
    when '404'
      puts "404: Unable to delete #{@event['client']['name']}, doesn't exist!"
    when '500'
      puts "500: Miscellaneous error when deleting #{@event['client']['name']}"
    else
      puts "#{res}: Completely unsure of what happened!"
    end
  end

  def filter
    # override filter method to disable filtering of deregistration events
  end
end
```



sensu-server.log
```
{"timestamp":"2017-01-13T12:20:27.787597+0100","level":"info","message":"handler output","handler":{"type":"pipe","command":"deregister.rb","name":"deregister"},"output":["202: Successfully deleted Sensu client: client.example.orgl\n"]}
```

Please note that this will also trigger on sensu-client restart. But since its stateless, that should not harm, the client will be immediately re-added after restarting.

It will not trigger on when the server is unexpectedly shutdown or the process is killed. To act on that, keepalives can be expanded.


Keepalives
==

Another way to influence this is to use keepalives.

For Keepalives, the server will send 2 actions to the handler:
@event.action create and resolve.


```

{
  "client": {
    "name": "client.example.org",
    "subscriptions": [
      "default",
    ],
    "keepalive": {
      "thresholds": {
        "warning": 10,
        "critical": 300
      },
      "handler": "decommission"
    }
  }
}
```

Handlers on Sensu-Server

/etc/sensu/conf.d/handlers.json

```
{
  "handlers": {
    "decommission": {
      "type": "pipe",
      "command": "decommission.rb"
    }
  } 
}
```

Using the ruby script /etc/sensu/plugins/decommission.rb, one can do additional checks to see why the server has disappeared, before deleting it.

```
#!/usr/bin/env ruby

require 'rubygems' if RUBY_VERSION < '1.9.0'
require 'sensu-handler'

class Decomm < Sensu::Handler

  def handle
    if @event['action'].eql?('create')
      # Check the state here
    elsif @event['action'].eql?('resolve')
      # Resolved state
    end
  end

end
```
