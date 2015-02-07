rails-grip
================

Author: Konstantin Bokarius <kon@fanout.io>

A Ruby on Rails GRIP library.

License
-------

rails-grip is offered under the MIT license. See the LICENSE file.

Installation
------------

```sh
gem install rails_grip
```

Add the following lines to your Rails application Gemfile:

```Ruby
gem 'gripcontrol'
gem 'rails_grip'
```

Usage
-----

Set grip_proxies in your application configuration:

```
# pushpin and/or fanout.io is used for sending realtime data to clients
config.grip_proxies = [
    # pushpin
    {
        'control_uri' => 'http://localhost:5561',
        'key' => 'changeme'
    }
    # fanout.io
    #{
    #    'control_uri' => 'https://api.fanout.io/realm/your-realm',
    #    'control_iss' => 'your-realm',
    #    'key' => Base64.decode64('your-realm-key')
    #}
]
```

You can also set any other EPCP servers that aren't necessarily proxies with publish_servers:

```
config.publish_servers = [
    {
        'uri' => 'http://example.com/base-uri',
        'iss' => 'your-iss', 
        'key' => 'your-key'
    }
]
```

Note that in Rails 4 the following should be set for API endpoints in the ApplicationController to avoid CSRF authenticity exceptions:

```Ruby
protect_from_forgery except: :<api_endpoint>
```

Example controller:

```Ruby
class GripController < ApplicationController
  def get
    # if the request didn't come through a GRIP proxy, throw 501
    if !RailsGrip.is_grip_proxied(request)
      render :text => "Not implemented\n", :status => 501
      return
    end

    # subscribe every incoming request to a channel in stream mode
    RailsGrip.set_hold_stream(request, '<channel>')
    render :text => '[stream open]'
  end

  def post
    # publish data to subscribers
    data = request.body.read
    RailsGrip.publish('<channel>', HttpStreamFormat.new(data + "\n"))
    render :text => "Ok\n"
  end
end
```

Stateless WebSocket echo service with broadcast endpoint:

```Ruby
class WebSocketOverHttpGripController < ApplicationController
  def echo
    render nothing: true

    # reject non-websocket requests
    RailsGrip.verify_is_websocket(request)

    # if this is a new connection, accept it and subscribe it to a channel
    if ws.is_opening
      ws.accept
      ws.subscribe('test_channel')
    end

    while ws.can_recv do
      message = ws.recv

      # if return value is nil, then the connection is closed
      if message.nil?
        ws.close
        break
      end

      # echo the message
      ws.send(message)
    end
  end

  def broadcast
    if request.method == 'POST'

      # publish data to all clients that are connected to the echo endpoint
      data = request.body.read
      RailsGrip.publish('<channel>', WebSocketMessageFormat.new(data))

      render :text => "Ok\n"
    else
      render :text => "Method not allowed\n", :status => 405
    end
  end
end
```
