# Filter sensitive data

Note: this config option is also available as `define_cassette_placeholder`
  to reflect the fact that it is useful for more than just sensitive data.

  The `filter_sensitive_data` configuration option can be used to prevent
  sensitive data from being written to your cassette files.  This may be
  important if you commit your cassettes files to source control and do
  not want your sensitive data exposed.  Pass the following arguments to
  `filter_sensitive_data`:

    - A substitution string.  This is the string that will be written to
      the cassettte file as a placeholder.  It should be unique and you
      may want to wrap it in special characters like `{ }` or `< >`.
    - A symbol specifying a tag (optional).  If a tag is given, the
      filtering will only be applied to cassettes with the given tag.
    - A block.  The block should return the sensitive text that you want
      replaced with the substitution string.  If your block accepts an
      argument, the HTTP interaction will be yielded so that you can
      dynamically specify the sensitive text based on the interaction
      (see the last scenario for an example of this).

  When the interactions are replayed, the sensitive text will replace the
  substitution string so that the interaction will be identical to what was
  originally recorded.

  Warning, the data you're filtering may have been escaped or encoded.
  Thus you may need to escape the sensitive text. For example, if
  "some&password" is in your cassette within a context which is html
  escaped, you will need to filter "some&amp;password" or
  `CGI::escapeHTML('some&password')`. Similarly if it's escaped in a
  query string, then you will need to filter "some%26password"
  or `CGI::escape('some&password')`

  You can specify as many filterings as you want.

## Multiple filterings

_Given_ a file named "filtering.rb" with:

```ruby
if ARGV.include?('--with-server')
  $server = start_sinatra_app do
    get('/') { "Hello World" }
  end
end

require 'vcr'

VCR.configure do |c|
  c.hook_into :webmock
  c.cassette_library_dir = 'cassettes'
  c.filter_sensitive_data('<GREETING>') { 'Hello' }
  c.filter_sensitive_data('<LOCATION>') { 'World' }
end

VCR.use_cassette('filtering') do
  response = Net::HTTP.get_response('localhost', '/', $server ? $server.port : 0)
  puts "Response: #{response.body}"
end
```

_When_ I run `ruby filtering.rb --with-server`

_Then_ the output should contain "Response: Hello World"

_And_ the file "cassettes/filtering.yml" should contain "<GREETING> <LOCATION>"

_And_ the file "cassettes/filtering.yml" should not contain "Hello"

_And_ the file "cassettes/filtering.yml" should not contain "World"

_When_ I run `ruby filtering.rb`

_Then_ the output should contain "Hello World".

## Filter tagged cassettes

_Given_ a file named "tagged_filtering.rb" with:

```ruby
if ARGV.include?('--with-server')
  response_count = 0
  $server = start_sinatra_app do
    get('/') { "Hello World #{response_count += 1 }" }
  end
end

require 'vcr'

VCR.configure do |c|
  c.hook_into :webmock
  c.cassette_library_dir = 'cassettes'
  c.filter_sensitive_data('<LOCATION>', :my_tag) { 'World' }
end

VCR.use_cassette('tagged', :tag => :my_tag) do
  response = Net::HTTP.get_response('localhost', '/', $server ? $server.port : 0)
  puts "Tagged Response: #{response.body}"
end

VCR.use_cassette('untagged', :record => :new_episodes) do
  response = Net::HTTP.get_response('localhost', '/', $server ? $server.port : 0)
  puts "Untagged Response: #{response.body}"
end
```

_When_ I run `ruby tagged_filtering.rb --with-server`

_Then_ the output should contain each of the following:

| Tagged Response: Hello World 1   |
| Untagged Response: Hello World 2 |

_And_ the file "cassettes/tagged.yml" should contain "Hello <LOCATION> 1"

_And_ the file "cassettes/untagged.yml" should contain "Hello World 2"

_When_ I run `ruby tagged_filtering.rb`

_Then_ the output should contain each of the following:

| Tagged Response: Hello World 1   |
| Untagged Response: Hello World 2 |

## Filter dynamic data based on yielded HTTP interaction

_Given_ a file named "dynamic_filtering.rb" with:

```ruby
include_http_adapter_for('net/http')

if ARGV.include?('--with-server')
  $server = start_sinatra_app do
    helpers do
      def request_header_for(header_key_fragment)
        key = env.keys.find { |k| k =~ /#{header_key_fragment}/i }
        env[key]
      end
    end

    get('/') { "#{request_header_for('username')}/#{request_header_for('password')}" }
  end
end

require 'vcr'

USER_PASSWORDS = {
  'john.doe' => 'monkey',
  'jane.doe' => 'cheetah'
}

VCR.configure do |c|
  c.hook_into :webmock
  c.cassette_library_dir = 'cassettes'
  c.filter_sensitive_data('<PASSWORD>') do |interaction|
    USER_PASSWORDS[interaction.request.headers['X-Http-Username'].first]
  end
end

VCR.use_cassette('example', :match_requests_on => [:method, :uri, :headers]) do
  port = $server ? $server.port : 0
  puts "Response: " + response_body_for(
    :get, "http://localhost:#{port}/", nil,
    'X-Http-Username' => 'john.doe',
    'X-Http-Password' => USER_PASSWORDS['john.doe']
  )
end
```

_When_ I run `ruby dynamic_filtering.rb --with-server`

_Then_ the output should contain "john.doe/monkey"

_And_ the file "cassettes/example.yml" should contain "john.doe/<PASSWORD>"

_And_ the file "cassettes/example.yml" should contain a YAML fragment like:

```
X-Http-Password:
- <PASSWORD>
```

_When_ I run `ruby dynamic_filtering.rb`

_Then_ the output should contain "john.doe/monkey".
