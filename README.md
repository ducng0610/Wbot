# Chatbot Development Tutorial: How to build a fully functional weather bot on Facebook Messenger

## Overview

This is an example on how you can build a Weather Chatbot on Facebook platform, using Ruby on Rails and Wit.ai The technology stack we used is

<div class="page" title="Page 1">

<div class="layoutArea">

<div class="column">

*   Server backend: Ruby on Rails
*   Natural Language Processing platfrom: Wit.ai
*   Deployment on Facebook Messenger
*   The Singapore National Environment Agency provides a nice [API](https://www.nea.gov.sg/api/) (for free) that gives current weather as well as forecasts

Sample feature: (1) Able to return the “2 Hour Nowcast” when user asks for the current weather User: “What’s the weather in Singapore?” Bot: “The weather in Singapore is {weather}” User: “How’s the weather in Singapore?” Bot: “The weather in Singapore is {weather}” User: “Is it raining in Singapore” Bot: “The weather in Singapore is {weather}”</div>

</div>

</div>

You can try out the weather bot here: [WBot By Robusttechhouse](https://www.facebook.com/wbotbyrth) ![Screen Shot 2017-02-07 at 12.03.28 PM](https://robusttechhouse.com/wp-content/uploads/2017/02/Screen-Shot-2017-02-07-at-12.03.28-PM-1.png)

## Setting Up Wit.AI

Go to [https://wit.ai/home](https://wit.ai/home) and create a wit app for you. Read [https://wit.ai/docs/quickstart](https://wit.ai/docs/quickstart) and follow the steps there. Then, go to the settings in your wit app and get the token id. ![Screen Shot 2017-02-07 at 12.09.05 PM](https://robusttechhouse.com/wp-content/uploads/2017/02/Screen-Shot-2017-02-07-at-12.09.05-PM.png)

## Integrate Rails app with Wit

### Models

We’re gonna only have two, relatively simple models; Message and Conversation In _/app/models/message.rb_

<pre class="lang:java decode:true" title="message.rb"># frozen_string_literal: true
class Message
  include Mongoid::Document
  include Mongoid::Timestamps

  field :body, type: String
  field :conversation_id, type: Integer
  field :kind, type: String

  has_many :quick_replies
  belongs_to :conversation

  validates_inclusion_of :kind, in: %w(outgoing incoming), allow_nil: false
  validates :conversation, presence: true
end
</pre>

  In _/app/models/conversation.rb_

<pre class="lang:default decode:true" title="conversation.rb"># frozen_string_literal: true
class Conversation
  include Mongoid::Document
  include Mongoid::Timestamps

  field :uid, type: String
  field :context, type: Hash

  has_many :messages
end
</pre>

### WitExtension Singleton

'wit' is a very nice gem that support our rails app to integrate with wit.ai

<pre class="lang:default decode:true ">gem 'wit'</pre>

  Create a new wit_extension.rb file in _/extensions_. What we need to do now, is create a WitExtension Singleton class and in it’s initializer, we set up a Wit client, it’s **access_token** and **actions**. Thanks to the above code, we can call **WitExtension.instance.client** anywhere in our Rails application and it would return the same instance of **WitExtension** and hence, the same Wit client object. Note that you shouldn’t actually have your access_token or any other token like it just lying around in code waiting to be put in version control. You should use the secrets.yml for development and environment variables in production.

Our **actions** Hash get the conversation to update the context with the right keys at the right time.

The code in the **getForecast** action extracts the entities from Wit’s request parameter and updates **context**’s keys and values based on whatever it requires to operate.

It is from the returned context Hash that Wit decides what to do based on the presence and/or absence of any of the keys.

<pre class="lang:java decode:true" title="wit_extension.rb"># frozen_string_literal: true
require 'wit'
require 'singleton'

class WitExtension
  include Singleton

  def initialize
    access_token = ENV['server_access_token']
    actions = {
      send: lambda do |_request, response|
        puts("[debuz] got response... #{response['text']}")

        message = Message.create(body: response['text'], kind: 'outgoing', conversation: @conversation)
        message.digest
      end,

      getForecast:
        lambda do |request|
          context = request['context']
          entities = request['entities']

          location = first_entity_value(entities, 'location') || context['location']

          if location
            forecast = search_forecast(location)
            context['forecast'] = forecast
            new_context = {}
          else
            new_context = context
          end

          @conversation.update(context: new_context)
          return context
        end
    }

    @client = Wit.new(access_token: access_token, actions: actions)
  end

  attr_reader :client

  def set_conversation(conversation)
    @conversation = conversation
  end

  private

  def first_entity_value(entities, entity)
    return nil unless entities.key? entity
    val = entities[entity][0]['value']
    return nil if val.nil?
    val.is_a?(Hash) ? val['value'] : val
  end

  def search_forecast(location)
    puts "[debuz] Searching for weather in #{location} ..."
    WeatherExtension.search_forecast(location)
  end
end
</pre>

### Chat Extension

This class acts as a controller between messages from Facebook and our Bot's brain (Wit). It also has the responsibility to backup messages & conversations.

<pre class="lang:default decode:true" title="chat_extension.rb"># frozen_string_literal: true
class ChatExtension
  class << self
    def response(message, uid)
      puts "[debuz] asking WIT for... #{message}"
      find_or_initialize_conversation(uid)
      create_incoming_message(message)
      WitExtension.instance.client.run_actions(@conversation.uid, message, @conversation.context.to_h)
    end

    private

    def find_or_initialize_conversation(uid)
      @conversation = Conversation.find_or_create_by(uid: uid)
      WitExtension.instance.set_conversation(@conversation)
    end

    def create_incoming_message(message)
      create_message('incoming', message)
    end

    def create_message(kind, message)
      @message = @conversation.messages.create(
        body: message,
        kind: kind
      )
    end
  end
end
</pre>

## Integrate Rails app with Facebook Messenger

### Set up Facebook App

Head on over to the [developer console](https://developers.facebook.com/apps) and press “Add a New App”. After creating one, you can skip the quick start. You’ll end up here. ![1*0DZc9c_t2Sr2DuUpIYvO5Q](https://robusttechhouse.com/wp-content/uploads/2017/02/10DZc9c_t2Sr2DuUpIYvO5Q.png) From here, you’re going to want to press “+Add Product” and add Messenger. After we configure a webhook, Facebook wants us to validate the URL to our application.

### Set up Rails App

We’ll be using the [facebook-messenger](https://github.com/hyperoslo/facebook-messenger) gem. It’s arguably the best Ruby client for Facebook Messenger. Add initializer code

<pre class="lang:default decode:true " title="facebook_messenger.rb"># frozen_string_literal: true
# config/initializers/facebook_messenger.rb

unless Rails.env.production?
  bot_files = Dir[Rails.root.join('app', 'bot', '**', '*.rb')]
  bots_reloader = ActiveSupport::FileUpdateChecker.new(bot_files) do
    bot_files.each { |file| require_dependency file }
  end

  ActionDispatch::Callbacks.to_prepare do
    bots_reloader.execute_if_updated
  end

  bot_files.each { |file| require_dependency file }
end
</pre>

  Add initial code for our bot

<pre class="lang:default decode:true " title="listen.rb"># frozen_string_literal: true
# app/bot/listen.rb

require 'facebook/messenger'

include Facebook::Messenger

Facebook::Messenger::Subscriptions.subscribe(access_token: ENV['ACCESS_TOKEN'])

Bot.on :message do |message|
  # message.id          # => 'mid.1457764197618:41d102a3e1ae206a38'
  # message.sender      # => { 'id' => '1008372609250235' }
  # message.sent_at     # => 2016-04-22 21:30:36 +0200
  # message.text        # => 'Hello, bot!'

  begin
    if message.text.nil?
      message_text = KnownLocation.guess_known_location_by_coordinates(message.attachments.first['payload']['coordinates'].values)
    else
      message_text = message.text
    end

    puts "[debuz] got from Facebook... #{message.text}"
    ChatExtension.response(message_text, message.sender['id'])

  rescue => e
    puts '[debuz] got unhandlable message: ' + e.message + ' :@: ' + message.to_json
  end
end
</pre>

  Add to _config/application.rb_ so rails knows about our bot files

<pre class="lang:default decode:true "># Auto-load /bot and its subdirectories

config.paths.add File.join("app", "bot"), glob: File.join("**","*.rb")
config.autoload_paths += Dir[Rails.root.join("app", "bot", "*")]</pre>

  Update routes for _/bot_

<pre class="lang:default decode:true " title="route.rb"># config/routes.rb

Rails.application.routes.draw do
  mount Facebook::Messenger::Server, at: "bot"
end</pre>

  Set the _env_ variables for the following

<pre class="lang:default decode:true ">ACCESS_TOKEN=
VERIFY_TOKEN=
APP_SECRET=</pre>

## Wrap up

Now that we have a functional bot, we can play around with the UI elements that Facebook provides. You can check them out [here](https://developers.facebook.com/docs/messenger-platform/send-api-reference). With this post, I hope you can build a chat-bot for for your own. This is an interesting space as it’s fairly new, good luck! _You can find the example Rails app here: [https://github.com/duc4nh/Wbot](https://github.com/duc4nh/Wbot)_

## References

Special thanks to:

*   [How to Create a Facebook Messenger Bot with Ruby on Rails](https://chatbotslife.com/create-a-facebook-messenger-bot-with-ruby-on-rails-4ffd8b851135#.i8sm0hpsg)

*   [Wit.ai Explained — Part 2— Building a bot with Ruby on Rails](https://chunksofco.de/wit-ai-explained-part-2-building-a-bot-with-ruby-on-rails-360d469ea20a#.m05cubm3u)

*   [Wit-Facebook](https://github.com/hunkim/Wit-Facebook)

  Brought to you by [SingaporeChabots.sg](http://singaporechatbots.sg). Want to build a chatbot? Drop us a note !